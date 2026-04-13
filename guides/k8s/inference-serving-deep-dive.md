# Inference Serving 딥다이브 — KServe / Triton / vLLM

> **목적**: 학습이 끝난 모델을 **K8s 위에서 서빙**할 때의 스택을 계층별로 이해. KServe(오케스트레이션) → Triton/vLLM(런타임) → GPU(하드웨어) + 오토스케일/라우팅/카나리까지.
> **선행**: [mlops-stack-deep-dive.md](mlops-stack-deep-dive.md), [../hw/gpu-gpudirect-deep-dive.md](../hw/gpu-gpudirect-deep-dive.md)
> **연결**: [multi-tenancy-scheduler-deep-dive.md](multi-tenancy-scheduler-deep-dive.md), [../hw/cuda-stack-deep-dive.md](../hw/cuda-stack-deep-dive.md)

---

## 0. 큰 그림

```
Client
  │  HTTP/gRPC
  ▼
Istio Gateway (OIDC 인증, 라우팅)
  │
  ▼
KServe InferenceService (CRD)
  ├─ Predictor   ← 모델 실행 (Triton or vLLM or 자체 구현)
  ├─ Transformer ← 전/후처리 (선택)
  └─ Explainer   ← 모델 해석 (선택)
  │
  ▼
Knative Service (자동 스케일, scale-to-zero)
  │
  ▼
Pod (GPU 요청)
```

**면접 프레임**: "서빙은 (1) **라우팅/인증**, (2) **오토스케일**, (3) **모델 런타임**, (4) **GPU 리소스 공유**, (5) **배포 전략** 5개 축을 어떻게 풀지 결정하는 일."

---

## 1. KServe

### 1.1 KServe가 해결하는 것

모델 서빙의 반복되는 **보일러플레이트를 CRD 1개로 캡슐화**:

- 모델 파일 URI (S3/GCS/PVC) → 자동 다운로드 sidecar
- 프레임워크별 기본 런타임 이미지
- Knative를 통한 scale-to-zero
- Istio 통한 라우팅 + 카나리
- 표준 V2 Inference Protocol (`/v2/models/.../infer`)

### 1.2 CRD 모양

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama3-8b
  namespace: serving
spec:
  predictor:
    minReplicas: 1
    maxReplicas: 4
    containerConcurrency: 1
    model:
      modelFormat:
        name: huggingface       # 또는 triton, onnx, pytorch, sklearn
      storageUri: "s3://models/llama3-8b"
      resources:
        limits:
          nvidia.com/gpu: 1
    nodeSelector:
      accelerator: h100
```

### 1.3 Deployment Mode — Serverless vs RawDeployment

| 모드 | 설명 | 언제 |
|------|------|------|
| **Serverless (default)** | Knative Service로 배포, scale-to-zero 가능, activator 경유 | 간헐적 트래픽, dev |
| **RawDeployment** | 일반 Deployment + HPA. Knative 없음 | 상시 트래픽, Knative 의존 피하기 |

`serving.kserve.io/deploymentMode: "RawDeployment"` 어노테이션으로 전환.

**회사에서 RawDeployment 선택 이유**:
- LLM은 로드 시간이 수십 초 ~ 분 → scale-to-zero가 실제론 첫 요청 지연만 악화.
- Knative activator가 단일 장애점이 됨.
- 고정 replicas + HPA(GPU util 기반)가 운영이 단순.

### 1.4 Predictor/Transformer/Explainer 파이프라인

```
request
  ├──▶ Transformer Pod (전처리 — 토큰화, 리사이즈)
  │       ▼
  │    Predictor Pod (모델)
  │       ▼
  │    Transformer Pod (후처리 — softmax, NMS)
  └──▶ Explainer Pod (선택 — SHAP/LIME)
```

각자 별도 Pod, 별도 스케일. **멀티 모델 앙상블**은 Predictor 내부에서 처리하거나 별도 InferenceGraph CRD 사용.

### 1.5 모델 스토리지 사이드카

`storageInitializer` initContainer가 Pod 시작 시 `storageUri`에서 `/mnt/models`로 복사. S3, GCS, Azure, PVC, OCI, URI(http) 지원.

**주의**: LLM 수십 GB 모델을 매 Pod 재시작마다 다운로드하면 비용/지연 크다. 대안:
- **PVC** 에 미리 캐시 + `ReadOnlyMany` 마운트.
- 노드 로컬 **hostPath**에 daemon이 미리 pull.
- Triton Model Repository를 **NFS/CephFS**로.

---

## 2. Triton Inference Server

### 2.1 무엇을 주나

- **다중 프레임워크 백엔드** — TensorRT, TensorFlow, PyTorch, ONNX, Python, OpenVINO, custom C++
- **동적 배치** (Dynamic Batching) — 대기열에서 request 모아 GPU 한 번에 실행
- **인스턴스 그룹** — 같은 모델 여러 copy를 한 GPU/여러 GPU에 배치
- **앙상블** — 모델 그래프를 config로 연결
- **HTTP + gRPC + C API** (in-process)

### 2.2 Model Repository 구조

```
models/
├── bert/
│   ├── config.pbtxt
│   └── 1/                    # version 1
│       └── model.onnx
├── bert/
│   └── 2/                    # version 2 (hot reload)
│       └── model.onnx
└── ensemble_pipeline/
    ├── config.pbtxt          # 그래프 정의
    └── 1/
```

`config.pbtxt` 핵심:
```
name: "bert"
backend: "onnxruntime"
max_batch_size: 64
input  [{ name: "input_ids",     data_type: TYPE_INT64, dims: [ -1 ] }]
output [{ name: "logits",        data_type: TYPE_FP32,  dims: [ -1, 2 ] }]
dynamic_batching {
  preferred_batch_size: [ 16, 32, 64 ]
  max_queue_delay_microseconds: 100
}
instance_group [{ count: 2, kind: KIND_GPU, gpus: [ 0 ] }]
```

### 2.3 Dynamic Batching 튜닝

- **preferred_batch_size**: 큰 배치 우선 (GPU utilization↑).
- **max_queue_delay_microseconds**: 지연 상한. **latency SLO 반영**.

트레이드오프: 대기 ↑ → throughput ↑, p99 latency ↑. 목표 SLO가 "p99 < 200ms"면 queue delay를 50ms 이하로.

### 2.4 Instance Group — 한 GPU에 모델 2개 복사?

`count: 2, kind: KIND_GPU` → 같은 GPU에 **모델 인스턴스 2개** 로드. 스트림 병렬로 요청 분산. GPU 메모리 여유 시 효과적 (warmup 후 동일 커널을 여러 스트림이 동시 실행).

---

## 3. vLLM — LLM 전용 런타임

### 3.1 왜 별도 런타임인가

LLM 특성:
- **자기회귀 디코딩** — token 1개씩 생성, 각 step이 이전 KV cache 의존.
- **가변 길이** — 프롬프트 길이가 요청마다 다름.
- **배치가 어렵다** — 길이가 달라서 padding 낭비.

Triton의 Dynamic Batching은 고정 shape 가정 → LLM에 부적합.

### 3.2 PagedAttention

**핵심 아이디어**: OS의 가상 메모리처럼 KV cache를 **고정 크기 페이지**로 관리.

- KV cache를 페이지(block) 단위로 할당. 페이지 크기 기본 16 토큰.
- 시퀀스마다 페이지 테이블.
- **연속적 메모리 불필요** → 단편화 거의 없음.
- 시퀀스 간 **페이지 공유** (prefix sharing) — 같은 system prompt는 페이지 재사용.

효과: GPU 메모리 사용률 **~3배** 향상, throughput **2-5배**.

### 3.3 Continuous Batching (= iteration-level scheduling)

전통 배치: 배치 내 모든 시퀀스가 **끝나야** 다음 배치 시작.
vLLM: **매 디코딩 스텝마다** 배치 재구성. 끝난 시퀀스 제거, 새 요청 추가. GPU 유휴 제거.

### 3.4 vLLM + KServe

```yaml
spec:
  predictor:
    model:
      modelFormat:
        name: huggingface
      args:
      - --model
      - meta-llama/Llama-3-8B
      - --tensor-parallel-size
      - "1"
      - --gpu-memory-utilization
      - "0.92"
```

또는 Triton 백엔드 중 `vllm_backend`로 벌크.

### 3.5 Tensor Parallel vs Pipeline Parallel

- **Tensor Parallel (TP)**: 레이어 내부 matmul을 여러 GPU에 쪼갬 — 통신 지연 크리티컬, NVLink 필수.
- **Pipeline Parallel (PP)**: 레이어를 그룹으로 나눠 GPU별로 — stage간 bubble, 배치 필요.

Llama-3-70B를 H100 8장에 올릴 때: TP=8 (NVLink + NVSwitch로 통신 지연 흡수).

---

## 4. 오토스케일

### 4.1 Knative 기반 (Serverless mode)

- **KPA (Knative Pod Autoscaler)** — 기본. `concurrency` 기반 (Pod당 동시 요청 수).
- `containerConcurrency: 1` → 한 Pod이 한 번에 한 요청.
- **Activator** — scale-to-zero 상태에서 요청 버퍼링 → Pod 뜨면 포워드.

**장점**: 유휴 비용 0.
**단점**: cold start — GPU 모델 로딩은 수십 초.

### 4.2 HPA 기반 (RawDeployment)

- CPU/Memory HPA는 GPU 워크로드에 부적절.
- **Custom metric** 사용: DCGM `DCGM_FI_DEV_GPU_UTIL`, 요청 queue 길이, 토큰 생성 속도.
- Prometheus Adapter로 metric API에 노출 → HPA가 참조.

예:
```yaml
metrics:
- type: External
  external:
    metric:
      name: dcgm_gpu_util_avg_5m
      selector:
        matchLabels:
          pod: llama3-8b
    target:
      type: AverageValue
      averageValue: "70"
```

### 4.3 회사에서 HPA 선택

LLM 로드 시간이 길어 KPA scale-to-zero는 비현실적. `minReplicas: 1` + custom metric HPA로 **70% util 유지**.

---

## 5. 라우팅 / 카나리 / A-B 테스트

### 5.1 KServe 내장 카나리

```yaml
spec:
  predictor:
    canaryTrafficPercent: 10
    model:
      ...                # 현재 모델 (90%)
```

새 spec을 apply하면 기본 10%만 신규로 라우팅. `canaryTrafficPercent: 100` 에서 promote.

### 5.2 Istio VirtualService

세밀한 라우팅:

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
spec:
  http:
  - match:
    - headers:
        x-model-version:
          exact: v2
    route:
    - destination:
        host: llama3-8b-predictor-v2.serving.svc.cluster.local
  - route:                  # 기본
    - destination:
        host: llama3-8b-predictor-v1.serving.svc.cluster.local
      weight: 90
    - destination:
        host: llama3-8b-predictor-v2.serving.svc.cluster.local
      weight: 10
```

---

## 6. GPU 리소스 공유 전략

### 6.1 노출 방식 3가지

| 방식 | 설명 | 장점 | 단점 |
|------|------|------|------|
| **독점** | Pod 1개당 GPU N개 (`nvidia.com/gpu: 1`) | 격리 완전 | 유휴 낭비 |
| **MIG** | GPU를 1/2, 1/3, 1/7 파티션 (H100은 7 instance max) | HW 격리 | 작은 모델만 |
| **MPS** | 소프트웨어 타임셰어링 | 유연 | 격리 약함 |
| **Time-slicing** | Kubernetes GPU operator의 소프트웨어 replicas | 저비용 | 격리 없음 |

### 6.2 회사 전략 (예상/설계 가이드)

- 학습 워크로드: **독점** (NCCL/IB 성능 보장)
- 서빙 LLM: **독점** 또는 **MIG 3g.40gb** (모델 크기별)
- 내부 실험 서빙: **time-slicing** (dev namespace)

### 6.3 MIG 프로파일 (H100)

- `1g.10gb` — 가장 작음 (7 per GPU)
- `2g.20gb`
- `3g.40gb` — 중간 LLM
- `4g.40gb`
- `7g.80gb` — 전체 차지 (= 독점과 동일)

---

## 7. Observability

### 7.1 KServe 표준 메트릭

- `kserve_inference_request_duration_seconds` (histogram)
- `kserve_inference_requests_total`
- 서빙 컨테이너가 Prometheus `/metrics` 노출

### 7.2 Triton 메트릭

- `nv_inference_queue_duration_us` — 배치 대기 시간
- `nv_inference_compute_infer_duration_us`
- `nv_inference_request_success`
- `nv_gpu_utilization`

### 7.3 vLLM 메트릭

- `vllm:num_requests_running`
- `vllm:num_requests_waiting`
- `vllm:time_to_first_token_seconds`
- `vllm:time_per_output_token_seconds`

**면접 Q**: "LLM 서빙 SLO를 뭘 걸겠나?" → **TTFT (Time To First Token)** + **ITL (Inter-Token Latency)** + throughput(tokens/s). 일반 웹서비스의 p99와 다름.

---

## 8. 장애/트러블 시나리오

### 8.1 "모델 로드 중 OOM"

- GPU 메모리 부족. `gpu-memory-utilization: 0.92` 같은 설정 확인.
- KV cache가 모델 weight 외 메모리 먹음 — 특히 LLM.
- 해결: 더 큰 GPU, TP 증가, quantization (int8/fp8).

### 8.2 "첫 요청 30초, 이후 정상"

- Cold start. 로드 + CUDA context 초기화.
- Warmup 요청을 readinessProbe로.
- KPA 쓰면 `minScale: 1` 또는 `initialScale` 조정.

### 8.3 "GPU util 30%인데 p99 latency 높다"

- Dynamic batching 미활성 또는 `max_queue_delay` 너무 큼.
- `instance_group count` 늘리면 병렬 스트림.

### 8.4 "스토리지 initializer가 timeout"

- 모델 수십 GB → 기본 timeout 부족.
- `activeDeadlineSeconds` 조정, 또는 PVC에 미리 캐시.

---

## 9. 회사 특화 포인트

- **Keycloak OIDC** — Istio RequestAuthentication + JWT 검증. [authn-authz-deep-dive.md](authn-authz-deep-dive.md).
- **모델 저장소** — **Ceph RGW SSD pool** (`s3://models/...`). MLflow와 같은 인프라.
- **Registry 연동** — MLflow Model Registry → stage:Production → KServe storageUri 자동 갱신.
- **모니터링** — Prometheus + Grafana. DCGM-exporter 메트릭과 KServe 메트릭 교차 대시보드.
- **Namespace 분리** — `serving-prod`, `serving-dev`. Kyverno로 prod namespace에는 `imagePullPolicy: IfNotPresent` 강제.

---

## 10. 실습

### 10.1 KServe 간단 예제

```bash
kubectl apply -f - <<EOF
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: sklearn-iris
  namespace: serving
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: gs://kfserving-examples/models/sklearn/1.0/model
EOF

# 호출
SERVICE_HOSTNAME=$(kubectl -n serving get inferenceservice sklearn-iris -o jsonpath='{.status.url}' | cut -d/ -f3)
curl -H "Host: ${SERVICE_HOSTNAME}" http://$INGRESS_IP/v1/models/sklearn-iris:predict -d '{"instances":[[6.8,2.8,4.8,1.4]]}'
```

### 10.2 vLLM 로컬 테스트

```bash
docker run --gpus all -p 8000:8000 \
  vllm/vllm-openai:latest \
  --model meta-llama/Llama-3-8B

curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"meta-llama/Llama-3-8B","prompt":"Hello","max_tokens":16}'
```

### 10.3 Triton perf_analyzer

```bash
perf_analyzer -m bert -b 1 --concurrency-range 1:8 --measurement-interval 10000
```

**봐야 할 값**: throughput(infer/s), p50/p95/p99 latency, GPU util.

---

## 11. 면접 Q&A

### Q1 (기초). KServe가 Knative를 쓰는 이유와 안 쓰면?
A. Knative는 **HTTP concurrency 기반 scale-to-zero**를 제공. 서빙 워크로드 특성(요청 단위 과금, 유휴 GPU 비용)에 적합. 단점은 activator 홉 + cold start. 피하려면 `deploymentMode: RawDeployment`로 일반 Deployment + HPA 사용.

### Q2. Triton Dynamic Batching vs vLLM Continuous Batching 차이?
A. Triton은 **요청 단위**로 모아 한 번 GPU 호출 — 시퀀스 완료까지 배치 고정. vLLM은 **매 토큰 생성 스텝마다** 배치 재구성 — 끝난 시퀀스 제거, 새 요청 합류. LLM은 토큰별로 길이가 달라 vLLM이 throughput 2-5배.

### Q3. LLM 서빙의 SLO를 뭘로?
A. (1) **TTFT** — 첫 토큰 지연. (2) **ITL / TPOT** — 토큰 간 간격. (3) **Throughput** — tokens/s. 일반 REST의 p99 latency로는 스트리밍 UX 평가 불가.

### Q4. MIG 3g.40gb와 MPS 차이?
A. MIG는 **하드웨어 파티션** — SM, L2 cache, 메모리 대역폭까지 물리 격리. MPS는 **소프트웨어 타임셰어 + 스트림 병합** — QoS 격리 없음. 보안이나 multi-tenant엔 MIG, 워크로드 섞을 땐 MPS.

### Q5. PagedAttention의 핵심 기여?
A. KV cache를 **고정 크기 페이지로 관리** → 단편화 제거, 메모리 이용률 ~3배. 추가로 **prefix sharing** — 같은 system prompt 여러 시퀀스가 공유. 결과: 같은 GPU에서 동시 접속자 2-5배.

### Q6. 카나리 배포 시 HTTP vs gRPC 차이 주의점?
A. Istio VirtualService의 traffic split은 HTTP 기반 헤더/path 매칭 쉬움. gRPC는 HTTP/2 multiplexing이라 **connection 단위로만 split** — request 단위 weight에 제약. gRPC 카나리는 클라이언트 측 service discovery에 가중치 주는 패턴이 더 안전.

### Q7. 모델 파일이 60GB. Pod 재시작마다 S3 다운로드하면 5분. 어떻게?
A. (1) `ReadOnlyMany` PVC에 미리 캐시 후 Pod들이 공유 마운트. (2) 노드별 DaemonSet이 `hostPath`에 pull. (3) 컨테이너 이미지에 모델 베이크(재현성 좋음, 이미지 비대). (4) Ceph RGW + `rclone cache` mount.

### Q8. Serverless 모드에서 GPU Pod scale-to-zero 효과 있나?
A. LLM은 로드 30-60초 → scale-to-zero는 첫 요청 UX 희생. **활용 가능한 곳**: 내부 dev 서빙, 주말 야간 batch. 상시 프로덕션은 `minReplicas: 1` + HPA가 현실적.

### Q9. DCGM util은 높은데 응답이 느리다. 원인?
A. (1) **모든 Pod이 한 GPU에 MIG 없이 time-share** → util은 100%로 보이지만 waiting. (2) PCIe 또는 NVLink 병목 — 다른 Pod이 대역폭 뺏음. (3) 큰 배치 때문에 queue delay 상승. (4) CPU-GPU host-to-device copy가 bottleneck (`nvidia-smi dmon` 에서 PCIe util 확인).

### Q10. MLflow Registry → KServe 자동 배포 플로우?
A. (1) 팀이 MLflow에서 stage를 "Production"으로 전환. (2) CI/CD (ArgoCD 또는 Airflow DAG)가 event 감지. (3) InferenceService YAML의 `storageUri`를 새 버전 경로로 업데이트. (4) GitOps apply → KServe reconcile → 카나리 배포.

---

## 12. 체크리스트

- [ ] KServe Serverless vs RawDeployment 선택 기준
- [ ] Predictor/Transformer/Explainer 역할
- [ ] Triton Dynamic Batching 파라미터 2개 (preferred_batch_size, max_queue_delay)
- [ ] vLLM PagedAttention + Continuous Batching 핵심
- [ ] TP vs PP 선택 기준 (NVLink 유무)
- [ ] LLM 서빙 SLO: TTFT/ITL/throughput
- [ ] MIG vs MPS vs time-slicing 격리 수준
- [ ] KServe 카나리 `canaryTrafficPercent`
- [ ] 큰 모델 로딩 전략 4가지 (PVC, hostPath, image bake, RGW cache)

---

## 13. 참고

- KServe docs: https://kserve.github.io/website/
- Triton: https://docs.nvidia.com/deeplearning/triton-inference-server/
- vLLM: https://docs.vllm.ai/
- PagedAttention paper: Kwon et al., SOSP 2023
- 관련: [mlops-stack-deep-dive.md](mlops-stack-deep-dive.md), [multi-tenancy-scheduler-deep-dive.md](multi-tenancy-scheduler-deep-dive.md)
