# DGX 클러스터 InfiniBand 멀티노드 트레이닝 - 기초부터 우리 회사 구성까지

> **목적**: InfiniBand, Multus, RDMA, NAD, NCCL 같은 개념을 처음 보는 사람도 이해할 수 있도록 기초부터 쌓아 올리고, 마지막에 우리 회사 DGX 클러스터 구성을 완전히 해석할 수 있게 만드는 문서.

---

## 0-A. 업무 분담 (★ 면접 시 명확히 구분해서 말할 것)

이 클러스터는 두 주체가 나눠서 구축했습니다.

### MDS (벤더사) 담당 — Day 0 인프라 구축

IDC 현장에서 물리/네트워크/OS 레벨까지 셋업을 완료해 **"Kubernetes를 올릴 수 있는 상태"**로 인계.

| 영역 | 작업 내용 |
|------|----------|
| 하드웨어 설치 | DGX 4노드 랙 마운트, 전원/쿨링 |
| InfiniBand 물리 구성 | NVIDIA Quantum-2 IB 스위치 설치, 케이블링(노드당 8개 = 총 32개) |
| IB 네트워크 구성 | Subnet Manager(opensm) 설정, GID/SL/TC 등 IB fabric 파라미터 |
| OS 설치 | Ubuntu / DGX OS 설치 |
| 드라이버 | NVIDIA GPU 드라이버, MOFED (Mellanox OFED) 드라이버 설치 |
| 검증 | `ib_write_bw`, `nccl-tests` 등으로 노드 간 IB 대역폭/지연 실측 검증 |
| 일반 NIC | 관리망 이더넷 설정 |

→ MDS가 인계한 시점에서 **베어메탈 레벨에서 RDMA 통신은 이미 동작하는 상태**.

### 우리 회사 담당 — Day 1+ 플랫폼 구축/운영

MDS가 넘긴 베어메탈 위에 **Kubernetes 기반 ML 플랫폼**을 구축하고 운영.

| 영역 | 작업 내용 |
|------|----------|
| Kubernetes | v1.26.5 클러스터 구성, control plane / worker 구성 |
| 기본 CNI | Calico 설치 (Pod 네트워크) |
| IB 통합 | NVIDIA Network Operator 배포 → Multus + Host-Device CNI + RDMA Shared Device Plugin |
| NAD 설계 | Namespace ↔ NAD ↔ 물리 IB 포트 전속 매핑 정책 수립 및 NAD 리소스 작성 |
| 학습 플랫폼 | Kubeflow Training Operator 배포 (PyTorchJob CRD) |
| 운영 정책 | namespace별 IB 포트 할당, GPU 리소스 쿼터, Istio 예외 정책 등 |
| 사용자 가이드 | PyTorchJob YAML 템플릿 / NCCL 환경변수 표준화 / 멀티노드 확장 패턴 (Worker2/3...) |
| 모니터링 | (구축 예정/구축됨) Prometheus + dcgm-exporter + infiniband_exporter |

### 책임 분기점 한 줄 요약

> **MDS = 베어메탈에서 RDMA가 동작하는 상태까지**
> **우리 회사 = 그 위에 K8s를 올려 멀티 테넌트 ML 학습 플랫폼으로 만드는 전 과정**

면접에서 "이 환경 누가 셋업했나요?" 질문 받으면:
> "물리 인프라(DGX, IB 스위치, MOFED, opensm)는 MDS가 IDC에서 구축해서 인계해줬고, 그 위의 Kubernetes 클러스터 구성부터 Network Operator를 통한 IB 통합, NAD 설계, PyTorchJob 운영 가이드까지는 저희 팀에서 직접 했습니다."

이렇게 답하면 책임 범위가 명확해집니다.

---

## 0. 큰 그림 먼저

우리가 풀려는 문제는 단 하나입니다.

> **"여러 대의 GPU 서버를 묶어서 하나의 큰 모델을 학습시키고 싶다"**

이걸 위해 필요한 것:

1. **빠른 네트워크** (서버 간 GPU가 데이터를 주고받아야 함) → **InfiniBand**
2. **빠른 네트워크를 컨테이너 안에서 쓰는 방법** → **Multus + Host-Device CNI + RDMA Plugin**
3. **분산 학습을 잘 오케스트레이션 하는 방법** → **Kubeflow PyTorchJob + NCCL**
4. **팀별로 자원을 격리하는 방법** → **Namespace + NAD 매핑**

이 4가지를 차근차근 풀어봅시다.

---

## 1. InfiniBand (IB) 기초

### 1.1 InfiniBand가 뭔가

GPU 서버 간 통신을 **이더넷보다 훨씬 빠르고 지연이 낮게** 해주는 별도의 네트워크 기술입니다.

| 항목 | 이더넷 (10/25/100GbE) | InfiniBand (HDR/NDR) |
|------|----------------------|----------------------|
| 대역폭 | 10~100Gbps | 200~400Gbps |
| 지연(latency) | ~10μs | ~1μs |
| 전송 방식 | TCP/IP (커널 거침) | RDMA (커널 우회) |
| 용도 | 일반 트래픽 | HPC/AI 분산 학습 |

**핵심 개념: RDMA (Remote Direct Memory Access)**
- 원격 서버의 메모리에 **CPU 개입 없이** 직접 읽고 씀
- 커널 네트워크 스택을 거치지 않아 매우 빠름
- AI 학습에서 노드 간 gradient 동기화에 필수적

### 1.2 IB 구성 요소

이더넷과 1:1 대응됩니다.

| InfiniBand | 이더넷 | 역할 |
|------------|--------|------|
| HCA (Host Channel Adapter) | NIC (랜카드) | 서버에 꽂는 카드 |
| IB Switch | L2/L3 스위치 | 패킷 라우팅 |
| QSFP56/112 케이블 | SFP+ 케이블 | 물리 연결 |
| Subnet Manager (opensm) | (없음, ARP가 대체) | 네트워크 경로 중앙 관리 |

### 1.3 케이블 종류

| 종류 | 거리 | 용도 |
|------|------|------|
| **DAC** (Direct Attach Copper) | ~3~5m | 같은 랙 내 단거리 |
| **AOC** (Active Optical Cable) | ~수십m | 랙 간 |
| **광트랜시버 + 광케이블** | ~수백m | 데이터센터 간 |

### 1.4 DGX 서버의 IB 구성

DGX H100 한 대 = **GPU 8장 + HCA 8장**.

```
DGX H100
├── GPU 0 ─ NVLink ─ 다른 GPU와 노드 내 통신
├── GPU 1 ─ ...
├── ...
├── GPU 7
│
├── HCA 0 (mlx5_0) → IB Switch  ← GPU 0 전용
├── HCA 1 (mlx5_1) → IB Switch  ← GPU 1 전용
├── ...
├── HCA 7 (mlx5_7) → IB Switch  ← GPU 7 전용
│
└── 일반 NIC (eth0)  → 관리용 이더넷 스위치
```

**왜 GPU 1장당 HCA 1장인가**
- 8개 GPU가 1개 HCA를 공유하면 → IB 경합 발생
- 각 GPU에 전용 HCA를 줘야 AllReduce 시 병목 없음
- 그래서 DGX 4노드 클러스터 → 32개 케이블이 IB 스위치에 꽂힘

---

## 2. 분산 학습이란 - AllReduce와 NCCL

### 2.1 데이터 분산 학습의 기본 흐름

```
Forward Pass → Loss → Backward Pass → Gradient 계산
                                          ↓
                                   ★ AllReduce ★
                                          ↓
                                  Optimizer Step
```

### 2.2 AllReduce

모든 노드의 gradient를 합쳐 평균을 구한 뒤, 모든 노드가 같은 값을 가지도록 동기화하는 연산.

```
Before:                        After AllReduce (sum):
Node 0: [4, 8]                 Node 0: [12, 16]
Node 1: [2, 4]      →          Node 1: [12, 16]
Node 2: [6, 2]                 Node 2: [12, 16]
Node 3: [0, 2]                 Node 3: [12, 16]
```

이걸 **매 학습 step마다** 수행합니다. 모델이 클수록 gradient도 커서 IB 대역폭이 절대적으로 중요해집니다.

### 2.3 NCCL (NVIDIA Collective Communications Library)

NVIDIA가 만든 GPU 간 통신 라이브러리. AllReduce, AllGather, Broadcast 등을 GPU 친화적으로 구현합니다.

- 노드 내부: **NVLink** 사용 (자동 감지)
- 노드 간: **InfiniBand RDMA** 사용 (자동 감지)
- TCP fallback도 가능 (단, 성능 폭락)

PyTorch의 `torch.distributed` backend로 `nccl`을 지정하면 NCCL이 동작합니다.

---

## 3. Kubernetes 기초 (CNI까지)

### 3.1 Pod와 Namespace

- **Pod**: 컨테이너의 최소 실행 단위 (1개 이상의 컨테이너 묶음)
- **Namespace**: 클러스터 안의 논리적 격리 공간 (팀별/프로젝트별 분리)

### 3.2 CNI (Container Network Interface)

Pod에 네트워크를 붙여주는 표준 인터페이스. K8s는 CNI 플러그인을 통해 Pod에 IP를 할당하고 통신을 가능하게 합니다.

대표적인 CNI 구현:
- **Calico** (우리 회사가 사용) — Pod 간 라우팅, Network Policy 등
- Flannel, Cilium, Weave 등

기본적으로 **Pod 1개 = 네트워크 인터페이스 1개 (eth0)**.

### 3.3 그런데 인터페이스가 1개로는 부족하다

문제 상황:
- Pod의 `eth0`는 K8s 기본 네트워크(Calico 위, 이더넷)
- 그런데 학습용 트래픽은 InfiniBand로 나가야 함
- → **Pod에 인터페이스를 2개 붙여야 함**

이걸 가능하게 하는 게 **Multus**.

---

## 4. Multus - Pod에 인터페이스 여러 개 붙이기

### 4.1 Multus가 하는 일

> "기본 CNI(Calico)는 그대로 두고, **추가 네트워크**를 Pod에 더 붙여줄 수 있게 해주는 메타 CNI"

```
일반 Pod (Multus 없음):
[Pod] ── eth0 (Calico)

Multus 적용 Pod:
[Pod] ── eth0 (Calico)        ← 기본 네트워크 (제어)
       └ net1 (추가 네트워크)  ← IB 등 (데이터)
```

### 4.2 NetworkAttachmentDefinition (NAD)

추가 인터페이스를 어떻게 만들지 정의하는 K8s CRD(커스텀 리소스).

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ib-ibp64s0-conf-slm
  namespace: slm
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "host-device",
      "device": "ibp64s0"
    }
```

해석:
- 이름이 `ib-ibp64s0-conf-slm`인 NAD
- `slm` 네임스페이스에서만 사용
- `host-device` CNI를 써서 노드의 `ibp64s0` 인터페이스를 Pod에 붙임

### 4.3 Pod에서 NAD 사용

```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: ib-ibp64s0-conf-slm  # ← 이거 한 줄
spec:
  containers: [...]
```

이 annotation 한 줄이면 Pod 안에 `net1` 인터페이스로 IB가 붙습니다.

---

## 5. Host-Device CNI - 물리 IB 카드를 Pod에 통째로 주기

### 5.1 동작 방식

- 노드의 물리 네트워크 인터페이스(예: `ibp64s0`)를 **Pod 네임스페이스로 옮김**
- 호스트에서는 그 인터페이스가 안 보이게 됨 (Pod이 독점)
- Pod 안에서는 마치 본인 디바이스처럼 직접 사용

```
Pod 시작 전:
[Host] - ibp64s0 (호스트 소유)

Pod 시작 후:
[Host] - (ibp64s0 사라짐)
[Pod]  - net1 (= ibp64s0, Pod이 독점)
```

### 5.2 장점

- **커널/가상 네트워크 우회** → 진짜 RDMA 성능 그대로
- 오버헤드 거의 없음

### 5.3 단점 (★ 매우 중요 - 면접 단골)

> **물리 디바이스를 Pod이 독점하므로 한 노드의 한 IB 포트는 한 Pod만 사용 가능**

이게 바로 회사 자료의 **"1 Node = 1 Pod = 1 NAD"** 제약의 원인입니다.

대안:
- **SR-IOV**: HCA 1장을 가상 함수(VF) 여러 개로 쪼개서 여러 Pod에 나눠줌 (VF 개수만큼 공유 가능)
- 우리 회사는 SR-IOV가 아닌 Host-Device 방식 채택 → 격리/성능은 좋지만 공유 불가

---

## 6. RDMA Shared Device Plugin

### 6.1 K8s Device Plugin이란

GPU, FPGA, RDMA 카드 같은 특수 하드웨어를 K8s가 관리할 수 있게 노출시켜주는 플러그인.

```yaml
spec:
  containers:
  - resources:
      limits:
        nvidia.com/gpu: 8                       # NVIDIA GPU Plugin
        rdma/hca_shared_devices_a: 1            # Mellanox RDMA Plugin
```

### 6.2 Mellanox k8s-rdma-shared-dev-plugin

- 노드의 RDMA HCA를 K8s 리소스로 노출
- Pod이 위 `limits` 명시하면 → RDMA 디바이스 파일(`/dev/infiniband/*`)에 접근 가능

### 6.3 Host-Device CNI vs RDMA Plugin (역할 분리)

| 컴포넌트 | 역할 |
|---------|------|
| **Multus + Host-Device CNI** | IB 인터페이스(L2/L3 네트워크)를 Pod에 붙임 |
| **RDMA Device Plugin** | IB 디바이스 파일(`/dev/infiniband/uverbs*`)을 Pod에 마운트 |

**둘 다 있어야** Pod이 RDMA를 쓸 수 있습니다.

---

## 7. NVIDIA Network Operator

위의 RDMA Plugin, Host-Device CNI, Multus 등을 한꺼번에 설치/관리해주는 K8s Operator.

```
NVIDIA Network Operator가 설치/관리하는 것들:
  ├── MOFED 드라이버 (Mellanox OFED)
  ├── RDMA Shared Device Plugin
  ├── Multus
  ├── Host-Device CNI
  └── (옵션) IB Kubernetes, NV-IPAM 등
```

수동으로 일일이 설치할 필요 없이 Operator가 알아서 해줍니다.

---

## 8. Kubeflow PyTorchJob

### 8.1 PyTorchJob CRD

K8s 위에서 PyTorch 분산 학습을 선언적으로 정의하는 리소스.

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: train-job
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template: [...]
    Worker:
      replicas: 3   # ← 일반적으론 이렇게 늘림
      template: [...]
```

Training Operator가 자동으로:
- Master/Worker Pod 생성
- 환경 변수(`MASTER_ADDR`, `RANK`, `WORLD_SIZE`) 주입
- Pod 상태 추적

### 8.2 우리 회사의 변형 (★ 핵심)

일반적으로는 Worker `replicas: 3` 같이 늘리면 끝.

**그런데 우리는 안 됩니다.** 왜?
- Host-Device CNI는 1 Node = 1 Pod = 1 NAD 제약
- Worker `replicas`를 늘리면 같은 NAD를 여러 Pod이 쓰려고 함 → 충돌

**해결책**: Replica Type 자체를 늘림.

```yaml
spec:
  pytorchReplicaSpecs:
    Master:  { replicas: 1, ... NAD: ib-ibp64s0-conf-slm }
    Worker:  { replicas: 1, ... NAD: ib-ibp64s0-conf-slm }
    Worker2: { replicas: 1, ... NAD: ib-ibp64s0-conf-slm }
    Worker3: { replicas: 1, ... NAD: ib-ibp64s0-conf-slm }
```

각 Replica Type은 K8s 스케줄러에 의해 다른 노드에 배치되고, 각자 그 노드의 IB 포트를 독점 사용합니다.

---

## 9. NCCL 환경 변수 해석

회사 자료의 환경 변수들을 한 줄씩 풀어보면:

| 변수 | 값 | 의미 |
|------|-----|------|
| `NCCL_DEBUG` | `INFO` | 디버그 로그. 트러블슈팅 시 필수 |
| `NCCL_IB_DISABLE` | `0` | IB 사용 ON (`1`이면 TCP로 떨어져서 성능 폭락) |
| `NCCL_IB_HCA` | `mlx5_0,...,mlx5_11` | 사용할 HCA 명시. DGX의 모든 HCA 나열 |
| `NCCL_SOCKET_IFNAME` | `net,^lo,^eth,...` | 컨트롤 소켓이 쓸 인터페이스. `net*`만 허용, `lo`/`eth*` 제외 |
| `NCCL_IB_GID_INDEX` | `3` | RoCEv2 GID 인덱스. 네이티브 IB는 0, RoCE는 3 |
| `NCCL_IB_TC` | `106` | Traffic Class (QoS 정책) |
| `NCCL_IB_SL` | `3` | Service Level (Subnet Manager가 SL→VL 매핑) |
| `NCCL_IB_TIMEOUT` | `22` | 연결 타임아웃 (값 4 = 4*4.096μs * 2^value) |

**반드시 모든 Master/Worker Pod에 동일하게 설정**해야 합니다. 한 Pod만 다르면 통신이 안 맞아 hang 발생.

---

## 10. Istio Sidecar 비활성화 (왜 중요한가)

```yaml
annotations:
  sidecar.istio.io/inject: "false"
```

**이유**:
- Istio는 Pod에 Envoy 프록시를 자동 주입하는 service mesh
- Envoy는 TCP/HTTP 트래픽을 가로채 라우팅
- 그런데 RDMA는 커널을 우회 → Envoy 입장에서 보이지 않음
- Envoy가 net1을 잡으려 시도하면 → 통신 깨짐 또는 hang

따라서 IB 쓰는 Pod은 **반드시 Istio injection 비활성화**.

---

## 11. 우리 회사 구성 완전 해석

### 11.1 하드웨어

```
DGX 01, 02, 03, 04 (4노드)
  각 노드: GPU 8 + HCA 8
  → 클러스터 전체 IB 케이블 32개 → NVIDIA Quantum-2 IB Switch
```

### 11.2 K8s 스택

```
Kubernetes v1.26.5
├── Calico (기본 CNI, eth0)
├── NVIDIA Network Operator
│   ├── Multus
│   ├── Host-Device CNI
│   └── Mellanox RDMA Shared Device Plugin
└── Kubeflow Training Operator (PyTorchJob)
```

### 11.3 Pod 네트워크 이중화

```
[PyTorch Worker Pod]
├── eth0  (Calico)        → K8s API, 제어 트래픽
└── net1  (Multus + Host-Device) → IB HCA → Quantum-2 Switch
                                    → NCCL AllReduce, Gradient Sync
```

### 11.4 Namespace ↔ NAD ↔ IB 포트 매핑 (★ 사용자가 헷갈렸던 부분)

| Namespace | NAD | Physical Interface |
|-----------|-----|-----|
| slm | ib-ibp64s0-conf-slm | ibp64s0 |
| slm | ib-ibp79s0-conf-slm | ibp79s0 |
| semi-parametric | ib-ibp94s0-conf-semi-parametric | ibp94s0 |
| vlm | ib-ibp192s0-conf-vlm | ibp192s0 |
| mlops | ib-ibp206s0-conf-mlops | ibp206s0 |
| nia | ib-ibp154s0-conf-nia | ibp154s0 |
| (Ceph 전용) | - | ibp24s0, ibp220s0 |

### 11.5 "GPU당 8개씩 32개 케이블이 꽂혀있는데, 왜 표에는 namespace당 1~2개 포트뿐인가?"

**답**:
- 물리적으로는 모든 노드에 8개 HCA가 모두 케이블로 연결되어 있음 (총 32개)
- 다만 **각 namespace에는 그 8개 중 일부만 할당**되어 있는 것
- 인터페이스 이름(`ibp64s0` 등)은 **PCI 주소 기반 명명**이라 모든 DGX 노드에서 동일한 이름이 존재
- 즉 `slm` 네임스페이스가 `ibp64s0`을 쓴다 = 어느 DGX 노드에 스케줄되든 그 노드의 `ibp64s0`을 사용

```
DGX01: ibp24, ibp64, ibp79, ibp94, ibp154, ibp192, ibp206, ibp220  (8개 HCA)
DGX02: ibp24, ibp64, ibp79, ibp94, ibp154, ibp192, ibp206, ibp220  (8개 HCA, 동일 명명)
DGX03: 동일
DGX04: 동일

slm 팀에 할당된 것: 모든 노드의 ibp64, ibp79  (노드당 2개)
mlops 팀에 할당된 것: 모든 노드의 ibp206     (노드당 1개)
Ceph 전용:           모든 노드의 ibp24, ibp220
```

→ 각 팀(namespace)이 노드당 1~2개의 IB 포트를 **전속 사용**하는 구조.
→ 8개 HCA를 8개 namespace에 나눠 쓰는 격리 모델.

### 11.6 학습 잡 실행 흐름 (예: slm 팀이 4노드 학습)

1. 사용자가 PyTorchJob YAML 작성 (Master + Worker + Worker2 + Worker3)
2. 모든 Pod의 annotation에 `k8s.v1.cni.cncf.io/networks: ib-ibp64s0-conf-slm`
3. 모든 Pod에 `rdma/hca_shared_devices_a: 1` 리소스 요청
4. K8s 스케줄러가 4개 Pod을 4개 DGX 노드에 분산 배치
5. Multus가 각 노드의 `ibp64s0`을 해당 Pod의 `net1`로 이동
6. RDMA Plugin이 `/dev/infiniband/*` 디바이스를 Pod에 마운트
7. PyTorch가 NCCL backend로 분산 학습 시작
8. NCCL이 `mlx5_*` HCA를 통해 IB Switch 거쳐 노드 간 AllReduce 수행

---

## 12. 면접 예상 질문 & 답

### Q1. "Multus는 왜 필요한가요?"
> Pod의 기본 CNI(Calico)는 인터페이스를 1개만 붙여줍니다. 그런데 분산 학습은 제어 트래픽(K8s API, 컨테이너 관리)과 데이터 트래픽(GPU 간 gradient sync)을 분리해야 합니다. 데이터 트래픽은 InfiniBand로 보내야 하기 때문에, Multus를 통해 Pod에 IB 인터페이스를 추가로 붙여줍니다.

### Q2. "왜 SR-IOV가 아니라 Host-Device를 썼나요?"
> SR-IOV는 HCA 1장을 VF로 쪼개 여러 Pod이 공유할 수 있지만, 설정이 복잡하고 노드별 VF 개수 제약이 있습니다. 우리는 namespace별 격리와 성능을 우선해, 물리 포트를 통째로 할당하는 Host-Device 방식을 선택했습니다. 트레이드오프로 1 Node = 1 Pod = 1 NAD 제약이 생깁니다.

### Q3. "PyTorchJob에서 왜 Worker.replicas를 안 늘리고 Worker2, Worker3를 만드나요?"
> Host-Device CNI는 물리 인터페이스를 Pod 1개가 독점합니다. Worker.replicas=3을 하면 같은 NAD를 가리키는 Pod 3개가 같은 노드에서 같은 IB 포트를 쓰려고 충돌합니다. 그래서 Replica Type을 분리해, K8s 스케줄러가 각각 다른 노드에 배치되도록 유도합니다.

### Q4. "Istio sidecar를 왜 비활성화 하나요?"
> Istio Envoy는 TCP 트래픽을 가로채 라우팅합니다. 하지만 RDMA는 커널을 우회하기 때문에 Envoy 관점에서는 비정상 트래픽으로 보이고, 실제로는 net1 인터페이스에 개입해서 통신이 hang 되거나 깨집니다. 그래서 IB Pod은 반드시 sidecar.istio.io/inject: false 설정이 필요합니다.

### Q5. "AllReduce가 뭔가요?"
> 분산 학습에서 모든 노드의 gradient를 합산해 평균을 구한 뒤, 그 결과를 모든 노드가 갖도록 동기화하는 collective 통신 연산입니다. 이게 매 step마다 일어나기 때문에 IB 대역폭이 학습 속도에 직접적인 영향을 줍니다.

### Q6. "NCCL_IB_HCA를 명시하는 이유는?"
> NCCL이 자동 탐색을 하긴 하지만, 멀티 HCA 환경에서 의도와 다른 HCA를 잡거나 일부만 쓰는 경우가 있습니다. DGX의 8개 HCA를 모두 사용하도록 명시하면 GPU별로 전용 HCA를 통해 통신하게 되어 병목을 최소화합니다.

### Q7. "IB 모니터링은 어떻게 하나요?"
> DCGM은 GPU 전용이라 IB는 못 봅니다. Prometheus + infiniband_exporter로 포트별 rx/tx, 에러 카운터(symbol_error, port_rcv_errors, link_downed)를 수집합니다. ib_write_bw로 노드 간 실측 대역폭을, nccl-tests로 실제 collective 성능을 벤치마크합니다.

### Q8. "노드 간 통신이 hang 걸리면 어디부터 봐야 하나요?"
> ① NCCL_DEBUG=INFO 로그에서 IB transport가 잡혔는지 확인 → ② Pod의 net1 인터페이스가 정상 UP 되어 있는지 → ③ /dev/infiniband 디바이스가 마운트 됐는지 → ④ Istio sidecar 주입이 꺼져 있는지 → ⑤ ib_write_bw로 노드 간 직접 IB 통신 가능한지 → ⑥ Subnet Manager가 동작 중인지(opensm).

---

## 13. 한 줄 요약

> **"DGX 4노드의 32개 HCA를 IB 스위치에 모두 꽂아두고, NVIDIA Network Operator가 깔아주는 Multus + Host-Device CNI + RDMA Plugin으로 각 namespace에 IB 포트를 전속 할당한다. PyTorchJob은 Replica Type을 늘려 노드 확장하며, NCCL이 IB 위에서 AllReduce를 수행한다."**

이 한 문장이 면접에서 술술 나오면 통과입니다.
