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

### 1.5 실제 dgx03 노드 출력 해석 (★ 면접에서 "설명해보세요" 나오면 이걸로)

DGX03 노드에서 실제 조회한 결과를 **교과서 그림과 비교**하면서 읽어보자.

#### (A) `lspci | grep -i mellanox` — 하드웨어 카드 식별

```
18:00.0 Infiniband controller  [ConnectX-7]
29:00.0 Infiniband controller  [ConnectX-7]
29:00.1 Ethernet controller    [ConnectX-7]   ← 같은 카드(29:00)의 두 번째 포트
40:00.0 Infiniband controller  [ConnectX-7]
4f:00.0 Infiniband controller  [ConnectX-7]
5e:00.0 Infiniband controller  [ConnectX-7]
9a:00.0 Infiniband controller  [ConnectX-7]
aa:00.0 Infiniband controller  [ConnectX-7]
aa:00.1 Ethernet controller    [ConnectX-7]   ← 같은 카드(aa:00)의 두 번째 포트
c0:00.0 Infiniband controller  [ConnectX-7]
ce:00.0 Infiniband controller  [ConnectX-7]
dc:00.0 Infiniband controller  [ConnectX-7]
```

**해석**:
- ConnectX-7 카드는 총 **12장** 보임. "GPU 1장당 HCA 1장 = 8장"이라는 설명은 **Compute Fabric(IB) 전용 HCA** 기준이고, 실제 DGX H100은 여기에 **Storage/In-Band Fabric용 듀얼포트 HCA 2장을 추가**로 더 가짐.
- `29:00.0` + `29:00.1`은 **같은 물리 카드**. 포트 1은 IB, 포트 2는 Ethernet으로 설정된 Dual-port ConnectX-7. `aa:00`도 동일 구조.
- 수많은 `PCI bridge` 라인은 ConnectX-7 칩 내부의 **PCIe 스위치**. 한 카드가 여러 function을 가지기 위한 내부 브릿지라 신경 안 써도 됨.

→ **12 HCA = 8 (Compute) + 2 (Storage Dual-port, IB+Eth) + 2 (Storage 추가)** 구조. DGX H100 레퍼런스 토폴로지와 일치.

#### (B) `ibdev2netdev` — RDMA device ↔ net interface 매핑

```
mlx5_0   port 1 ==> ibp24s0          (Up)     ← bus 0x18 = 24
mlx5_1   port 1 ==> ibp41s0f0        (Down)   ← bus 0x29, function 0 (IB 포트)
mlx5_2   port 1 ==> enp41s0f1np1     (Down)   ← bus 0x29, function 1 (Ethernet 포트)
mlx5_3   port 1 ==> ibp64s0          (Up)     ← bus 0x40 = 64
mlx5_4   port 1 ==> ibp79s0          (Up)     ← bus 0x4f = 79
mlx5_5   port 1 ==> ibp94s0          (Up)     ← bus 0x5e = 94
mlx5_6   port 1 ==> ibp154s0         (Up)     ← bus 0x9a = 154
mlx5_7   port 1 ==> ibp170s0f0       (Down)   ← bus 0xaa, function 0 (IB)
mlx5_8   port 1 ==> enp170s0f1np1    (Down)   ← bus 0xaa, function 1 (Ethernet)
mlx5_9   port 1 ==> ibp192s0         (Up)     ← bus 0xc0 = 192
mlx5_10  port 1 ==> ibp206s0         (Up)     ← bus 0xce = 206
mlx5_11  port 1 ==> ibp220s0         (Up)     ← bus 0xdc = 220

irdma0   port 1 ==> ens6f0           (Up)     ← Intel NIC (mgmt용)
irdma1   port 1 ==> ens6f1           (Down)
```

**인터페이스 이름 규칙** (systemd Predictable Network Interface Names):
- `ib` = InfiniBand link type
- `en` = Ethernet link type
- `p<N>` = PCI bus 번호 (16진수를 10진수로) — `p64` = bus 0x40
- `s<N>` = PCI slot
- `f<N>` = function (dual-port 카드의 포트 구분)
- `np<N>` = Mellanox 네트워크 port suffix (ethernet 변형 시)

**핵심 관찰**:
1. **Compute IB 8개가 모두 Active**: `mlx5_0, 3, 4, 5, 6, 9, 10, 11` → GPU 8장에 1:1 매핑되는 400Gbps NDR 링크.
2. **Down 4개는 예상된 상태**:
   - `mlx5_1 (ibp41s0f0)` / `mlx5_7 (ibp170s0f0)` = Storage/In-Band 카드의 IB 포트. 이 사이트에서는 미사용으로 Disabled.
   - `mlx5_2` / `mlx5_8` = 같은 카드의 Ethernet 포트. 역시 미결선.
3. **`irdma0/1`** = 메인보드 온보드 Intel NIC (RDMA 지원). 관리 네트워크용이라 Compute 패브릭과 무관. iWARP/RoCE 구현이 다름.
4. **회사 values.yaml의 `rdmaSharedDevicePlugin.vendors: [15b3]`이 `irdma`를 걸러내는 이유** — `15b3`은 Mellanox 벤더 ID. Intel(`8086`)은 자동 제외됨.

#### (C) `ibstat` — 각 HCA 상세 상태

발췌 (Active인 것 하나만):
```
CA 'mlx5_3'
        CA type: MT4129                  ← ConnectX-7
        Firmware version: 28.39.3004
        Port 1:
                State: Active
                Physical state: LinkUp
                Rate: 400                ← 400 Gbps NDR
                Base lid: 25             ← subnet manager가 할당한 LID
                SM lid: 2                ← Subnet Manager의 LID
                Port GUID: 0x58a2e1030046d3a6
                Link layer: InfiniBand
```

**해석 포인트**:
- `CA type: MT4129` = ConnectX-7 NDR 400Gbps 모델.
- `Rate: 400` = NDR 400Gbps 링크가 정상 협상됨.
- `SM lid: 2` = 패브릭 내 Subnet Manager가 LID 2에 존재. **모든 Active HCA가 같은 SM lid(2)를 보는지 확인** → 패브릭이 하나의 subnet으로 잘 묶여 있다는 증거.
- `Base lid`가 포트별로 다름 (18~25) → SM이 각 포트에 고유 LID 부여. LID = IB 세계의 "IP"처럼 라우팅용 주소.
- `Port GUID` = HCA 포트의 MAC에 해당. 패브릭 상 영구 식별자. UFM/SM에서 Pod과 HCA 추적할 때 이걸 봄.

**Down인 `mlx5_1`은**:
```
Port 1: State: Down, Physical state: Disabled, SM lid: 0
```
→ SM이 이 포트를 아예 못 보는 상태. 케이블이 빠졌거나 스위치 포트가 꺼짐.

#### (D) `rdma link` — 커널 RDMA subsystem 관점

```
link mlx5_0/1  subnet_prefix fe80::  lid 23  state ACTIVE  physical_state LINK_UP
link mlx5_2/1                                 state DOWN   physical_state DISABLED  netdev enp41s0f1np1
link mlx5_3/1  subnet_prefix fe80::  lid 25  state ACTIVE  physical_state LINK_UP
...
```

**`ibstat`과의 차이**:
- `rdma link`는 RDMA **subsystem(netlink 기반, netns 인식)** 관점. Namespace isolation을 체크할 때 이걸 씀.
- Ethernet link layer로 설정된 `mlx5_2`, `mlx5_8`에는 `netdev enp...`이 붙음 → RDMA-capable Ethernet(RoCE) 가능성 열려있음.
- `subnet_prefix fe80::`은 IB의 link-local GID prefix. IPv6 link-local과 형식 공유.

#### (E) `ip -d link show ibp64s0` — 리눅스 커널 네트워크 관점

```
9: ibp64s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 2044
   link/infiniband 00:00:10:49:fe:80:...
   ipoib pkey 0xffff mode datagram
   parentbus pci parentdev 0000:40:00.0
```

**해석**:
- `mtu 2044` = IPoIB 기본 MTU (datagram 모드). Connected 모드면 65520까지 가능. NCCL은 IPoIB를 거의 안 쓰고 verbs 직접 호출이라 MTU는 IB 자체 MTU(4K)를 씀.
- `ipoib pkey 0xffff` = default partition (full membership). UFM 없으니 모두 같은 파티션.
- `parentdev 0000:40:00.0` = **이 netdev의 부모 PCI device**. 회사 NAD가 `device: ibp64s0`로 지목하면 Host-Device CNI는 결국 이 PCI 주소의 인터페이스를 Pod netns로 옮김.
- `link/infiniband 00:00:10:49:fe:80:...` = 20바이트 IB hardware address (LID + QPN + GID). Ethernet의 6바이트 MAC과 구조가 완전히 다름.

#### (F) 표로 정리 — dgx03 HCA 용도별 분류

| mlx5 | netdev | PCI bus | 상태 | Rate | 용도 (추정) |
|---|---|---|---|---|---|
| mlx5_0 | ibp24s0 | 0x18 | Active | 400 | Compute GPU0 |
| mlx5_1 | ibp41s0f0 | 0x29.0 | Down | - | Storage IB (미사용) |
| mlx5_2 | enp41s0f1np1 | 0x29.1 | Down | - | Storage Eth (미사용) |
| mlx5_3 | ibp64s0 | 0x40 | Active | 400 | Compute GPU1 |
| mlx5_4 | ibp79s0 | 0x4f | Active | 400 | Compute GPU2 |
| mlx5_5 | ibp94s0 | 0x5e | Active | 400 | Compute GPU3 |
| mlx5_6 | ibp154s0 | 0x9a | Active | 400 | Compute GPU4 |
| mlx5_7 | ibp170s0f0 | 0xaa.0 | Down | - | Storage IB (미사용) |
| mlx5_8 | enp170s0f1np1 | 0xaa.1 | Down | - | Storage Eth (미사용) |
| mlx5_9 | ibp192s0 | 0xc0 | Active | 400 | Compute GPU5 |
| mlx5_10 | ibp206s0 | 0xce | Active | 400 | Compute GPU6 |
| mlx5_11 | ibp220s0 | 0xdc | Active | 400 | Compute GPU7 |
| irdma0 | ens6f0 | - | Active | 100 | Onboard mgmt (Intel) |
| irdma1 | ens6f1 | - | Down | - | Onboard mgmt (Intel) |

**Compute fabric = 8 포트 Active × 400Gbps = 노드당 3.2 Tbps IB 대역폭**. 이 대역이 NCCL AllReduce에 쓰이는 실효 용량.

#### (G) 이걸 K8s NAD/PyTorchJob과 어떻게 연결?

회사 values.yaml + 위 Active 인터페이스 기준이면:
- NAD는 **Active 8개 중 하나**를 `device: ibp64s0` 같은 식으로 지목해야 함 (Down 포트 지목하면 Pod ADD 실패).
- 노드별로 PCI bus 번호가 완전히 같지는 않을 수 있음 — DGX 슬롯 배치는 동일해서 bus 번호도 대부분 일치하지만, **bus 번호가 다른 노드가 섞이면 NAD 하나로 전체 노드 커버가 안 됨** → 운영 시 확인 포인트.
- NCCL은 `NCCL_IB_HCA=mlx5_0,mlx5_3,mlx5_4,mlx5_5,mlx5_6,mlx5_9,mlx5_10,mlx5_11`처럼 **Active HCA만** 나열해야 Down HCA 초기화 시도로 인한 hang 방지.

#### 한 줄 정리

**"dgx03에는 ConnectX-7이 12장 있지만 Compute용 8장만 Active(400Gbps)이고, 나머지 4장은 Storage용인데 이 사이트에선 미결선 상태다. `ibp64s0` 같은 이름의 `64`는 PCI bus 0x40(=64)을 의미하며, 이 PCI 주소가 바로 Host-Device CNI가 Pod netns로 옮기는 대상이다."**

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

### 4.4 Annotation 한 줄의 정체 (★ 면접 단골)

위의 `k8s.v1.cni.cncf.io/networks` 가 **어디서 왔고**, **kubelet부터 CNI까지 어떻게 흘러가는지** 하나하나 뜯어보자.

#### (A) Annotation이란?

Kubernetes 객체(Pod, Service 등)에 **임의의 key=value 메타데이터**를 붙이는 필드. Label과 달리 **스케줄링/필터링에 안 쓰이고**, 특정 컴포넌트(여기선 Multus)가 읽어 해석하는 **"확장 지시서"** 역할.

```yaml
metadata:
  labels:       # ← 스케줄러/셀렉터가 봄 (쿼리 대상)
    app: slm
  annotations:  # ← 특정 tool이 봄 (자유 메타데이터)
    k8s.v1.cni.cncf.io/networks: ib-ibp64s0-conf-slm
```

- **Key 형식**: `<도메인>/<이름>`. 네임스페이스 충돌 방지용 DNS-like prefix.
- `k8s.v1.cni.cncf.io` = **CNCF의 Network Plumbing Working Group**이 정한 공식 네임스페이스. Multus가 이 prefix만 믿고 파싱.
- `networks` = 이 annotation key가 "붙일 추가 네트워크 목록"이라는 뜻.

#### (B) Value 포맷 — 세 가지 형식 지원

Multus는 value를 **세 가지 방식**으로 해석한다:

**1) 문자열 (간단형)** — 회사가 쓰는 방식
```yaml
k8s.v1.cni.cncf.io/networks: ib-ibp64s0-conf-slm
```
= "현재 Pod의 namespace에서 이름이 `ib-ibp64s0-conf-slm`인 NAD를 찾아 붙여라"

**2) 콤마 구분 (여러 개)**
```yaml
k8s.v1.cni.cncf.io/networks: ib-net-a, ib-net-b
```
= NAD 두 개를 순서대로 붙여 `net1`, `net2` 생성.

**3) JSON (세밀 제어)**
```yaml
k8s.v1.cni.cncf.io/networks: |
  [
    {"name": "ib-ibp64s0-conf-slm", "namespace": "network", "interface": "ib0"},
    {"name": "storage-net", "ips": ["10.0.0.5/24"]}
  ]
```
- `namespace`: **다른 namespace의 NAD**를 참조 (공용 NAD를 한 ns에 두고 팀별 Pod이 공유할 때)
- `interface`: Pod 안에서 보일 이름 지정 (기본은 `net1`, `net2`...)
- `ips`: whereabouts 없이 정적 IP 지정
- `mac`: 정적 MAC

회사는 NAD 이름에 PF(`ibp64s0`)와 팀명(`slm`)을 박아서 JSON을 안 써도 "뭐가 붙는지" 명확하게 만든 전략.

#### (C) 이름 컨벤션 분해 — `ib-ibp64s0-conf-slm`

| 조각 | 의미 |
|---|---|
| `ib-` | InfiniBand 네트워크라는 prefix |
| `ibp64s0` | 이 NAD가 지목하는 **호스트 PF** (PCI bus 0x40) |
| `conf` | NetworkAttachmentDefinition의 줄임 (사이트 컨벤션) |
| `slm` | **사용 팀/네임스페이스** (Small Language Model 팀) |

→ "slm 팀 전용으로 `ibp64s0` PF를 Host-Device로 붙이는 NAD". Annotation만 봐도 **어느 팀의 Pod이 어느 물리 IB 포트를 독점할지** 즉시 판별 가능.

#### (D) Annotation이 CNI 호출로 변환되는 전체 흐름

```
[1] 사용자가 Pod/PyTorchJob 제출
       ↓
[2] kube-apiserver 저장 → kube-scheduler가 노드 선정
       ↓
[3] 노드의 kubelet이 Pod spec 수신
       ↓
[4] kubelet → containerd(CRI) → pause 컨테이너 생성 → netns 생성
       ↓
[5] kubelet이 /etc/cni/net.d/00-multus.conf 를 읽고 CNI ADD 호출
    (primary CNI = Multus)
       ↓
[6] Multus가 Pod spec의 annotations 파싱
       ↓
[7] ┌─ delegates (Calico 등) 먼저 호출 → eth0 생성
    └─ k8s.v1.cni.cncf.io/networks 값 읽음 = "ib-ibp64s0-conf-slm"
       ↓
[8] Multus가 API Server에 쿼리:
    GET /apis/k8s.cni.cncf.io/v1/namespaces/<pod-ns>/network-attachment-definitions/ib-ibp64s0-conf-slm
       ↓
[9] NAD의 spec.config JSON 추출:
    {"type":"host-device", "device":"ibp64s0", "ipam":{"type":"whereabouts",...}}
       ↓
[10] Multus가 host-device CNI 바이너리를 **stdin에 위 JSON 넣고** exec
     (ENV: CNI_COMMAND=ADD, CNI_NETNS=/var/run/netns/xxx, CNI_IFNAME=net1)
       ↓
[11] host-device CNI:
     - netlink로 ibp64s0를 Pod netns로 setns
     - 이름을 net1로 rename
     - whereabouts에 IPAM 위임 → IP 할당
     - 결과 JSON을 stdout으로 반환
       ↓
[12] Multus가 결과들을 합쳐 kubelet에 반환
       ↓
[13] kubelet이 컨테이너 기동 → Pod 안에서 net1 사용 가능
```

**핵심은 [6]~[8]**: annotation 값(문자열)이 NAD의 이름으로 쓰여 API Server 조회 키가 되고, NAD의 `spec.config`가 실제 CNI plugin에게 전달되는 **설정 본체**. Annotation은 "포인터", NAD는 "실제 데이터".

#### (E) kubelet은 annotation을 모른다 — Multus만 본다

**자주 오해하는 부분**. kubelet은 annotation을 파싱하지 않음. kubelet이 하는 일은:
1. CNI config (`00-multus.conf`)를 읽고 **등록된 primary CNI**만 호출
2. Pod의 전체 spec을 **환경변수 `CNI_ARGS`**와 **stdin**으로 CNI에 넘김

Annotation을 "해석"하는 주체는 **Multus binary**다. 그래서 Multus를 떼면 이 annotation은 그냥 주석 취급되어 무시됨.

#### (F) Pod 입장에서 확인 방법

```bash
# Pod이 붙은 네트워크 목록 (Multus가 결과를 annotation에 역기록)
kubectl get pod <name> -o jsonpath='{.metadata.annotations}' | jq
```
→ Multus는 성공 후 `k8s.v1.cni.cncf.io/network-status` annotation에 **실제 할당 결과**(IP, MAC, device-info)를 역으로 기록한다. 디버깅 시 이 값을 보면 "어떤 NAD가 실제로 붙었고 IP가 뭐 받았는지" 즉시 확인 가능.

```bash
kubectl exec -it <pod> -- ip -d link show net1
kubectl exec -it <pod> -- ibv_devinfo
```

#### (G) PyTorchJob에서의 위치

PyTorchJob은 내부적으로 Pod을 생성하므로 annotation은 **PodTemplateSpec**에 들어간다:

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
spec:
  pytorchReplicaSpecs:
    Worker:
      template:
        metadata:
          annotations:
            k8s.v1.cni.cncf.io/networks: ib-ibp64s0-conf-slm   # ← 여기
            sidecar.istio.io/inject: "false"
        spec:
          containers: [...]
```

회사에서 **Worker2/Worker3 트릭**을 쓸 때, 각 Replica마다 **다른 NAD 이름**을 annotation에 박아넣는 것이 핵심. Worker는 `ibp64s0`, Worker2는 `ibp79s0`... 이런 식으로 **노드별 PF를 분산 배정**하는 원리.

#### 한 줄 요약

**"`k8s.v1.cni.cncf.io/networks: <NAD이름>` annotation은 kubelet이 아니라 Multus가 읽는 지시서이며, 해당 NAD의 CNI 설정을 fetch해서 delegate CNI(host-device)에게 넘겨 Pod netns에 추가 인터페이스를 붙이게 만드는 트리거다."**

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
