# GPU & GPUDirect 완전 정복 - DGX의 본질

> **목적**: NVIDIA GPU 스택, NVLink/NVSwitch, GPUDirect RDMA/Storage, MIG, K8s GPU Operator를 처음부터 끝까지. 왜 DGX가 비싸고 왜 NCCL이 빠른지 진짜 이유를 안다.

---

## 0. 큰 그림

```
[App: PyTorch]
   ↓
[CUDA Runtime / cuDNN / NCCL]   ← 유저 라이브러리
   ↓
[CUDA Driver]                   ← 커널 모드 드라이버
   ↓
[NVIDIA GPU]                    ← 하드웨어
   ↓
NVLink ── NVSwitch ── 다른 GPU
PCIe   ── CPU
PCIe   ── IB NIC (GPUDirect RDMA)
PCIe   ── NVMe (GPUDirect Storage)
```

---

## 1. NVIDIA 드라이버 스택

### 1.1 구성

```
유저 공간:
  - libcudart.so       (CUDA Runtime)
  - libcudnn.so        (Deep learning primitives)
  - libnccl.so         (Collective communications)
  - libcuda.so         (CUDA Driver API)

커널 공간:
  - nvidia.ko          (메인 드라이버)
  - nvidia-uvm.ko      (Unified Memory)
  - nvidia-modeset.ko  (디스플레이)
  - nvidia-peermem.ko  (GPUDirect RDMA용)
```

### 1.2 디바이스 파일

```bash
ls /dev/nvidia*
# /dev/nvidia0 ~ /dev/nvidia7   ← GPU 디바이스
# /dev/nvidiactl                 ← 컨트롤
# /dev/nvidia-uvm                ← Unified Memory
# /dev/nvidia-uvm-tools

ls /dev/dri/      # 그래픽 인터페이스 (서버에선 보통 안 씀)
```

컨테이너에서 GPU를 쓰려면 이 디바이스 파일들이 마운트되어야 합니다 → **NVIDIA Device Plugin**의 일.

### 1.3 드라이버 설치 방법

```bash
# 일반: NVIDIA 공식 .run 파일
sudo ./NVIDIA-Linux-x86_64-535.xx.xx.run

# 또는 패키지 매니저
sudo apt install nvidia-driver-535

# DGX는 보통 DGX OS에 미리 설치됨 (MDS가 셋업)
```

### 1.4 nvidia-smi

GPU 상태를 보는 가장 기본적인 도구.

```bash
nvidia-smi
# +---------+--------+-------+
# | GPU 0   | H100   | 80GB  |
# | Util    | 95%    | Temp 70C |
# | Process | 12345 python3 60GB |
# +---------+--------+-------+
```

알아둘 것:
- `Util`: SM(Streaming Multiprocessor) 사용률
- `Memory`: HBM 사용량
- `Power`: 전력 소비 (H100은 700W까지)
- `nvidia-smi dmon`: 실시간 모니터링
- `nvidia-smi topo -m`: GPU 간 토폴로지 (NVLink/PCIe)

---

## 2. GPU 메모리 계층

### 2.1 종류

| 메모리 | 위치 | 크기 | 대역폭 |
|--------|------|------|--------|
| Register | SM 내부 | 작음 | 최고 |
| Shared Memory / L1 | SM 내부 | ~228KB | 매우 빠름 |
| L2 Cache | GPU 칩 | 50MB (H100) | 빠름 |
| **HBM (Global Memory)** | GPU 보드 | **80GB (H100)** | ~3TB/s |
| Host RAM (System) | 메인보드 | 1TB+ | 50GB/s (PCIe5) |
| NVMe SSD | 디스크 | TB+ | 7GB/s |

### 2.2 GPU 메모리가 작아 보이는 이유

- LLM 학습 시: 모델 + activations + gradients + optimizer states = 모델 크기의 4~16배
- 70B 모델 → FP16 weight만 140GB → 여러 GPU 필요

### 2.3 Unified Memory

```cuda
cudaMallocManaged(&ptr, size);
// CPU/GPU 어디서든 같은 포인터로 접근 가능
// 드라이버가 페이지 단위로 자동 마이그레이션
```

편리하지만 명시적 cudaMemcpy보다 성능 변동성이 큼.

---

## 3. NVLink와 NVSwitch (★ DGX의 핵심)

### 3.1 PCIe로는 부족하다

전통적 GPU 서버:
```
GPU 0 ── PCIe (32GB/s) ── CPU ── PCIe ── GPU 1
```

GPU 간 통신마다 CPU/메모리를 거침. 멀티 GPU 학습이 느림.

### 3.2 NVLink

NVIDIA가 만든 GPU 간 직접 연결.

```
DGX H100 (GPU 8장):
GPU0 ─NVLink─ GPU1
  │             │
NVLink        NVLink
  │             │
GPU2 ─NVLink─ GPU3
...

NVLink 4세대: 한 링크 25GB/s × 18개 = 450GB/s per GPU
```

대역폭이 PCIe보다 10배 이상.

### 3.3 NVSwitch

GPU가 8개라면 다 직접 연결 시 28쌍의 링크가 필요. 비현실적.

→ **NVSwitch**: GPU용 가상 스위치 칩.

```
GPU0 ─┐
GPU1 ─┤
GPU2 ─┤
...   ├── NVSwitch ── 모든 GPU 간 풀 대역폭 통신
GPU7 ─┘

DGX H100: NVSwitch 4개 = total 7.2 TB/s 양방향
```

→ 노드 안의 GPU 8개가 **같은 보드인 것처럼** 통신.

### 3.4 NVLink의 한계와 IB의 역할

NVLink는 **노드 안에서만** 동작.

```
[DGX01]                 [DGX02]
GPU0─NVLink─GPU1        GPU0─NVLink─GPU1
   ↑                       ↑
   └── ??? ────────────────┘
   여기는 NVLink 안 됨
```

→ 노드 간은 **InfiniBand**가 담당. 그래서 IB가 멀티노드 학습의 핵심.

---

## 4. GPUDirect (★ 진짜 마법)

### 4.1 일반적인 GPU + NIC 통신

```
[GPU 메모리]
   ↓ cudaMemcpy (GPU→Host)
[Host RAM]
   ↓ socket send / RDMA send
[NIC]
```

문제: **GPU 메모리 → CPU 메모리 → NIC** (2번 복사, 큰 오버헤드).

### 4.2 GPUDirect RDMA

```
[GPU 메모리]
   ↓ NIC이 PCIe로 GPU 메모리에 직접 DMA
[NIC]   ← Host RAM 안 거침!
```

필요 조건:
- 같은 PCIe root complex 아래 GPU와 NIC이 있어야 (DGX는 그렇게 설계됨)
- 커널 모듈 `nvidia-peermem` 로드
- IB NIC이 GPUDirect 지원 (Mellanox ConnectX-5 이상)

NCCL이 자동으로 활용 → AllReduce가 진짜 빠른 이유.

### 4.3 GPUDirect Storage

```
[NVMe SSD] ─PCIe─ [GPU 메모리]
   ↑                    ↑
   └─── 직접 DMA ────────┘
        (CPU/Host RAM 안 거침)
```

ML 데이터셋을 디스크에서 GPU로 로딩할 때 CPU bottleneck 제거.
- cuFile API 사용
- 큰 데이터셋 학습에서 데이터 로딩 병목 해결

### 4.4 GPUDirect P2P

같은 노드 GPU끼리 NVLink/PCIe로 직접 메모리 교환.
- NCCL 노드 내 통신의 기반

---

## 5. DGX H100 한 대의 전체 토폴로지

```
                      [NVSwitch x4]
                           │
    ┌───┬───┬───┬───┬───┬───┬───┬───┐
    │   │   │   │   │   │   │   │   │
   GPU0 GPU1 GPU2 GPU3 GPU4 GPU5 GPU6 GPU7
    │   │   │   │   │   │   │   │   │
   PCIe PCIe PCIe PCIe PCIe PCIe PCIe PCIe
    │   │   │   │   │   │   │   │
   HCA0 HCA1 HCA2 HCA3 HCA4 HCA5 HCA6 HCA7
    │   │   │   │   │   │   │   │
    └───┴───┴───┴───┴───┴───┴───┴───┘
              │
        [IB Quantum-2 Switch]
              │
        다른 DGX 노드들
```

핵심:
- 노드 안: NVSwitch로 GPU 간 풀 대역폭
- GPU 1장당 IB HCA 1장 직접 페어링 (GPUDirect RDMA용)
- 노드 간: IB 스위치

---

## 6. CUDA 프로그래밍 모델 (간단히)

### 6.1 SIMT (Single Instruction Multiple Threads)

```
[Block] ── 수백~수천 thread (같은 코드, 다른 데이터)
[Grid]  ── 수많은 Block
```

GPU는 thread 단위가 아닌 **warp(32 thread 묶음)** 단위로 실행.

### 6.2 메모리 패턴이 성능 좌우

- Coalesced access: 인접 thread가 인접 메모리 → 빠름
- Bank conflict, divergent branch → 느림

PyTorch/TensorFlow는 이런 거 신경 안 써도 cuDNN/cuBLAS가 잘 짜둠. 깊이 들어가면 CUTLASS 같은 라이브러리 직접 작성.

---

## 7. NCCL (이미 다뤘지만 더 깊게)

### 7.1 collective 연산

| 연산 | 의미 |
|------|------|
| AllReduce | 모든 노드의 값 합산 → 모든 노드에 결과 |
| AllGather | 각 노드의 데이터를 모두에게 전파 |
| Broadcast | 한 노드의 값을 모두에게 |
| ReduceScatter | 합산 후 각자에게 일부분만 |

### 7.2 알고리즘

NCCL은 토폴로지를 보고 최적 알고리즘 선택:
- **Ring**: 노드들이 원으로 데이터 전달 (대역폭 효율)
- **Tree**: 트리 구조로 (latency 효율)

### 7.3 NCCL이 IB를 쓰는 흐름

1. PyTorch가 `dist.all_reduce()` 호출
2. NCCL이 GPU 메모리에서 데이터 위치 파악
3. **GPUDirect RDMA**로 IB NIC이 GPU 메모리 직접 DMA
4. IB Switch 통과
5. 반대편 NIC이 그쪽 GPU 메모리에 직접 쓰기

CPU/Host RAM 한 번도 안 거침. 진짜 빠름.

---

## 8. MIG (Multi-Instance GPU)

### 8.1 컨셉

A100/H100에서 GPU 1장을 **하드웨어 레벨로 7개로 분할**.

```
H100 (80GB)
├── MIG 7g.80gb (1개, 전체)
├── MIG 3g.40gb + 3g.40gb (2개, 절반씩)
├── MIG 1g.10gb x7 (7개, 작게)
└── 등 다양한 조합
```

각 instance는:
- 별도 SM과 메모리 슬라이스
- 별도 디바이스 파일
- 독립 실행 (인터페이스가 같지 않음)

### 8.2 K8s 사용

```yaml
resources:
  limits:
    nvidia.com/mig-1g.10gb: 1
```

학습보다는 **인퍼런스 다중 워크로드**에 적합.

---

## 9. NVIDIA GPU Operator

K8s에서 GPU 관련 모든 컴포넌트를 자동 설치/관리.

```
GPU Operator가 설치하는 것:
├── NVIDIA Driver (DaemonSet으로 노드에 설치)
├── NVIDIA Container Toolkit (containerd가 GPU 마운트 가능)
├── NVIDIA Device Plugin (K8s에 GPU 노출)
├── GPU Feature Discovery (노드 라벨링)
├── DCGM Exporter (모니터링)
└── MIG Manager (MIG 설정)
```

수동 설치 안 해도 됨.

### 9.1 NVIDIA Container Toolkit

도커/containerd가 컨테이너에 `--gpus all` 같은 옵션을 처리할 수 있게 해주는 hook.

```toml
# containerd config
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
  [plugins.".....nvidia".options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
```

`nvidia-container-runtime`이 컨테이너 시작 직전에 GPU 디바이스 파일과 드라이버 라이브러리를 마운트.

### 9.2 NVIDIA Device Plugin

K8s에 GPU를 자원으로 노출.

```bash
kubectl describe node dgx01
# Capacity:
#   nvidia.com/gpu: 8
# Allocatable:
#   nvidia.com/gpu: 8
```

스케줄러가 이 정보 보고 GPU 요청한 Pod 배치.

---

## 10. DCGM (Data Center GPU Manager)

### 10.1 무엇

GPU 메트릭/헬스/진단을 수집하는 NVIDIA 공식 도구.

### 10.2 dcgm-exporter

Prometheus 형식으로 메트릭 노출.

```
# 주요 메트릭
DCGM_FI_DEV_GPU_UTIL          # GPU 사용률
DCGM_FI_DEV_MEM_COPY_UTIL     # 메모리 IO 사용률
DCGM_FI_DEV_FB_USED           # HBM 사용량
DCGM_FI_DEV_GPU_TEMP          # 온도
DCGM_FI_DEV_POWER_USAGE       # 전력
DCGM_FI_PROF_SM_ACTIVE        # SM 활성도
DCGM_FI_PROF_PCIE_TX_BYTES    # PCIe 트래픽
DCGM_FI_PROF_NVLINK_TX_BYTES  # NVLink 트래픽
```

### 10.3 모니터링 못 하는 것

- IB 네트워크 (별도 infiniband_exporter 필요)
- Ceph 등 외부 시스템

---

## 11. 트러블슈팅 시나리오

### 11.1 "GPU OOM"

```
RuntimeError: CUDA out of memory.
Tried to allocate 8.00 GiB
```

원인:
- 배치 크기 너무 큼
- gradient accumulation
- 메모리 fragmentation (PYTORCH_CUDA_ALLOC_CONF)

확인:
```bash
nvidia-smi  # 다른 프로세스가 메모리 점유 중인지
```

### 11.2 "GPU Util 낮은데 학습 느림"

→ 입력 파이프라인 병목 또는 통신 병목

진단:
- `nvidia-smi dmon`: SM Active 낮으면 GPU가 놀고 있음
- DataLoader num_workers 늘려보기
- IB 사용률 확인 (멀티노드면 통신 병목)

### 11.3 "GPUDirect RDMA 안 잡힘"

```bash
# nvidia-peermem 모듈 확인
lsmod | grep nvidia_peermem

# 없으면 로드
modprobe nvidia-peermem

# NCCL 로그에서 확인
NCCL_DEBUG=INFO  # 출력에 [send via direct] 보여야 함
```

### 11.4 "노드 토폴로지 확인"

```bash
nvidia-smi topo -m
#       GPU0  GPU1  GPU2  ...  HCA0
# GPU0   X    NV12  NV12       PIX     # NV: NVLink, PIX: same PCIe switch
# GPU1   NV12  X
# ...
```

---

## 12. 우리 회사 환경 정리

### 12.1 DGX 4노드의 GPU 자원

```
4 노드 × 8 GPU = 32 GPU (H100 기준 32 × 80GB = 2.5TB HBM)
4 노드 × 8 HCA = 32 HCA (NDR 400Gbps each)
NVSwitch 풀 대역폭 노드 내부
NVIDIA Quantum-2 IB Switch 노드 간
```

### 12.2 학습 워크로드

```
PyTorchJob → Pod (1 노드 = 8 GPU)
  ├ NCCL backend 사용
  ├ 노드 내: NVSwitch로 8 GPU AllReduce (수 TB/s)
  ├ 노드 간: GPUDirect RDMA로 IB AllReduce (400 Gbps × 8 = 3.2 Tbps)
```

### 12.3 모니터링

```
GPU Operator 배포 시 자동 설치되는 것들:
  - DCGM exporter → Prometheus → Grafana
  - GPU Util, 메모리, 온도, 전력
  - NVLink 트래픽
```

---

## 13. 면접 예상 질문

### Q1. "왜 DGX 한 대가 일반 GPU 서버 8대보다 비싼가요?"
> "단순히 GPU 8장이 들어 있어서가 아니라, 그 8장을 NVSwitch로 풀 대역폭(7.2TB/s) 연결하고, GPU 1장당 IB HCA 1장씩 직접 페어링해서 GPUDirect RDMA가 가능하게 만든 통합 시스템이기 때문입니다. 일반 서버 8대를 묶으면 노드 간은 IB로 연결되지만 노드 내부에서도 NVLink가 약하고 IB 대역폭도 GPU당 할당되지 않아 분산 학습 시 통신이 병목이 됩니다."

### Q2. "GPUDirect RDMA가 정확히 뭔가요?"
> "GPU 메모리와 IB NIC이 같은 PCIe root complex 아래 있을 때, NIC이 GPU 메모리에 직접 DMA로 접근할 수 있게 하는 기술입니다. 일반 통신은 GPU → Host RAM → NIC으로 두 번 복사가 필요한데, GPUDirect RDMA는 한 번에 갑니다. 활성화하려면 nvidia-peermem 커널 모듈이 로드되어야 하고, NCCL은 자동으로 활용합니다. 우리 DGX 환경의 분산 학습 성능 핵심입니다."

### Q3. "GPU는 왜 cgroup으로 제한 못 하나요?"
> "cgroup은 커널이 직접 관리하는 자원에만 적용됩니다. GPU의 메모리/연산은 NVIDIA 드라이버가 자체 관리하기 때문에 cgroup 영역 밖입니다. K8s에서는 NVIDIA Device Plugin이 GPU를 'nvidia.com/gpu' 같은 extended resource로 노출하고, 컨테이너 시작 시 nvidia-container-runtime이 드라이버 라이브러리와 디바이스 파일을 마운트하는 방식으로 격리합니다."

### Q4. "MIG는 언제 쓰나요?"
> "GPU 1장의 SM과 메모리를 하드웨어 레벨로 여러 인스턴스로 쪼개는 기술입니다. 한 학습 잡이 GPU를 다 쓰지 못하는 인퍼런스 워크로드, 노트북 사용자 같은 다중 테넌트에 적합합니다. 학습은 보통 GPU 풀 대역폭이 필요해서 MIG보다는 통째로 쓰는 게 낫습니다."

### Q5. "NCCL 통신이 느립니다. 어떻게 진단하나요?"
> "먼저 NCCL_DEBUG=INFO로 IB transport가 잡혔는지, GPUDirect가 활성화됐는지 확인합니다. 그다음 nccl-tests의 all_reduce_perf로 실측 대역폭을 봅니다. 노드 간이 느리면 IB 쪽: ib_write_bw, infiniband_exporter 메트릭(에러 카운터, 대역폭)을 확인. 노드 내가 느리면 nvidia-smi topo로 NVLink가 제대로 잡혔는지 봅니다."

---

## 14. 한 줄 요약

> **"DGX는 GPU 8장을 NVSwitch로 풀 대역폭 묶고 GPU당 IB HCA 1장으로 GPUDirect RDMA를 가능하게 만든 시스템이다. 노드 내부는 NVLink/NVSwitch가, 노드 간은 IB가 책임지며, NCCL은 두 영역 모두에서 GPU 메모리 → 다른 GPU 메모리로 CPU 개입 없이 데이터를 흘려보낸다. K8s에서는 NVIDIA GPU Operator가 드라이버부터 device plugin까지 전부 자동 관리한다."**
