# NCCL Collective 딥다이브 — 알고리즘 / 토폴로지 / env 매트릭스

> **목적**: PyTorchJob env에 박혀있는 `NCCL_IB_HCA/GID_INDEX/TC/SL` 같은 값을 **"왜 이 값이어야 하는가"**, Ring/Tree/NVLS가 **언제 바뀌는가**, `NCCL_DEBUG=INFO` 로그를 **어느 줄에서 무엇을 읽을지**까지 체계화.
> **선행**: [gpu-gpudirect-deep-dive.md](gpu-gpudirect-deep-dive.md), [../integration/dgx-ib-multinode-training-guide.md](../integration/dgx-ib-multinode-training-guide.md) §2
> **연결**: [../k8s/mlops-stack-deep-dive.md](../k8s/mlops-stack-deep-dive.md), [../k8s/nvidia-network-operator-deep-dive.md](../k8s/nvidia-network-operator-deep-dive.md)

---

## 0. 왜 NCCL을 "딥하게" 알아야 하나

면접관 시나리오:
- "PyTorchJob에 `NCCL_IB_TC=106`이 왜 들어있죠?" → 답변 못하면 그 아래 질문(IB QoS, SL-to-TC 매핑) 전부 막힘.
- "AllReduce 32GPU와 64GPU에서 알고리즘이 자동 바뀌는데 기준이 뭐죠?" → NCCL `tuner` 이해 필요.
- "IB 포트 한쪽이 죽으면 성능이 반토막인가 전체 멈추나?" → bootstrap vs transport, fallback.

답하려면 **알고리즘, 링크/토폴로지, 프로토콜, 튜닝 레버**를 계층으로 쌓아야 한다.

---

## 1. Collective 기초

### 1.1 연산 종류

| 연산 | 의미 | 사용 예 |
|------|------|---------|
| **AllReduce** | 모든 rank의 값을 합산/평균해서 모두에게 배포 | DDP gradient sync |
| **ReduceScatter** | 합산 결과를 각 rank가 1조각씩 받음 | ZeRO-2/3, FSDP |
| **AllGather** | 각 rank의 값을 모두에게 뿌림 | FSDP forward param unshard |
| **Broadcast** | 한 rank의 값을 모두에게 | 모델 초기 가중치 배포 |
| **Reduce** | 한 rank로만 합산 | Loss gather (드묾) |
| **AlltoAll** | 모든 rank 간 matrix transpose | MoE expert 라우팅 |

**면접 포인트**: DDP는 AllReduce, FSDP/ZeRO는 ReduceScatter + AllGather 쌍. AllReduce = ReduceScatter + AllGather (수학적으로 동치).

### 1.2 성능의 축

- **대역폭-bound**: 큰 메시지(> 수 MB) — 알고리즘의 step당 전송량이 중요.
- **지연-bound**: 작은 메시지(< 수십 KB) — 알고리즘의 hop 수가 중요.

NCCL은 이 두 레짐을 **자동 전환**(tuner).

---

## 2. AllReduce 알고리즘 3종

### 2.1 Ring

```
GPU0 ─▶ GPU1 ─▶ GPU2 ─▶ GPU3 ─▶ GPU0
```

- 각 rank가 **"chunk의 1/N"** 을 이웃에게 릴레이.
- Step 수: `2(N-1)` (ReduceScatter N-1 + AllGather N-1).
- **전송량 per GPU**: `2 × (N-1)/N × M` (M = 메시지 크기) → N이 커져도 수렴.
- **장점**: 대역폭 최대 활용. 단일 링크가 병목이면 그 링크만큼 제한.
- **단점**: hop 수 = N. 지연 누적 → 작은 메시지 불리.

### 2.2 Tree (Double Binary Tree)

```
        root
       /    \
     N1      N2
    / \     / \
   N3 N4   N5 N6
```

- NCCL은 **"두 개의 이진트리를 반대 방향으로 오버레이"** 해서 대역폭 절반 손실을 만회.
- Step 수: `2 × log2(N)`.
- **장점**: 지연 낮음, 작은 메시지에 유리.
- **단점**: 내부 노드의 대역폭 부담이 편향.

### 2.3 NVLS (NVLink SHARP)

- **SHARP** (Scalable Hierarchical Aggregation and Reduction Protocol) — IB 스위치 또는 NVSwitch가 **하드웨어에서 직접 reduce** 수행.
- NVLink SHARP = NVSwitch에서 reduce. **H100 + NVSwitch 3세대** 이상.
- Step 수: 사실상 1 (스위치가 집계).
- **조건**: `NCCL_NVLS_ENABLE=1`, H100 + 4세대 NVLink + 적절한 NVSwitch, 메시지가 특정 크기 이상.

### 2.4 크로스오버 (언제 어떤 알고리즘이 선택되나)

NCCL 2.18+ 의 tuner는 (GPU 수, 링크 대역폭, 메시지 크기)로 자동 결정. 대략:

| 메시지 크기 | 2-8 GPU (NVLink 1노드) | 16-64 GPU (IB 다노드) | 128+ GPU |
|-------------|------------------------|------------------------|----------|
| < 64KB | Tree | Tree | Tree + SHARP |
| 64KB ~ 1MB | Ring | Tree | NVLS/SHARP |
| > 1MB | Ring | Ring | NVLS/SHARP + Ring fallback |

**강제 레버**: `NCCL_ALGO=Ring|Tree|CollnetChain|CollnetDirect|NVLS`.

### 2.5 회사 설정 (32 GPU, 4노드 × 8H100)

- 단일 노드 내: NVSwitch 기반 **NVLink**, NVLS 가능.
- 노드 간: IB NDR 400Gbps × 8포트 per node.
- 대형 메시지(수백 MB, 수 GB) 위주 LLM 학습 → **Ring이 기본, NVLS로 가속**.

---

## 3. NCCL의 전송 계층

### 3.1 Bootstrap vs Transport

- **Bootstrap**: rank 간 **초기 discovery** (UNIX socket, TCP). `NCCL_SOCKET_IFNAME` 이 여기서 씀.
- **Transport**: 실제 데이터 전송.
  - `NET/IB` — InfiniBand verbs (ibverbs) + GPUDirect RDMA
  - `NET/Socket` — 일반 TCP (폴백)
  - `P2P` — 같은 노드 내 GPU-GPU (NVLink 또는 PCIe)
  - `SHM` — 같은 노드 CPU shared memory

### 3.2 로그 해석 (`NCCL_DEBUG=INFO`)

```
NCCL INFO Bootstrap : Using eth0:192.168.10.11<0>
  → Bootstrap NIC: eth0 (관리망). IB가 아님 — 정상.

NCCL INFO NET/IB : Using [0]mlx5_3:1/IB [1]mlx5_4:1/IB ...
  → Transport NIC: IB HCA 목록. NCCL_IB_HCA 필터 반영.

NCCL INFO NET/Plugin : No plugin found (libnccl-net.so)
  → 외부 플러그인 없음. 기본 동작.

NCCL INFO Channel 00 : 0[1b000] -> 1[1c000] via NET/IB/0/GDRDMA
  → Channel 00, GPU 0 → GPU 1, IB HCA index 0, GPUDirect RDMA 활성.
```

**체크포인트**:
- `NET/IB` 안 보이고 `NET/Socket`만 → IB 미활성. NCCL_IB_HCA, RDMA plugin 확인.
- `GDRDMA` 대신 `/CPU` → GPUDirect RDMA 불가. peermem 모듈 확인.
- `Channel` 수 적음 (예: 2) → 대역폭 저하. `NCCL_MIN_NCHANNELS` 조정.

### 3.3 Channel 개념

NCCL은 collective를 **여러 channel**로 분할해 병렬 전송. Channel 수 = GPU 수 × 링크 수 요인. 더 많을수록 대역폭↑ but SM 자원 차지↑.

---

## 4. env 매트릭스 (회사 PyTorchJob 값 해석)

### 4.1 회사 실제 값 (manifest 발췌)

```yaml
env:
- name: NCCL_IB_HCA
  value: mlx5_3,mlx5_4,mlx5_5,mlx5_6,mlx5_7,mlx5_8,mlx5_9,mlx5_10
- name: NCCL_IB_GID_INDEX
  value: "3"
- name: NCCL_IB_TC
  value: "106"
- name: NCCL_IB_SL
  value: "3"
- name: NCCL_IB_TIMEOUT
  value: "22"
- name: NCCL_DEBUG
  value: "INFO"
- name: NCCL_SOCKET_IFNAME
  value: "eth0"
```

### 4.2 각 값의 의미

| env | 값 | 의미 |
|-----|-----|------|
| `NCCL_IB_HCA` | `mlx5_3..10` | 사용할 IB HCA **화이트리스트**. 관리용 HCA(mlx5_0~2) 제외. |
| `NCCL_IB_GID_INDEX` | `3` | RoCEv2용 GID index. **IB에서도 명시하면 ambiguity 해소**. 0=Link local, 3=관례적 RoCEv2 |
| `NCCL_IB_TC` | `106` | Traffic Class (IP ToS/DSCP 역할). IB 스위치의 QoS priority group 매핑 |
| `NCCL_IB_SL` | `3` | Service Level. IB fabric의 VL(Virtual Lane)에 매핑 |
| `NCCL_IB_TIMEOUT` | `22` | 전송 타임아웃. `4.096μs × 2^value` = 약 **17초** (값 22일 때) |
| `NCCL_DEBUG` | `INFO` | 로그 verbosity |
| `NCCL_SOCKET_IFNAME` | `eth0` | **Bootstrap용** NIC. IB가 아님 — rank discovery는 관리망으로 |

### 4.3 TC=106 / SL=3 의 의미 (IB QoS)

IB fabric은 **16 SL × N VL** QoS 체계. opensm에서 설정:

1. **SL → VL 매핑 (SL2VL 테이블)**: opensm이 스위치에 배포.
2. **SL → TC 매핑**: host 측 (NCCL env로).
3. **VL 간 대역폭 비율 (VLArb)**: opensm.

**왜 SL=3, TC=106?** MDS가 IB fabric 설계할 때 학습용 traffic을 특정 SL로 격리 → 관리/스토리지 traffic과 경합 방지. 이 값은 **fabric 설정에 맞춰 벤더가 지정한 값을 사용**.

**면접 답변**: "이 값의 절대적 의미보다 **fabric QoS 정책**과 일치시키는 게 핵심. 우리 fabric은 학습 traffic을 SL 3 + TC 106 priority group으로 격리하도록 MDS가 설정했고, NCCL env는 그 규약을 따라감."

### 4.4 NCCL_IB_TIMEOUT 공식

`timeout = 4.096μs × 2^value`

| value | timeout |
|-------|---------|
| 14 | 67ms |
| 18 | 1초 |
| 22 | **17초** (회사 값) |
| 23 | 34초 |
| 31 | 146분 |

**왜 17초?** IB RC(Reliable Connection) 재전송 + 스위치 failover를 기다리되, **NCCL collective는 barrier 성격**이라 너무 길면 Job 전체가 hang. 17초면 순간적 fabric 혼잡은 용납, 링크 완전 다운은 빠르게 감지.

---

## 5. 주요 튜닝 레버

### 5.1 알고리즘/프로토콜

| env | 값 | 효과 |
|-----|-----|------|
| `NCCL_ALGO` | `Ring`/`Tree`/`CollnetChain`/`CollnetDirect`/`NVLS` | 알고리즘 강제 |
| `NCCL_PROTO` | `Simple`/`LL`/`LL128` | wire protocol. LL은 저지연 소형, LL128은 NVLink 최적화 |
| `NCCL_NVLS_ENABLE` | `1` | NVSwitch SHARP 활성 |
| `NCCL_COLLNET_ENABLE` | `1` | IB SHARP 활성 (IB 스위치 지원 시) |

### 5.2 채널/SM

| env | 효과 |
|-----|------|
| `NCCL_MIN_NCHANNELS` / `NCCL_MAX_NCHANNELS` | 최소/최대 채널 수. 대역폭 vs SM 트레이드오프 |
| `NCCL_NTHREADS` | 채널당 스레드 수 |

### 5.3 IB 세부

| env | 효과 |
|-----|------|
| `NCCL_IB_QPS_PER_CONNECTION` | QP 수. 많을수록 대역폭↑ (NDR에서 유의미) |
| `NCCL_IB_SPLIT_DATA_ON_QPS` | 여러 QP에 데이터 스트라이프 |
| `NCCL_IB_DISABLE_RELAXED_ORDERING` | PCIe relaxed ordering 끄기 (특정 플랫폼 호환성) |

### 5.4 GPUDirect / P2P

| env | 효과 |
|-----|------|
| `NCCL_P2P_DISABLE=1` | 노드 내 P2P 끄기 (디버깅용) |
| `NCCL_SHM_DISABLE=1` | shared memory 경로 끄기 |
| `NCCL_NET_GDR_LEVEL` | GPUDirect RDMA 사용 거리 (PIX/PXB/PHB/SYS) |

### 5.5 토폴로지 파일

```
NCCL_TOPO_FILE=/path/to/topo.xml
```

NVIDIA Network Operator가 배포하는 topo.xml(노드 PCI 트리 기반)을 NCCL이 읽어 **P2P/GDR 가능 여부 + 최적 경로** 결정. 사용자 개입 없으면 자동 감지되지만, **SR-IOV 컨테이너 환경**에선 틀릴 수 있어 명시 권장.

---

## 6. 토폴로지 인지 (왜 중요한가)

### 6.1 `nvidia-smi topo -m` 해석

```
        GPU0   GPU1   GPU2   GPU3   GPU4   GPU5   GPU6   GPU7   mlx5_0 ...
GPU0    X      NV18   NV18   NV18   NV18   NV18   NV18   NV18   PXB
GPU1    NV18   X      NV18   NV18   NV18   NV18   NV18   NV18   PXB
...
```

| 코드 | 의미 |
|------|------|
| `NV#` | NVLink (숫자는 링크 개수) |
| `PIX` | 같은 PCIe 스위치 |
| `PXB` | PCIe 스위치 hop (다른 브릿지) |
| `PHB` | PCIe host bridge (CPU socket 내) |
| `SYS` | QPI/UPI (socket 간, 최악) |

**H100 DGX**: GPU-GPU는 전부 NVLink(NV18 = NVLink 18 lanes), GPU-HCA는 `PIX` (같은 PCIe 스위치).

### 6.2 GPU:HCA 페어링

회사 DGX H100: **1:1 페어링**. GPU 0 ↔ mlx5_3, GPU 1 ↔ mlx5_4, ... . 한 GPU의 AllReduce 데이터가 **자기 HCA**로 나감 — PCIe switch hop 최소화.

### 6.3 `NCCL_NET_GDR_LEVEL` 미세 조정

기본값 `PIX` 이상만 GDR 사용. `PXB`까지 허용하려면 `NCCL_NET_GDR_LEVEL=PXB`. 그러나 **성능 하락** 가능.

---

## 7. SHARP (InfiniBand SHARP)

### 7.1 개념

IB **스위치 내부에서 reduce 연산 수행** → 호스트 왕복 제거.

- Aggregation Tree는 스위치 OS가 fabric 전체로 형성.
- 호스트는 "내가 이 트리의 leaf"임을 등록 → NCCL이 CollNet plugin으로 사용.
- 요구: Mellanox/NVIDIA IB 스위치 (SHARPv3) + SHARP daemon (`sharpd`) + Subnet Manager 설정.

### 7.2 회사 지원 여부

Quantum-2 스위치는 SHARPv3 지원. **실제 사용 여부는 MDS의 opensm/sharp 설정**에 달림.
- `NCCL_COLLNET_ENABLE=1` + `libnccl-net-sharp.so` 플러그인 로드 필요.
- 로그에서 `NET/CollNet` 또는 `CollNet: SHARP` 보이면 활성.

**면접 꼬리**: "NVLS와 SHARP 차이?" → NVLS는 노드 내 NVSwitch, SHARP는 노드 간 IB 스위치. 둘은 **계층적으로 조합** 가능.

---

## 8. 장애/디버깅 시나리오

### 8.1 "NCCL timeout"

로그 예:
```
NCCL WARN Call to ibv_create_qp failed
NCCL ERROR Bootstrap : no socket interface found
[RANK 5] Watchdog caught collective operation timeout
```

체크 순서:
1. `NCCL_DEBUG=INFO` — 어느 단계에서 멈추는지 (Bootstrap/Transport/Collective).
2. `ibstatus` 각 Pod에서 — IB link Active?
3. `ibv_devinfo -d mlx5_3` — GID, MTU 확인.
4. `NCCL_SOCKET_IFNAME` 이 IB 아닌 eth인지 (bootstrap용).
5. MASTER_ADDR DNS 해결.

### 8.2 "대역폭 40% 밖에 안 나온다"

- `nccl-tests/all_reduce_perf` 로 측정.
- 예상 대역폭: `(BW per link) × channel 수 × 효율(0.8)`.
- 체크:
  - `NCCL INFO` 의 Channel 수 — 4 이하면 `NCCL_MIN_NCHANNELS` ↑.
  - GDR 활성 — `GDRDMA` 문자열.
  - IB MTU — 스위치와 Host가 4096(4KB) 매치.
  - `NCCL_IB_QPS_PER_CONNECTION=4` 로 NDR 포화.

### 8.3 "특정 노드만 느리다"

- `ibstat` — Rate(NDR=400), LinkWidth(4X), PhysState(LinkUp).
- `mlxlink -d mlx5_3 -m` — BER (Bit Error Rate).
- `ibdiagnet` — fabric 전체 진단.

### 8.4 "알고리즘이 Tree로 고정되는데 Ring으로 강제하고 싶다"

```
NCCL_ALGO=Ring NCCL_PROTO=Simple ./my_job
```

- 성능 비교 후 tuner의 선택이 최적인지 검증.

---

## 9. 실습

### 9.1 nccl-tests

```bash
# Pod 안에서
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests && make MPI=0

# 32 GPU AllReduce (PyTorchJob 환경에서)
NCCL_DEBUG=INFO ./build/all_reduce_perf -b 8 -e 8G -f 2 -g 1
```

**봐야 할 값**:
- `busbw` (algorithm bandwidth) — 이론 대역폭의 80-90% 기대.
- out-of-place vs in-place 차이 미미.

### 9.2 로그 저장 패턴

```yaml
env:
- name: NCCL_DEBUG
  value: "INFO"
- name: NCCL_DEBUG_FILE
  value: "/tmp/nccl-%h-%p.log"
- name: NCCL_DEBUG_SUBSYS
  value: "INIT,NET,GRAPH,TUNING"
```

`%h` = hostname, `%p` = pid. Rank별 파일 분리.

### 9.3 GDR 확인

```
NCCL INFO Channel 00/0 : 0[1b:00.0] -> 1[1c:00.0] [send] via NET/IB/0/GDRDMA
```

`GDRDMA` 있으면 성공. 없고 `/read` 또는 `/CPU`면 CPU bounce 경로.

---

## 10. 면접 Q&A

### Q1 (기초). AllReduce의 대역폭 복잡도가 N에 비례하지 않는 이유?
A. Ring AllReduce는 각 GPU가 **2(N-1)/N × M** 만 보냄. N 커져도 수렴(≈ 2M). 그래서 수만 GPU 스케일에도 대역폭 부담 일정.

### Q2. NCCL_IB_TC=106, SL=3 둘 다 왜 있나? 중복 아닌가?
A. 다른 계층. **SL**은 IB fabric 내부 QoS (VL 매핑). **TC**는 host의 IP ToS/DSCP 필드 — IPoIB 또는 RoCEv2일 때 사용되며, IB-native에서도 일부 스위치는 TC를 SL2VL 힌트로 사용. 둘을 **동일 priority group**을 가리키도록 일치시키는 게 관례.

### Q3. Ring vs Tree 크로스오버 지점을 어떻게 판단?
A. 메시지 크기 × GPU 수의 곱. 대략 `M × N < threshold`면 Tree 유리(지연 중심), 아니면 Ring(대역폭 중심). NCCL tuner는 이 경계를 런타임에 계산. 강제 비교는 `NCCL_ALGO` + `nccl-tests`.

### Q4. NVLS와 SHARP 차이 및 조합?
A. NVLS = NVSwitch(노드 내), SHARP = IB 스위치(노드 간). 둘 다 있으면 NCCL이 **계층적으로 적용** — 노드 내 NVLS로 먼저 reduce → 노드 간 SHARP로 추가 reduce. 계산은 스위치가, 호스트 CPU는 0.

### Q5. GPUDirect RDMA가 로그에 안 뜬다. 원인 후보?
A. (1) `nvidia-peermem` 커널 모듈 미로드. (2) GPU-HCA가 다른 PCIe root complex (SYS 거리) → `NCCL_NET_GDR_LEVEL`로 허용 불가. (3) IOMMU 설정(ACS)이 P2P를 막음. (4) 컨테이너에 peermem 관련 커널 디바이스 접근 권한 없음.

### Q6. NCCL timeout 17초로 둔 판단 근거?
A. IB RC 재전송(수 ms) + 링크 복구(수백 ms) + subnet manager reroute(초 단위)를 흡수하되, collective는 barrier 성격이라 너무 길면 전체 Job이 hang. 17초는 **fabric 혼잡 흡수 + 영구 장애 조기 감지** 의 타협점.

### Q7. Bootstrap과 Transport는 왜 분리?
A. Bootstrap은 소규모 rank discovery — TCP/socket으로 충분, 관리망 사용해도 성능 영향 미미. Transport는 고대역폭 — IB verbs 전용. **분리해두면 관리망 장애와 데이터망 장애를 구분해 진단** 가능.

### Q8. 노드 수를 4 → 8로 늘리면 자동으로 알고리즘/채널 수가 바뀌나?
A. tuner가 재계산. Ring 길이 2배 → 지연 2배, 대역폭은 비슷. 채널 수는 HCA/NVLink 대역폭 대비 자동 조정. **수동 개입 필요한 것**: `NCCL_TOPO_FILE`을 클러스터 확장 후 갱신, `NCCL_MIN_NCHANNELS` 조정, opensm의 SL2VL가 새 스위치 반영됐는지.

### Q9. DDP gradient sync가 step마다 AllReduce인데, 체감 overhead 어떻게 줄이나?
A. (1) **gradient bucket** — 작은 grad 묶어 대형 AllReduce (PyTorch 기본 25MB). (2) **overlap** — backward와 AllReduce 동시 진행 (PyTorch `DistributedDataParallel` 기본). (3) FSDP로 전환 — ReduceScatter로 바뀌어 메모리↓, 통신량은 AllReduce와 동일.

### Q10. `nccl-tests` 결과가 공식 스펙의 60%다. 어디부터 보나?
A. 순서:
1. `ibstatus` Rate — NDR 400 아닌 HDR 200으로 협상된 케이스.
2. `ibv_devinfo` MTU — 2048로 떨어진 케이스.
3. Channel 수 — 적으면 `NCCL_MIN_NCHANNELS` 증가.
4. GDR 활성 — 로그 `GDRDMA`.
5. CPU affinity — `numactl` 잘못 묶였는지.
6. 스위치 단 SL2VL / VLArb — 다른 traffic과 경합.

---

## 11. 체크리스트

- [ ] AllReduce 의 Ring/Tree/NVLS 트레이드오프를 40초로 설명
- [ ] 회사 env 값 7개 의미 암기 (HCA, GID=3, TC=106, SL=3, TIMEOUT=22, DEBUG, SOCKET_IFNAME)
- [ ] NCCL_IB_TIMEOUT 공식 `4.096μs × 2^value`
- [ ] GPU:HCA 1:1 페어링, `nvidia-smi topo -m` 해석
- [ ] NVLS vs IB SHARP 계층 구분
- [ ] `NCCL_DEBUG=INFO` 로그에서 Bootstrap/Transport/Channel/GDRDMA 추출
- [ ] Timeout/대역폭/노드 단일 느림 시나리오 트리아지 순서

---

## 11.5 2025-2026 최신 키워드 (면접 심화)

- **NCCL 2.20+**: SHARPv3, NVLS+SHARP 계층적 조합, `NCCL_NCHANNELS_PER_NET_PEER` 세분화.
- **NVSHMEM 3.x**: NCCL collective 외부에 one-sided PGAS 모델. MoE AlltoAll에서 NCCL 대비 지연↓.
- **MSCCL / MSCCL++**: Microsoft 튜너. 커스텀 알고리즘 XML로 Ring/Tree 외 토폴로지 특화 경로 정의.
- **RoCEv2 전환 시 차이**: TC/SL 대신 DSCP/PFC(802.1Qbb) 설정. `NCCL_IB_GID_INDEX` 재계산 필수.

## 12. 연계 문서

- HW 기반: [./gpu-gpudirect-deep-dive.md](./gpu-gpudirect-deep-dive.md), [./cuda-stack-deep-dive.md](./cuda-stack-deep-dive.md)
- IB/CNI: [../kernel/rdma-ib-deep-dive.md](../kernel/rdma-ib-deep-dive.md), [../k8s/cni-multus-deep-dive.md](../k8s/cni-multus-deep-dive.md), [../k8s/nvidia-network-operator-deep-dive.md](../k8s/nvidia-network-operator-deep-dive.md)
- 학습 파이프라인: [../k8s/mlops-stack-deep-dive.md](../k8s/mlops-stack-deep-dive.md), [../integration/llm-training-workload-deep-dive.md](../integration/llm-training-workload-deep-dive.md)
- 관측/프로파일: [../k8s/observability-deep-dive.md](../k8s/observability-deep-dive.md)
- 허브: [../integration/dgx-ib-multinode-training-guide.md](../integration/dgx-ib-multinode-training-guide.md)
- 면접 서사: [../../interview/ml-platform-ownership-guide.md](../../interview/ml-platform-ownership-guide.md)

## 13. 참고

- NCCL docs: https://docs.nvidia.com/deeplearning/nccl/user-guide/
- `NCCL_IB_TIMEOUT` 공식 — IB 스펙 Section 9.7.6.1.1 (Timer interval).
- Confluence: 100663351 (Kubernetes with DDP — NCCL port, ephemeral port)
- 허브 문서: [../integration/dgx-ib-multinode-training-guide.md](../integration/dgx-ib-multinode-training-guide.md) §2.4, §2.5
- 관련: [../k8s/nvidia-network-operator-deep-dive.md](../k8s/nvidia-network-operator-deep-dive.md) (HCA 노출 경로)
