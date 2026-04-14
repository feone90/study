# CUDA 스택 딥다이브 — Driver / CUDA / NVML / MIG / MPS / DCGM

> **목적**: `nvidia-smi` 한 줄 뒤의 **driver, user-mode CUDA library, kernel module, device plugin, MIG/MPS 격리, DCGM 메트릭** 계층을 면접 깊이로.
> **선행**: [gpu-gpudirect-deep-dive.md](gpu-gpudirect-deep-dive.md)
> **연결**: [../k8s/nvidia-network-operator-deep-dive.md](../k8s/nvidia-network-operator-deep-dive.md), [../k8s/inference-serving-deep-dive.md](../k8s/inference-serving-deep-dive.md)

---

## 0. 큰 그림

```
┌─────────────────────────────────────────┐
│  사용자 코드 (PyTorch, TensorFlow)        │
│     │ CUDA C API (libcudart.so)          │
│     ▼                                   │
│  libcuda.so (CUDA Driver API)           │  user mode
│     │                                    │
│  ───┼──────── user / kernel 경계 ────────│
│     ▼                                    │
│  nvidia.ko  + nvidia-uvm.ko             │  kernel module
│  + nvidia-drm.ko + nvidia-peermem.ko    │
│     │                                    │
│     ▼                                    │
│  GPU Hardware (H100)                    │
└─────────────────────────────────────────┘
```

추가 계층:
- **NVML** (libnvidia-ml.so) — 관리/모니터링 API.
- **DCGM** — 엔터프라이즈 메트릭/진단 (NVML 위).
- **MIG/MPS** — 멀티 테넌트 GPU 공유.
- **NVIDIA Container Toolkit** — 컨테이너에 GPU 노출.
- **K8s GPU Operator** — 위 전부를 오케스트레이션.

---

## 1. Driver + Kernel Module

### 1.1 모듈 구성

```bash
lsmod | grep nvidia
# nvidia            55058432  ...
# nvidia_uvm        ...           # Unified Memory
# nvidia_drm        ...           # DRM interface
# nvidia_modeset    ...
# nvidia_peermem    ...           # GPUDirect RDMA
```

| 모듈 | 역할 |
|------|------|
| `nvidia` | 메인 드라이버. PCI device 초기화, 메모리, 실행 |
| `nvidia_uvm` | Unified Memory (CPU-GPU 공유 VA) |
| `nvidia_drm` | DRM/KMS (주로 display, 서버에선 `modeset=0`) |
| `nvidia_peermem` | GPU 메모리를 다른 PCIe 디바이스(HCA)에 노출 — **GPUDirect RDMA 필수** |

### 1.2 Driver vs CUDA Toolkit 버전 호환

- **Driver**가 CUDA Toolkit의 **상위 호환 보장**.
- H100 + 535.247.01 driver → CUDA 12.2 runtime 호환 (12.4도 대부분 OK).
- 노드 driver ≥ 컨테이너 CUDA runtime 규칙.

회사: driver **535.247.01**, CUDA runtime **12.2** (GPU Operator 배포).

### 1.3 nvidia-smi가 하는 일

```bash
nvidia-smi
```

- libnvidia-ml.so 호출 → /dev/nvidiactl, /dev/nvidia[0-N] ioctl → 커널 모듈 → GPU 쿼리.
- 읽는 것: 온도, 전력, SM util, 메모리, running processes.
- 쓰는 것 (옵션): `nvidia-smi -pl` 전력 제한, `-lgc` 클럭 락.

---

## 2. CUDA 런타임/드라이버 API

### 2.1 Runtime API vs Driver API

| API | 심볼 | 수준 | 사용처 |
|-----|------|------|--------|
| CUDA Runtime | `cudaMalloc`, `cudaMemcpy` | 고수준 (암시적 context) | 대부분 프레임워크 |
| CUDA Driver | `cuMemAlloc`, `cuLaunchKernel` | 저수준 (명시 context) | JIT 컴파일, 고급 툴 |

### 2.2 CUDA Context / Stream

- **Context**: 한 GPU당 프로세스마다 하나. 메모리, 모듈, 스트림을 소유.
- **Stream**: 비동기 작업 큐. 여러 stream → 병렬 실행 + overlap (compute/copy).
- **Default stream**: null stream. legacy behavior(동기화 암시) → 성능 저하 주의.

### 2.3 Unified Memory

```c
cudaMallocManaged(&ptr, size);
// CPU, GPU 양쪽 접근 가능. page migration 자동.
```

- **장점**: 편의성.
- **단점**: page fault overhead.
- LLM 학습에선 잘 안 씀 (explicit placement가 성능↑).

---

## 3. NVML

### 3.1 역할

**Management library** — nvidia-smi 이면 전체. 외부 툴(DCGM, GPU Operator, Prometheus exporter)이 호출.

### 3.2 주요 쿼리

| 함수 | 반환 |
|------|------|
| `nvmlDeviceGetUtilizationRates` | SM util %, mem util % |
| `nvmlDeviceGetMemoryInfo` | total / used / free |
| `nvmlDeviceGetTemperature` | 온도 |
| `nvmlDeviceGetPowerUsage` | 전력 mW |
| `nvmlDeviceGetPcieThroughput` | PCIe rx/tx |
| `nvmlDeviceGetComputeRunningProcesses` | 실행 중 process + mem |

### 3.3 "GPU util 100%"가 의미하는 것

**주의**: NVML의 `GPU-Util`은 **SM이 활성 상태인 샘플링 비율** 만. **TensorCore 활용도, 메모리 대역폭 사용률은 다른 지표**. DCGM이 이걸 보완.

---

## 4. DCGM (Data Center GPU Manager)

### 4.1 역할

NVML 상위 에이전트 — **진단(diag), 정책(policy), 메트릭(metrics)**.

- `dcgmi diag -r 3` : Level 3 진단 (수 분, GPU 전체 체크)
- `dcgmi health -c` : 실시간 건강 모니터
- **dcgm-exporter** — Prometheus용 메트릭 exporter (gRPC → HTTP :9400).

### 4.2 주요 메트릭 (회사 대시보드에 쓰이는 것들)

| Metric | 의미 |
|--------|------|
| `DCGM_FI_DEV_GPU_UTIL` | SM util (NVML과 동일) |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | 메모리 copy util |
| `DCGM_FI_PROF_SM_ACTIVE` | SM 활성 시간 비율 (profiling) |
| `DCGM_FI_PROF_SM_OCCUPANCY` | 워프 점유율 |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | **TensorCore 사용률** — AI 워크로드 핵심 |
| `DCGM_FI_PROF_DRAM_ACTIVE` | HBM 메모리 대역폭 활성 |
| `DCGM_FI_PROF_NVLINK_TX_BYTES` | NVLink 송신 바이트 |
| `DCGM_FI_PROF_PCIE_TX_BYTES` | PCIe 송신 바이트 |
| `DCGM_FI_DEV_GPU_TEMP` | 온도 (℃) |
| `DCGM_FI_DEV_POWER_USAGE` | 전력 (W) |
| `DCGM_FI_DEV_XID_ERRORS` | **XID 에러** — GPU 하드/소프트 장애 |

### 4.3 "GPU util은 높은데 학습이 느리다"

`GPU_UTIL`이 90%여도 `PIPE_TENSOR_ACTIVE`가 10%면 TensorCore를 못 쓰는 것 (FP32 로 돌거나 커널이 matmul 아님). 이걸 구분 가능한 게 DCGM PROF 시리즈.

### 4.4 XID 에러

GPU 커널이 문제를 만나면 dmesg에 `NVRM: Xid (PCI:0000:..): 31, ...` 같은 로그. 주요 코드:

| XID | 의미 |
|-----|------|
| 13 | Graphics engine exception (SW 버그 가능) |
| 31 | GPU memory page fault |
| 43 | Fallen off the bus (PCIe 링크 다운) — HW 이상 |
| 48 | Double-bit ECC — HW 교체 신호 |
| 63 | ECC page retirement recorded |
| 79 | GPU has fallen off the bus — 재부팅/교체 필요 |

**면접 이야기 연결**: ML-27 dgx04 GPU 과열 셧다운 → dmesg에 XID 63, 79 병행 가능성. DCGM으로 선제 감지했으면 워크로드 drain 후 교체 가능.

---

## 5. NVIDIA Container Toolkit

### 5.1 역할

컨테이너 런타임(containerd, docker, cri-o)이 GPU 디바이스/드라이버를 **마운트**하게 해줌.

- `nvidia-container-runtime` — runc wrapper.
- OCI hook으로 `/dev/nvidia*`, 드라이버 라이브러리(`libcuda.so.*`) 를 컨테이너에 bind mount.

### 5.2 containerd 설정

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
```

RuntimeClass 사용:
```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
```

Pod `spec.runtimeClassName: nvidia`.

### 5.3 GPU Operator가 자동으로 하는 것

- Driver DaemonSet (컨테이너로 드라이버 설치)
- NVIDIA Container Toolkit 설치
- `nvidia-device-plugin` (Extended Resource `nvidia.com/gpu`)
- `dcgm-exporter` (Prometheus 메트릭)
- MIG Manager (MIG configure)
- Node Feature Discovery

---

## 6. Device Plugin 흐름

### 6.1 Extended Resource

kubelet은 GPU를 모름. `nvidia-device-plugin`이 kubelet과 gRPC로 대화:

1. DaemonSet으로 각 노드 배포.
2. `ListAndWatch` — 사용 가능한 GPU ID 리스트 반환.
3. kubelet이 `/etc/kubernetes/manifests` 가 아닌 **node status**에 `nvidia.com/gpu: 8` 노출.
4. scheduler가 이 리소스로 fit 계산.
5. Pod 생성 시 `Allocate` 호출 — 어떤 GPU ID 할당할지 반환.
6. kubelet이 해당 `/dev/nvidia<N>` 를 컨테이너에 bind.

### 6.2 리소스 정수 고정

GPU는 분할 불가 (기본). 1 Pod = 정수 GPU. 소수 불가. MIG/MPS 써야 세분화.

---

## 7. MIG (Multi-Instance GPU)

### 7.1 개념

GPU를 **하드웨어 레벨로 분할**. H100은 최대 **7 instance**.

- SM, L2 cache, DRAM bandwidth 전부 격리.
- 각 MI는 **독립 CUDA context** 가능.
- 다른 인스턴스 장애가 격리됨.

### 7.2 프로파일 (H100 80GB)

| 프로파일 | 메모리 | SM 슬라이스 | 최대 개수 |
|----------|--------|-------------|-----------|
| `1g.10gb` | 10GB | 1/7 | 7 |
| `2g.20gb` | 20GB | 2/7 | 3 |
| `3g.40gb` | 40GB | 3/7 | 2 |
| `4g.40gb` | 40GB | 4/7 | 1 |
| `7g.80gb` | 80GB | 7/7 | 1 |

### 7.3 동작

```
BEFORE: 1 physical GPU = 1 Kubernetes nvidia.com/gpu
AFTER (MIG 3g.40gb × 2):
  2 MIG instances = 2 Kubernetes 리소스 (nvidia.com/mig-3g.40gb: 2)
```

GPU Operator MIG Manager가 자동:
- 노드 라벨 `nvidia.com/mig.config=all-1g.10gb` 붙이면 해당 프로파일로 재구성.
- 재구성 시 GPU 사용 중이면 불가 — drain 선행.

### 7.4 장점/단점

**장점**: 하드웨어 격리 — 멀티 테넌트 안전. 리소스 예측 가능.
**단점**: NVLink는 MIG 안 지나감 — **MIG 간 NCCL 대역폭 저하**. 학습 분산에 부적합. 서빙/추론에 주로.

---

## 8. MPS (Multi-Process Service)

### 8.1 개념

**소프트웨어 기반** 여러 CUDA context를 하나의 GPU에서 병합 실행.

- `nvidia-cuda-mps-control` 데몬 실행 → 여러 client process 연결.
- 커널 실행을 merged context에서 처리 → context switch 오버헤드 제거.

### 8.2 MIG와의 차이

| 축 | MIG | MPS |
|----|-----|-----|
| 격리 | HW | 없음 (best-effort) |
| 장애 격리 | 완전 | 한 process가 OOM하면 전체 영향 |
| 설정 | GPU 재파티션 필요 | daemon 실행만 |
| 성능 | 분할 자원 상한 | 전체 GPU 공유, burst 가능 |
| 사용 | 서빙 멀티 테넌트 | 같은 팀 여러 dev 환경 |

### 8.3 K8s에서 MPS

K8s 1.27+ / GPU Operator가 **time-slicing** 또는 **MPS** mode를 지원:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: device-plugin-config
data:
  config.yaml: |
    version: v1
    sharing:
      mps:
        renameByDefault: true
        resources:
        - name: nvidia.com/gpu
          replicas: 4
```

이 경우 노드가 `nvidia.com/gpu: 4` 로 노출(물리 1개인데). 4 Pod이 공유.

---

## 9. GPU 온도/전력 관리 (ML-27 연관)

### 9.1 Thermal management

- H100 정상 온도: ~60~75℃ 로드 중.
- Throttling: 80℃ 근처에서 clock down.
- Shutdown: 85~92℃ 에서 강제 셧다운 (ML-27 실제 사례).

### 9.2 DCGM alert 셋

```yaml
# Prometheus rule 예
- alert: GPUHighTemp
  expr: DCGM_FI_DEV_GPU_TEMP > 80
  for: 5m
- alert: GPUThermalShutdownImminent
  expr: DCGM_FI_DEV_GPU_TEMP > 88
  for: 1m
  labels:
    severity: critical
```

### 9.3 대응 절차

1. DCGM으로 확인 → dmesg XID 체크.
2. `nvidia-smi -pl 350` (default 700W → 절반)로 즉시 전력 제한 — throttling.
3. 워크로드 drain → 노드 cordon.
4. 벤더 교체 (MDS테크). 회사 사례는 MDS테크 고영호 차장 에스컬레이션.

---

## 10. 면접 Q&A

### Q1 (기초). NVML과 DCGM 차이?
A. NVML = NVIDIA 기본 관리 library. DCGM = NVML 위 엔터프라이즈 agent, **진단(diag), 건강(health), 프로파일 메트릭(TensorCore/DRAM/NVLink 활성)** 제공. 대규모 운영은 DCGM 필수.

### Q2. `nvidia.com/gpu: 1` 이 실제로 동작하는 경로를 5단계로.
A. (1) device plugin이 kubelet에 `ListAndWatch` 응답. (2) kubelet이 노드 status 업데이트. (3) scheduler fit 계산. (4) Pod binding 시 kubelet이 device plugin `Allocate` 호출. (5) nvidia-container-runtime OCI hook이 `/dev/nvidia<N>` + libcuda bind mount.

### Q3. MIG와 MPS 중 학습과 서빙 각각 뭐?
A. 학습은 **둘 다 지양** — MIG는 NVLink 안 통해서 NCCL 불리, MPS는 격리 약해 충돌 시 전체 Job 위험. 서빙은 작은 모델 다수면 **MIG**, 같은 팀 실험 공유엔 **MPS** 또는 time-slicing.

### Q4. H100 MIG 3g.40gb 두 개, 남는 SM 1개는?
A. 7 SM slice 중 3+3=6 사용, 1 slice는 **유휴**. MIG는 재구성하지 않는 한 동적 회수 불가. 프로덕션에선 4g.40gb + 3g.40gb 조합으로 꽉 채우는 게 정석.

### Q5. DCGM `PIPE_TENSOR_ACTIVE` 가 낮다. 해석?
A. SM은 돌고 있지만 TensorCore를 못 쓰는 중. 원인: (1) dtype이 FP32 — AMP/BF16 미사용. (2) 커널이 matmul이 아닌 elementwise 위주. (3) cuBLAS 경로가 아닌 단순 커널. 대책: AMP 활성, 모델 구조 점검.

### Q6. XID 79 나오면?
A. "GPU fallen off the bus". PCIe 링크 다운. 재부팅으로 복구되는 경우도 있지만 **반복되면 하드 교체**. 로그 수집 후 벤더 에스컬레이션. 워크로드는 다른 노드로 이전.

### Q7. nvidia-peermem이 뭐, 왜 필요?
A. GPU 메모리를 다른 PCIe 장치(IB HCA)가 DMA로 직접 접근하게 만드는 커널 모듈. GPUDirect RDMA 필수. NCCL 로그에 `GDRDMA` 안 보이면 이 모듈 로드 여부부터 확인.

### Q8. GPU Operator가 각 GPU 노드에서 띄우는 DaemonSet 리스트?
A. (1) nvidia-driver-daemonset (드라이버 커널 모듈 설치). (2) nvidia-container-toolkit-daemonset. (3) nvidia-device-plugin-daemonset. (4) nvidia-dcgm + dcgm-exporter. (5) gpu-feature-discovery (NFD 라벨). (6) mig-manager (MIG 재구성). (7) validator (설치 후 검증).

### Q9. `nvidia-smi`가 프로세스 목록을 "Unknown"으로 표시. 원인?
A. 컨테이너 안의 PID가 호스트 PID namespace에서 보이지 않아 NVML이 프로세스 정보 얻지 못함. 호스트에서 `nvidia-smi --query-compute-apps` 해도 PID만 나옴. 컨테이너 추적은 K8s Pod 이름과 매핑 필요 (DCGM-exporter 라벨).

### Q10. CUDA 12.4 toolkit 컨테이너인데 driver 535 (12.2 호환). 뜨는가?
A. 상향(newer toolkit, older driver)은 **기본 금지**. 하지만 CUDA의 **Forward Compatibility Package**를 컨테이너에 포함하면 12.4도 535 driver에서 동작. NVIDIA 공식 이미지 일부 지원. 그래도 프로덕션은 driver 업그레이드가 정석.

---

## 11. 체크리스트

- [ ] nvidia 커널 모듈 4개 역할 (nvidia, uvm, drm, peermem)
- [ ] CUDA Runtime vs Driver API 차이
- [ ] DCGM vs NVML, DCGM PROF 메트릭 4개 이상
- [ ] XID 31/43/48/79 의미
- [ ] device plugin 5단계 흐름
- [ ] MIG 프로파일 + NVLink 제약
- [ ] MPS 장단점
- [ ] GPU 과열 대응 순서 (회사 ML-27 연결)
- [ ] GPU Operator DaemonSet 7종
- [ ] CUDA driver 호환성 룰 (상위 호환)

---

## 11.5 2025-2026 최신 키워드 (면접 심화)

- **CDI (Container Device Interface)**: containerd 1.7+/NVIDIA Container Toolkit 1.14+ 이후 OCI hook 대신 CDI spec(`/etc/cdi/nvidia.yaml`) 방식 권장. GPU Operator 24.x 기본값. hook 방식 대비 디버깅/투명성 우수.
- **Confidential Computing (H100 CC mode)**: H100 + CUDA 12.2+에서 `MIG+CC` 지원. VM 외부(호스트/하이퍼바이저)로부터 GPU 메모리 격리. 금융/의료 멀티테넌트 시 질문 가능.
- **NVSHMEM / NCCL 2.20+ SHARP v3**: NVLink/IB 초월 collective 최적화. H100+Quantum-2 환경에서 AllReduce 지연 20%↓.
- **nvidia-persistenced**: 드라이버 persistence mode. 미활성 시 첫 CUDA init 수백 ms 지연 — LLM 서빙 cold start 문제의 숨은 원인.
- **MIG + vGPU 조합**: vGPU 17.x (2024) 부터 MIG와 결합 가능 — VDI 겸용 클러스터에 유용.

## 12. 연계 문서

- HW/링크 계층: [./gpu-gpudirect-deep-dive.md](./gpu-gpudirect-deep-dive.md), [./nccl-collective-deep-dive.md](./nccl-collective-deep-dive.md)
- 컨테이너/런타임: [../kernel/container-runtime-deep-dive.md](../kernel/container-runtime-deep-dive.md), [../k8s/nvidia-network-operator-deep-dive.md](../k8s/nvidia-network-operator-deep-dive.md)
- 서빙/관측: [../k8s/inference-serving-deep-dive.md](../k8s/inference-serving-deep-dive.md), [../k8s/observability-deep-dive.md](../k8s/observability-deep-dive.md)
- 멀티테넌트 스케줄(MIG/MPS 할당): [../k8s/multi-tenancy-scheduler-deep-dive.md](../k8s/multi-tenancy-scheduler-deep-dive.md)
- 장애 서사(ML-27 과열): [../../interview/ml-platform-ownership-guide.md](../../interview/ml-platform-ownership-guide.md)
- 허브: [../integration/dgx-ib-multinode-training-guide.md](../integration/dgx-ib-multinode-training-guide.md)

## 13. 참고

- NVML: https://docs.nvidia.com/deploy/nvml-api/
- DCGM: https://docs.nvidia.com/datacenter/dcgm/latest/
- MIG 가이드: https://docs.nvidia.com/datacenter/tesla/mig-user-guide/
- XID 코드표: NVIDIA "Useful NVIDIA Toolkit Commands" 문서
- 관련: [gpu-gpudirect-deep-dive.md](gpu-gpudirect-deep-dive.md), [../k8s/inference-serving-deep-dive.md](../k8s/inference-serving-deep-dive.md), [../k8s/observability-deep-dive.md](../k8s/observability-deep-dive.md)
