# LLM 학습 워크로드 딥다이브 — DDP / FSDP / ZeRO / Megatron / 체크포인트

> **목적**: 학습 프레임워크 레이어의 **parallelism 전략**과 이게 **NCCL collective, 메모리, 스토리지**에 어떻게 매핑되는지 이해. 플랫폼 엔지니어로서 **리소스 기획과 장애 해석**을 할 수 있어야 함.
> **선행**: [../hw/nccl-collective-deep-dive.md](../hw/nccl-collective-deep-dive.md), [../hw/cuda-stack-deep-dive.md](../hw/cuda-stack-deep-dive.md), [../hw/parallel-filesystem-deep-dive.md](../hw/parallel-filesystem-deep-dive.md)
> **연결**: [dgx-ib-multinode-training-guide.md](dgx-ib-multinode-training-guide.md)

---

## 0. 큰 그림

모델이 GPU 1장에 안 들어갈 때 쓰는 parallelism **5종**:

| 약어 | 뭘 쪼개나 | 통신 |
|------|----------|------|
| **DP** (Data Parallel) | batch | AllReduce grad |
| **DDP** | 동일 (분산 버전) | AllReduce grad |
| **FSDP / ZeRO-3** | parameters/grad/optim | ReduceScatter + AllGather |
| **TP** (Tensor Parallel) | 한 레이어의 matmul | AllReduce activations |
| **PP** (Pipeline Parallel) | 레이어 그룹 | P2P (stage 간) |
| **EP** (Expert Parallel) | MoE experts | AlltoAll |
| **SP** (Sequence Parallel) | 시퀀스 차원 | AllGather |

**면접 프레임**: "Llama-3-70B를 32 H100에 어떻게 올리나" → **3D parallelism (TP × PP × DP)** 로 답.

---

## 1. DDP (Data Parallel)

### 1.1 동작

```
각 GPU: 전체 모델 copy
           │
         batch split (N-way)
           │
         forward  → loss
         backward → grad
           │
     AllReduce(grad)  ← NCCL
           │
         optimizer step
```

- 모델 파라미터: **전체 복제** (N GPU, N copy).
- 메모리 = model + grad + optim + activations.

### 1.2 메모리 수식 (Adam 기준)

1 모델 파라미터당:
- Parameter: 2 byte (bf16)
- Gradient: 2 byte (bf16)
- Optim state: 8 byte (fp32 momentum + variance) = 2x fp32
- Master weight: 4 byte (fp32)

**총 16 byte/param**. 70B 파라미터 → 1.12TB. H100 80GB 한 장 **못 담음**.

→ DDP는 "모델이 한 장에 들어갈 때" 에만 가능. LLM엔 부적합.

### 1.3 Gradient Bucketing

PyTorch DDP는 grad를 **bucket (기본 25MB)** 단위로 모아 AllReduce. Backward 완료된 레이어부터 **compute과 comm overlap**.

---

## 2. FSDP / ZeRO

### 2.1 핵심 아이디어

**파라미터/grad/optim state를 모든 GPU에 샤딩**. 필요할 때만 AllGather로 모아 forward/backward.

### 2.2 ZeRO 스테이지 (DeepSpeed)

| Stage | 샤딩 대상 | 메모리 절감 | 통신 비용 |
|-------|-----------|-------------|-----------|
| 0 | 없음 (= DDP) | 1x | AllReduce |
| 1 | optim state | ~4x | AllReduce |
| 2 | optim + grad | ~8x | ReduceScatter + AllReduce |
| 3 | optim + grad + params | N x | ReduceScatter + AllGather × 2 |

### 2.3 FSDP (PyTorch native)

ZeRO-3 와 거의 동일 아이디어. 차이:
- FSDP는 **wrap policy**로 모듈 단위 샤딩 제어.
- `FullyShardedDataParallel(module, sharding_strategy=...)`.

```python
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP
from torch.distributed.fsdp.wrap import transformer_auto_wrap_policy

model = FSDP(
    model,
    sharding_strategy=ShardingStrategy.FULL_SHARD,      # ZeRO-3
    auto_wrap_policy=transformer_auto_wrap_policy,
    mixed_precision=MixedPrecision(param_dtype=torch.bfloat16),
)
```

### 2.4 ShardingStrategy

| 값 | 의미 | 용도 |
|----|------|------|
| `FULL_SHARD` | ZeRO-3 | 메모리 최우선 |
| `SHARD_GRAD_OP` | ZeRO-2 | 중간 |
| `NO_SHARD` | DDP | 모델이 작을 때 |
| `HYBRID_SHARD` | 노드 내 FULL, 노드 간 DDP | **통신 최적** |

### 2.5 HYBRID_SHARD 의미

- 노드 내 (NVLink): FSDP full shard → AllGather 자주 해도 NVLink 빠름.
- 노드 간 (IB): DDP → AllReduce 1회 per step만.
- IB 대역폭 부담 감소. 대규모 클러스터 권장.

---

## 3. Tensor Parallel (Megatron)

### 3.1 개념

한 Linear 레이어의 weight matrix를 N 조각으로 쪼개 N GPU에 배치. Forward 시 input을 브로드캐스트 → 각 GPU가 자기 shard로 연산 → AllReduce로 결과 합침.

### 3.2 Column vs Row Parallel

- **Column parallel**: weight를 출력 차원으로 split. 입력 전체를 받음.
- **Row parallel**: weight를 입력 차원으로 split. 출력 시 AllReduce.

Megatron은 **두 개를 번갈아** 써서 forward pass 중 AllReduce 한 번만 되도록.

### 3.3 통신량

한 레이어당 AllReduce(activation). Activation은 `batch × seq × hidden`. **통신 양이 크고 지연 민감** → NVLink 필수, IB는 TP에 부적합.

### 3.4 적용 범위

- Llama 70B: TP=8 (한 노드 내 NVSwitch).
- GPT-3 175B 급: TP=8 + PP + DP 조합.

---

## 4. Pipeline Parallel

### 4.1 개념

```
Layer 1~20   → Stage 0 (GPU 0~7)
Layer 21~40  → Stage 1 (GPU 8~15)
Layer 41~60  → Stage 2 (GPU 16~23)
Layer 61~80  → Stage 3 (GPU 24~31)
```

forward는 stage 0 → 1 → 2 → 3, backward는 역순. Stage 간 **activation P2P 전송**.

### 4.2 Bubble

초반 stage 3는 놀고, backward 후반 stage 0는 놀음 → **GPU idle**.

Micro-batch로 쪼개 pipeline 채움:
```
time ─►
Stage 0: [m1][m2][m3][m4]     [bw4][bw3][bw2][bw1]
Stage 1:    [m1][m2][m3][m4]  [bw4][bw3][bw2][bw1]
Stage 2:       [m1][m2][m3][m4]  [bw4]...
```

Bubble 비율 ≈ `(P-1)/(P + M - 1)`. Micro-batch 수 M 커질수록 줄어듦.

### 4.3 1F1B vs Interleaved

- **1F1B**: 각 stage가 forward 1 → backward 1 교대. 메모리 적음.
- **Interleaved 1F1B (Megatron-LM)**: 레이어를 여러 chunk로 쪼개 stage 여러 pass. Bubble 줄지만 통신↑.

---

## 5. 3D Parallelism 조합

### 5.1 전형적 배치 (Llama-3-70B @ 32 H100)

```
TP = 8   (노드 내 NVSwitch)
PP = 2   (노드 간 pipeline)
DP = 2   (replica 수)
Total = 8 × 2 × 2 = 32
```

- TP 그룹: 한 노드 내 8 GPU.
- PP 그룹: 두 노드에 걸친 stage 0 ↔ stage 1.
- DP 그룹: 두 replica (노드 2대씩).

### 5.2 Communication 분포

- TP: 노드 내 NVLink (400GB/s).
- PP: 노드 간 IB (HCA별 50GB/s bi-dir).
- DP: 노드 간 AllReduce (grad sync).

**설계 원칙**: **통신 빈도 높은 것은 빠른 링크로**. TP가 NVLink에 붙어야 하는 이유.

---

## 6. Mixed Precision

### 6.1 BF16 / FP16 / FP8

| dtype | 범위 | 정밀도 | 하드웨어 |
|-------|------|--------|---------|
| FP32 | 광범위 | 7 digits | 모든 GPU |
| FP16 | 제한 (underflow) | 3-4 digits | 모든 현대 GPU |
| BF16 | 광범위 | 2-3 digits | A100+, TPU |
| FP8 | 매우 좁음 | 1-2 digits | H100 (TransformerEngine) |

### 6.2 Loss Scaling

FP16은 작은 grad가 underflow → `loss × 2^k` 스케일 후 backward → grad ÷ 2^k. Dynamic scaling(PyTorch AMP) 자동 조정.

BF16은 범위 넓어 loss scaling 불필요 → **LLM 기본 BF16**.

### 6.3 FP8 (H100 TransformerEngine)

- Linear 레이어의 matmul만 FP8, 나머지는 BF16.
- E4M3 (forward), E5M2 (backward) — scale 자동 관리.
- **처리량 2x 이상**, 정확도 영향 미미.

---

## 7. Activation Checkpointing (Recompute)

### 7.1 개념

Forward의 activation을 저장 안 하고 **backward 시 재계산**. 메모리 N배 절감, 연산 ~30% 증가.

```python
from torch.utils.checkpoint import checkpoint
y = checkpoint(transformer_block, x)
```

Megatron: `--recompute-granularity full`.

### 7.2 Selective

Megatron-LM의 **selective checkpointing**: dropout/attention 같이 메모리 많이 먹으면서 연산 적은 부분만 recompute. 30% 연산 오버헤드 → 10%로.

---

## 8. Checkpoint

### 8.1 전략

| 방식 | 설명 | 크기 (70B 모델) |
|------|------|-----------------|
| 전체 상태 | 모델 + optim + rng + step | ~1TB |
| 모델만 | 파라미터만 | ~140GB (bf16) |

### 8.2 Sharded Checkpoint

`torch.distributed.checkpoint` (DCP):
- 각 rank가 자기 shard만 저장 → 병렬 IO.
- 로드 시 world size 달라져도 재구성 가능.

### 8.3 Async Checkpoint

compute이 다음 step 돌리는 동안 **GPU→CPU copy → 디스크 write**. 학습 지연 거의 없음.

### 8.4 Frequency

- LLM pretrain: 수시간 단위 (30분~3시간).
- Fine-tune: epoch 기준.
- **너무 자주** → IO 부담. **너무 드물게** → 장애 복구 비용.

### 8.5 실운영 이슈

- 체크포인트 write 중 OOM (FP32 master weight copy).
- 병렬 파일시스템 bandwidth 병목 (CephFS HDD 부적합, Weka/Lustre 권장).
- PVC 부족으로 write 실패 → 학습 중단.

---

## 9. Elastic Training

### 9.1 PyTorch Elastic (torchrun)

- rendezvous 서버 (c10d, etcd) 기반.
- 노드 추가/제거 시 **in-place 재구성**.
- fault tolerance: 한 노드 죽으면 나머지로 재시작.

### 9.2 PyTorchJob + ElasticPolicy

```yaml
spec:
  elasticPolicy:
    rdzvBackend: c10d
    rdzvHost: pytorch-master.slm.svc
    minReplicas: 2
    maxReplicas: 4
```

**조건**: 학습 코드가 elastic 호환 (checkpoint 로드, world size 가변 처리).

---

## 10. 프로파일링

### 10.1 PyTorch Profiler

```python
with torch.profiler.profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    record_shapes=True,
    with_stack=True,
) as prof:
    ...
```

TensorBoard로 시각화 — GPU 커널, NCCL 호출, CPU/GPU idle.

### 10.2 Nsight Systems

`nsys profile python train.py` — CUDA kernel level. NCCL은 nvtx range로 표시.

### 10.3 DCGM 프로파일 메트릭 (교차)

`PIPE_TENSOR_ACTIVE` 낮으면 TensorCore 미활용 → AMP/FP8 점검.

---

## 11. 장애/운영 시나리오

### 11.1 "Step 1000에서 NCCL timeout"

- 노드 하나가 느려져 barrier 못 맞춤.
- 원인 후보: GPU thermal throttle, IB 링크 flaky, CPU noisy neighbor.
- 체크: DCGM `GPU_TEMP`, `ibstatus`, `htop`.

### 11.2 "OOM at step 0"

- 모델 + optim + activations 계산 불일치.
- `nvidia-smi` 메모리 꽉 참 확인.
- 대책: TP/PP 늘리기, ZeRO stage 올리기, activation checkpoint.

### 11.3 "학습 중 급격한 loss spike"

- FP16 underflow (loss scaling 부족).
- 데이터 shuffle 문제.
- 이상 노드의 grad 손상 가능성 (TensorCore 에러).

### 11.4 "체크포인트 로드 실패"

- world size 달라짐 → DCP sharding 호환 안 됨.
- dtype 불일치 (FP16 저장 → BF16 로드).
- 대책: DCP 표준 사용, dtype 명시.

---

## 12. 회사 관점 (4 DGX = 32 H100)

- **LLM 7~13B** 규모: DDP 또는 FSDP full shard로 충분.
- **Llama-3 70B** 규모: **TP=8 + PP=2 + DP=2**. 32 GPU 사용.
- **Llama-3 405B** 이상: 자원 부족 → distilled 또는 fine-tune만.

본인(cwkim) 관점 답변:
> "현재 4 DGX 환경에선 70B까지 TP+PP+DP 3D로 학습 가능합니다. FSDP HYBRID_SHARD를 쓰면 IB 대역폭 부담을 줄일 수 있고, FP8 TransformerEngine으로 추가 가속도 가능합니다. 현대차 규모로 확장되면 Megatron-LM 기반 interleaved 1F1B + Weka 스토리지 조합이 표준이 될 것입니다."

---

## 13. 면접 Q&A

### Q1 (기초). DDP와 FSDP 차이 한 줄?
A. DDP는 모델 전체 복제 후 grad만 AllReduce. FSDP는 params/grad/optim을 **샤딩**해 메모리 절약, 대신 ReduceScatter + AllGather 쌍.

### Q2. FSDP HYBRID_SHARD 장점?
A. 노드 내는 FULL_SHARD(NVLink로 AllGather 자주 OK), 노드 간은 NO_SHARD(DDP, AllReduce 1회만). **IB 대역폭 부담 감소** + 메모리도 노드 내 샤딩으로 절약. 클러스터 topology에 최적.

### Q3. TP를 노드 간에 안 쓰는 이유?
A. TP는 매 레이어 forward/backward마다 activation AllReduce. 지연 민감. IB 지연(~1μs) + 큰 activation으로 학습 속도 급감. NVLink(~0.2μs, 400GB/s)에 국한.

### Q4. PP에서 Bubble이 뭐고 줄이는 법?
A. Pipeline 초반/후반 GPU 유휴 시간. Micro-batch 늘리면 `(P-1)/(P+M-1)` 로 감소. Interleaved 1F1B는 추가 개선이지만 통신 증가.

### Q5. ZeRO-3와 FSDP FULL_SHARD는 같은가?
A. 개념적으로 동일 (params/grad/optim 전부 샤딩). 구현 차이: FSDP는 PyTorch native, wrap policy 기반. DeepSpeed ZeRO는 runtime 후킹 + 다양한 튜닝 knob(offload, CPU adam 등).

### Q6. BF16 대신 FP8을 쓸 때 주의?
A. H100 TransformerEngine에선 per-tensor scale 관리 자동. 그러나 (1) attention softmax는 여전히 BF16/FP32. (2) optimizer state는 FP32 master weight. (3) FP8 scale 불안정 모델은 발산 → warmup 시 BF16으로.

### Q7. Checkpoint를 2시간 간격으로 저장. 학습 속도 영향?
A. async checkpoint 쓰면 거의 없음. sync면 save 5~30초 × IO 지연. 70B + 1TB checkpoint를 CephFS HDD에 쓰면 수 분 block. **Weka/Lustre 필요** 근거.

### Q8. Elastic training을 production에서 켜면?
A. 장점: 노드 장애 시 자동 재시작. 단점: rendezvous 지연, world size 변경 대응 코드 필요, checkpoint 빈도↑. **pretrain에선 유용, short fine-tune은 오히려 오버헤드**.

### Q9. AllReduce → ReduceScatter + AllGather는 수학적으로 동치인데 왜 FSDP가 절반을 AllGather 씀?
A. FSDP는 forward에서 **AllGather로 param 모으고**, backward에서 **ReduceScatter로 grad 흩음**. **메모리를 덜 쓰기 위해** 각 단계를 분리. DDP는 메모리 여유 있어 AllReduce 한 방에.

### Q10. 70B 학습이 step 100에서 NaN. 어디부터?
A. (1) loss scale 확인 (FP16이면). (2) grad norm 로그 — 폭주 지점. (3) optimizer state — inf/nan 값. (4) 노이즈 데이터 배제. (5) 특정 rank에만 발생 → 해당 노드 GPU ECC 체크. DCGM `XID`.

---

## 14. 체크리스트

- [ ] DDP / FSDP / TP / PP / EP 각 통신 패턴
- [ ] 70B 메모리 수식 (16 byte/param)
- [ ] FSDP ShardingStrategy 4종
- [ ] HYBRID_SHARD 동작 이유
- [ ] Megatron column/row parallel 교대
- [ ] PP bubble 공식
- [ ] FP8 TransformerEngine 적용 범위
- [ ] Activation checkpoint selective
- [ ] Async + sharded checkpoint
- [ ] Elastic training 트레이드오프

---

## 15. 참고

- FSDP: https://pytorch.org/docs/stable/fsdp.html
- DeepSpeed ZeRO: https://www.deepspeed.ai/tutorials/zero/
- Megatron-LM: https://github.com/NVIDIA/Megatron-LM
- TransformerEngine: https://docs.nvidia.com/deeplearning/transformer-engine/
- 관련: [dgx-ib-multinode-training-guide.md](dgx-ib-multinode-training-guide.md), [../hw/nccl-collective-deep-dive.md](../hw/nccl-collective-deep-dive.md), [../hw/parallel-filesystem-deep-dive.md](../hw/parallel-filesystem-deep-dive.md)
