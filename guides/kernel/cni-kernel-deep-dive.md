# CNI를 커널 레벨에서 이해하기 - 진짜 무슨 일이 일어나는가

> **목적**: "Pod에 네트워크가 붙는다"는 추상적인 말이 커널에서 실제로 어떤 시스템 콜과 자료구조로 구현되는지 끝까지 따라가본다. 리눅스 네트워킹 기초가 없는 사람도 따라올 수 있게 단계별로 설명한다.

---

## 0. 시작 전에: 이 문서를 읽고 나면

다음 질문들에 막힘 없이 답할 수 있게 됩니다.

- "Pod이 뭔가요?" → 단순히 "컨테이너 묶음"이 아닌, **커널 자료구조 관점**의 답
- "Pod 두 개가 어떻게 통신하나요?" → 패킷이 거치는 모든 커널 단계
- "왜 RDMA가 빠른가요?" → "커널 우회"가 정확히 어떤 동작인지
- "Host-Device CNI는 뭐가 다른가요?" → 일반 CNI 대비 어떤 단계를 건너뛰는지

---

## 1. 빌딩 블록 ①: Network Namespace

### 1.1 namespace란 무엇인가

리눅스 커널은 시스템 자원을 **격리된 묶음**으로 나누는 기능을 제공합니다. 이걸 namespace라고 부릅니다.

| Namespace 종류 | 격리하는 것 |
|---------------|-----------|
| `mnt` | 파일시스템 마운트 |
| `pid` | 프로세스 ID 공간 |
| `net` | **네트워크 스택** ← 우리가 볼 것 |
| `uts` | 호스트네임 |
| `ipc` | 프로세스 간 통신 |
| `user` | UID/GID |
| `cgroup` | cgroup 루트 |

### 1.2 컨테이너 = namespace의 조합

도커 컨테이너든 K8s Pod이든, 본질은 동일합니다.

> **컨테이너 = 위 namespace들을 조합해서 격리된 공간 + cgroup으로 자원 제한 + chroot 비슷한 파일시스템 격리**

마법이 아니라, **리눅스 커널이 원래 가진 기능들을 묶어 쓴 것**일 뿐입니다.

### 1.3 Network Namespace는 뭘 격리하나

`net` namespace를 새로 만들면, 그 안에는 다음이 **완전히 새로** 생깁니다.

```
새 net namespace에 포함되는 것들:
├── 네트워크 인터페이스 (lo만 있고 다운 상태)
├── IP 주소 / 라우팅 테이블
├── ARP 테이블
├── iptables / netfilter 규칙
├── 소켓 (socket) 자료구조
├── /proc/net/* 통계
└── 포트 번호 공간 (다른 namespace의 80포트와 충돌 안 함)
```

### 1.4 직접 만져보기

이건 K8s 없이 리눅스 명령만으로 가능합니다.

```bash
# 1. 새 namespace 생성
sudo ip netns add my-pod

# 2. 목록 확인
sudo ip netns list
# my-pod

# 3. namespace 안에서 명령 실행
sudo ip netns exec my-pod ip a
# 1: lo: <LOOPBACK> mtu 65536 ...
#    (lo만 있고 그것도 DOWN 상태)

# 4. namespace 안의 lo 켜기
sudo ip netns exec my-pod ip link set lo up

# 5. namespace 안에서 ping
sudo ip netns exec my-pod ping 127.0.0.1
# 동작함 (자기 자신만)

# 6. namespace 안에서 호스트 인터페이스 보려고 하면?
sudo ip netns exec my-pod ip link show eth0
# Device "eth0" does not exist
# → 호스트의 eth0이 안 보임. 완전히 격리됨.
```

### 1.5 핵심 통찰

> **K8s Pod = 그냥 하나의 net namespace + 그 안에서 도는 컨테이너들**

Pod 안의 모든 컨테이너는 같은 net namespace를 공유합니다. 그래서 같은 Pod 컨테이너끼리는 `localhost`로 통신 가능. 이 사실 하나로 K8s 네트워킹의 절반은 이해된 셈입니다.

---

## 2. 빌딩 블록 ②: veth pair (가상 이더넷 케이블)

### 2.1 문제: 격리된 namespace를 어떻게 외부와 연결?

namespace 안은 완전히 분리되어 있습니다. 그런데 통신은 해야 합니다. 어떻게?

→ **가상 케이블**을 만들어서 한쪽은 호스트, 다른 한쪽은 Pod namespace에 꽂으면 됩니다. 이게 **veth pair**.

### 2.2 veth pair의 동작

```
veth pair = 양쪽 끝이 연결된 가상 네트워크 케이블

  veth-host  ←─────커널 메모리─────→  veth-pod
   (호스트)                            (Pod)

   한쪽에 패킷 넣으면 ──→ 반대쪽으로 즉시 나옴
```

물리 케이블처럼 보이지만, 실제로는 **커널이 메모리에서 패킷을 복사**하는 것뿐입니다.

### 2.3 직접 만들어보기

```bash
# 1. veth pair 생성 (양쪽 이름: veth-host, veth-pod)
sudo ip link add veth-host type veth peer name veth-pod

# 2. 호스트에서 확인
ip link show
# veth-host@veth-pod: ...
# veth-pod@veth-host: ...
#   → 둘 다 호스트에 있음 (아직 namespace 안 옮김)

# 3. 한쪽을 Pod namespace로 이동
sudo ip link set veth-pod netns my-pod

# 4. 호스트에서 다시 확인
ip link show veth-pod
# Device "veth-pod" does not exist
#   → 호스트에서 사라짐. Pod namespace로 옮겨짐.

# 5. Pod namespace 안에서 확인
sudo ip netns exec my-pod ip link show
# veth-pod@if??: ...
#   → 여기 있다!

# 6. 양쪽에 IP 부여
sudo ip addr add 10.0.0.1/24 dev veth-host
sudo ip link set veth-host up

sudo ip netns exec my-pod ip addr add 10.0.0.2/24 dev veth-pod
sudo ip netns exec my-pod ip link set veth-pod up

# 7. 통신 테스트
ping 10.0.0.2
# 통신 됨!
```

축하합니다. 방금 손으로 만든 게 **CNI 플러그인이 자동으로 하는 일의 절반**입니다.

---

## 3. 빌딩 블록 ③: bridge (가상 스위치)

### 3.1 문제: Pod이 여러 개면 어떻게?

veth pair는 1:1 연결입니다. Pod이 100개면 호스트에 veth가 100개 매달려 있어야 하고, 서로 통신하려면 라우팅이 복잡해집니다.

**해결**: 호스트에 **가상 스위치(bridge)**를 만들고 모든 veth를 거기 꽂는다.

```
            [Linux Bridge: cni0]  ← 가상 스위치 (L2)
             │     │     │     │
        veth-h1  veth-h2  veth-h3  veth-h4
             │     │     │     │
        ┌─[Pod1]─[Pod2]─[Pod3]─[Pod4]─┐
        └ 각각 veth-pod 인터페이스로 ┘
```

이건 물리 스위치와 똑같이 동작합니다. MAC 주소 기반으로 패킷을 적절한 포트로 보내줍니다.

### 3.2 직접 만들어보기

```bash
# bridge 생성
sudo ip link add cni0 type bridge
sudo ip link set cni0 up

# veth-host를 bridge에 연결
sudo ip link set veth-host master cni0
```

이제 같은 bridge에 연결된 모든 veth(즉, 모든 Pod)가 서로 통신 가능합니다.

---

## 4. CNI 플러그인이 실제로 하는 일

이제 CNI 플러그인이 뭘 하는지 정확히 보입니다.

### 4.1 Pod 1개 생성 시 CNI가 실행하는 명령들

K8s가 Pod을 생성할 때 컨테이너 런타임(containerd)이 CNI 플러그인을 호출합니다. CNI 플러그인은 사실상 다음을 자동화합니다.

```bash
# === CNI 플러그인이 실행하는 일 (의사코드) ===

POD_ID="abc123"
POD_IP="10.244.1.5"

# 1. 컨테이너 런타임이 미리 만든 namespace 경로 받기
NS_PATH="/var/run/netns/$POD_ID"

# 2. veth pair 생성
ip link add veth-$POD_ID type veth peer name eth0

# 3. veth의 한쪽을 Pod namespace로 이동
ip link set eth0 netns $NS_PATH

# 4. Pod namespace 안에서 IP 설정
ip -n $POD_ID addr add $POD_IP/24 dev eth0
ip -n $POD_ID link set eth0 up
ip -n $POD_ID link set lo up

# 5. Pod에서 외부로 나갈 default route
ip -n $POD_ID route add default via 10.244.1.1

# 6. 호스트쪽 veth를 bridge에 연결 (또는 라우팅으로 처리)
ip link set veth-$POD_ID master cni0
ip link set veth-$POD_ID up

# 7. iptables NAT/MASQUERADE 규칙 추가 (외부 통신용)
iptables -t nat -A POSTROUTING -s 10.244.0.0/16 -j MASQUERADE
```

**그게 전부입니다.** CNI 플러그인 = 위 명령들을 자동화한 프로그램.

### 4.2 결과: 완성된 그림

```
[호스트 default namespace]
    │
    ├── eth0 (실제 NIC, 192.168.1.10)
    ├── cni0 (bridge, 10.244.1.1)
    │     ├── veth-pod1 ──── eth0(10.244.1.5) [Pod1 namespace]
    │     ├── veth-pod2 ──── eth0(10.244.1.6) [Pod2 namespace]
    │     └── veth-pod3 ──── eth0(10.244.1.7) [Pod3 namespace]
    │
    ├── 라우팅 테이블
    └── iptables 규칙 (NAT, FILTER)
```

---

## 5. 패킷이 흘러가는 경로 (가장 중요)

이제 Pod1(10.244.1.5)에서 Pod2(10.244.1.6)으로 패킷을 보낼 때 **커널에서 정확히 무슨 일**이 일어나는지 따라가봅시다.

### 5.1 같은 노드의 Pod 간 통신

```
[Pod1의 nginx 프로세스]
    │ socket(), connect(10.244.1.6:80), send("GET /")
    ↓
[Pod1 namespace의 커널]
    │ 1. socket buffer (sk_buff) 생성
    │ 2. TCP 헤더 추가 (소스/목적 포트)
    │ 3. IP 헤더 추가 (10.244.1.5 → 10.244.1.6)
    │ 4. 라우팅 테이블 lookup → eth0(veth-pod)으로 송신
    │ 5. ARP로 다음 hop MAC 알아냄
    │ 6. 이더넷 헤더 추가
    ↓
[veth pair 통과]
    │ veth_xmit() 함수가 sk_buff를 반대쪽 큐로 enqueue
    │ (커널 메모리 안에서의 복사 — 실제 NIC 안 거침)
    ↓
[호스트 default namespace의 커널]
    │ 7. veth-pod1에서 패킷 수신
    │ 8. bridge(cni0)에 도달
    │ 9. bridge가 destination MAC 보고 veth-pod2로 forward
    │ 10. iptables FORWARD 체인 통과 (Network Policy 검사)
    ↓
[veth pair 통과 (반대 방향)]
    ↓
[Pod2 namespace의 커널]
    │ 11. eth0(veth-pod2)에서 수신
    │ 12. IP/TCP 스택 거쳐 socket buffer로
    ↓
[Pod2의 nginx 프로세스]
    │ recv() 호출이 리턴
```

**12단계**입니다. 단순해 보이는 Pod 간 통신이 커널에서 이렇게 많은 일을 합니다.

### 5.2 다른 노드의 Pod로 통신

```
[Pod1] → [노드A 호스트 커널] → [실제 eth0] → [네트워크]
                                                  ↓
[Pod2] ← [노드B 호스트 커널] ← [실제 eth0] ← ────┘
```

추가로:
- 노드A 커널에서 라우팅 테이블 lookup → Pod IP를 어느 노드로 보낼지 결정 (Calico가 BGP로 광고한 정보)
- 실제 NIC을 통해 물리 네트워크로 송신
- 노드B 도착 후 다시 bridge → veth → Pod namespace 경로

### 5.3 비용 분석

이 경로의 비용은 다음과 같습니다.

| 항목 | 비용 |
|------|------|
| sk_buff 할당/해제 | 매 패킷마다 |
| TCP/IP 스택 처리 | 헤더 파싱, 체크섬 |
| veth 통과 | 메모리 복사 |
| bridge forwarding | MAC 테이블 lookup |
| iptables 규칙 매칭 | 규칙 수에 비례 |
| context switch | 유저공간 ↔ 커널공간 전환 |
| 인터럽트 처리 | NIC IRQ |

수 μs 단위의 지연이 누적됩니다. 대부분의 워크로드에는 문제 없지만, **AI 분산 학습처럼 매 step마다 GB 단위로 통신하는 경우 치명적인 병목**이 됩니다.

---

## 6. Host-Device CNI: 위 과정을 통째로 우회

### 6.1 발상의 전환

> "veth pair로 가상 케이블 만들지 말고, **물리 NIC 자체를 Pod namespace로 옮겨버리면** 어떨까?"

이게 Host-Device CNI입니다.

### 6.2 동작

```
일반 CNI (Calico):
[Pod] - veth - bridge - 호스트커널 - 실제 NIC

Host-Device CNI:
[Pod] ─ 물리 NIC 직접 소유
        (호스트에서는 안 보임)
```

### 6.3 실제로 하는 일

CNI 플러그인의 코드는 사실상 한 줄입니다.

```bash
# Host-Device CNI가 하는 일의 본질
ip link set ibp64s0 netns $POD_NAMESPACE
```

진짜 이게 끝입니다. 물리 인터페이스를 namespace로 이동시키는 것.

### 6.4 결과

```bash
# Pod 시작 전 호스트
$ ip link show
1: lo: ...
2: eth0: ...
3: ibp64s0: ...      ← 여기 있음

# Pod 시작 후 호스트
$ ip link show
1: lo: ...
2: eth0: ...
                     ← ibp64s0 사라짐!

# Pod 안에서 보면
$ kubectl exec mypod -- ip link show
1: lo: ...
2: eth0: ...           ← Calico가 만든 거 (제어 트래픽용)
3: net1@ibp64s0: ...   ← 물리 IB 카드가 통째로!
```

### 6.5 장단점

**장점**:
- veth, bridge, iptables를 거치지 않음 → 오버헤드 거의 0
- 물리 NIC을 직접 제어 가능 → RDMA 같은 고급 기능 사용 가능

**단점**:
- 물리 NIC은 한 번에 하나의 namespace에만 속할 수 있음
- → **한 노드의 한 IB 포트 = 한 Pod 전용** (공유 불가)
- → 우리 회사의 "1 Node = 1 Pod = 1 NAD" 제약의 근본 원인

---

## 7. RDMA: 한 단계 더 깊은 우회

### 7.1 지금까지 본 모든 통신의 공통점

Host-Device CNI를 써도 통신은 여전히 **커널의 TCP/IP 스택**을 거쳤습니다.

```
앱 → socket() → 커널 TCP/IP 스택 → NIC 드라이버 → NIC
```

이 경로 자체에 비용이 있습니다.

### 7.2 RDMA의 발상

> "커널 자체를 건너뛰자. 앱이 NIC 하드웨어에 **직접** 명령을 내리게 하자."

### 7.3 RDMA 통신 경로

```
일반 통신:
[App 메모리]
   ↓ socket write()  ← system call (커널 진입)
[커널 socket buffer]
   ↓ TCP/IP stack
[커널 NIC 드라이버]
   ↓ DMA
[NIC 하드웨어]
   ↓
[네트워크]

RDMA 통신:
[App 메모리]
   ↓ libibverbs (유저공간 라이브러리)
   ↓ NIC의 메모리 영역(QP)에 직접 명령 write
[NIC 하드웨어]   ← 커널 안 거침!
   ↓ NIC이 알아서 App 메모리에서 DMA로 데이터 읽음
   ↓
[네트워크]
```

### 7.4 핵심 메커니즘: Queue Pair (QP)

RDMA NIC은 메모리 안에 **명령 큐**를 가지고 있습니다.

```
[NIC 하드웨어]
  ├── Send Queue (SQ)    ← 보낼 명령 들어오는 큐
  ├── Receive Queue (RQ) ← 받을 준비 큐
  └── Completion Queue   ← 완료 통지 큐
```

앱은 이 큐에 "이 메모리 주소에서 시작해서 X바이트를 저쪽 노드로 보내라"라고 적습니다. NIC이 그걸 보고 알아서:

1. 앱 메모리에서 직접 DMA로 데이터 읽음
2. 패킷화해서 네트워크로 송신
3. 완료되면 Completion Queue에 표시

**커널 개입 없음. CPU 거의 안 씀. 메모리 복사 없음.**

### 7.5 왜 빠른가 — 정량적으로

| 항목 | 일반 TCP/IP | RDMA |
|------|------------|------|
| System call | 매 send()마다 | 초기화 시 1회 |
| 메모리 복사 | App → 커널버퍼 → NIC | 0회 (zero-copy) |
| CPU 사용 | 높음 (TCP/IP 처리) | 거의 없음 |
| 지연 | 수 μs ~ 수십 μs | ~1μs |
| Context switch | 잦음 | 거의 없음 |

### 7.6 그래서 필요한 것: RDMA Device Plugin

RDMA를 쓰려면 앱이 NIC의 메모리 영역에 접근해야 하는데, 이건 **디바이스 파일**을 통해 이루어집니다.

```bash
# 호스트의 RDMA 디바이스 파일 (커널이 노출)
$ ls /dev/infiniband/
issm0       # Subnet Manager 인터페이스
rdma_cm     # Connection Manager
ucm0
umad0
uverbs0     # ★ User-space Verbs 인터페이스 (가장 중요)
uverbs1
...
```

Pod 안에서 `libibverbs`가 `/dev/infiniband/uverbs0`을 open()해서 NIC을 직접 제어합니다. 따라서 **Pod에 이 디바이스 파일이 마운트 되어야** RDMA가 동작합니다.

이 일을 **Mellanox RDMA Shared Device Plugin**이 해줍니다.

### 7.7 정리: 두 컴포넌트의 역할 분리

| 컴포넌트 | 하는 일 |
|---------|---------|
| Host-Device CNI | 물리 IB 인터페이스(L2/L3 네트워크 인터페이스)를 Pod namespace로 이동 |
| RDMA Device Plugin | RDMA 디바이스 파일(`/dev/infiniband/*`)을 Pod에 마운트 |

**둘 다 있어야** Pod이 RDMA 통신을 할 수 있습니다.

```yaml
# Pod 정의에서 둘 다 명시
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: ib-ibp64s0-conf-slm   # Host-Device CNI 호출
spec:
  containers:
  - resources:
      limits:
        rdma/hca_shared_devices_a: 1                    # RDMA Device Plugin 호출
```

---

## 8. Multus: 여러 CNI를 조합하기

### 8.1 문제

Pod에 인터페이스가 2개 필요한 상황입니다.
- `eth0`: K8s API, 일반 트래픽 (Calico)
- `net1`: IB RDMA (Host-Device)

그런데 K8s는 기본적으로 Pod에 CNI를 1개만 호출합니다.

### 8.2 Multus의 해법

Multus는 **메타 CNI**입니다. K8s가 Multus를 호출하면, Multus가 다시 여러 CNI를 순서대로 호출합니다.

```
[K8s/containerd]
    ↓ CNI 호출
[Multus] (메타 CNI)
    ├─→ Calico 호출       → eth0 만들어짐
    └─→ Host-Device 호출  → net1 만들어짐
```

### 8.3 NAD (NetworkAttachmentDefinition)

Multus가 "추가로 어떤 CNI를 어떻게 호출할지" 알려주는 K8s 리소스가 NAD입니다.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ib-ibp64s0-conf-slm
spec:
  config: |
    {
      "type": "host-device",
      "device": "ibp64s0"
    }
```

해석: "이 이름의 NAD를 호출하면, host-device CNI를 ibp64s0 인터페이스에 대해 실행해라."

### 8.4 Pod에서 사용

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: ib-ibp64s0-conf-slm
```

이 annotation 한 줄이 "Multus야, ib-ibp64s0-conf-slm NAD에 정의된 추가 네트워크를 net1로 붙여줘"라는 뜻입니다.

---

## 9. 전체 그림 — Pod 생성부터 RDMA 통신까지

이제 모든 조각을 합쳐봅시다. PyTorchJob이 학습 Pod을 만들 때 일어나는 일.

```
[1단계] PyTorchJob CRD 생성
    ↓
[2단계] Kubeflow Training Operator가 Pod 정의 생성
    - annotations: k8s.v1.cni.cncf.io/networks: ib-ibp64s0-conf-slm
    - resources: nvidia.com/gpu: 8, rdma/hca_shared_devices_a: 1
    ↓
[3단계] kube-scheduler가 노드 선택
    ↓
[4단계] kubelet이 노드에서 Pod 생성 시작
    ↓
[5단계] kubelet이 containerd에게 컨테이너 만들라고 요청
    ↓
[6단계] containerd가 Pod의 net namespace 생성 (ip netns add 같은 동작)
    ↓
[7단계] containerd가 Multus CNI 호출
    ├─→ Multus가 Calico 호출
    │     - veth pair 생성
    │     - 한쪽을 Pod namespace로 이동 → eth0 됨
    │     - IP 할당 (10.244.x.y)
    │     - 호스트쪽을 cni0 bridge에 연결
    │
    └─→ Multus가 Host-Device CNI 호출 (NAD ib-ibp64s0-conf-slm 보고)
          - 호스트의 ibp64s0 인터페이스를 Pod namespace로 이동
          - Pod 안에서 net1로 이름 변경
          - 호스트에서는 ibp64s0 사라짐
    ↓
[8단계] kubelet이 NVIDIA GPU Plugin 호출
    - /dev/nvidia0~7 디바이스 파일을 Pod에 마운트
    ↓
[9단계] kubelet이 RDMA Device Plugin 호출
    - /dev/infiniband/uverbs0 등을 Pod에 마운트
    ↓
[10단계] 컨테이너 시작
    - PyTorch 프로세스 실행
    ↓
[11단계] PyTorch가 NCCL 초기화
    - libibverbs를 통해 /dev/infiniband/uverbs0 open()
    - NIC의 Queue Pair 생성
    - 다른 노드의 Pod와 QP 연결 수립
    ↓
[12단계] 학습 루프
    - GPU에서 backward pass → gradient 생성
    - NCCL이 AllReduce 호출
    - libibverbs가 QP에 명령 enqueue
    - IB NIC이 GPU 메모리에서 직접 RDMA 읽기 (GPUDirect RDMA)
    - IB Switch 거쳐 다른 노드로 전송
    - 다른 노드 NIC이 그쪽 GPU 메모리에 직접 쓰기
    - 커널 한 번 안 거침
    ↓
[학습 완료]
```

---

## 10. 한 장으로 정리

```
                   네트워크 추상화 레벨

[유저 앱]          PyTorch / NCCL
                       ↓
[유저 라이브러리]  libibverbs            ← RDMA는 여기서 끝나고 NIC 직접 제어
                       ↓
[디바이스 파일]    /dev/infiniband/*     ← RDMA Device Plugin이 마운트
─────────────────커널 경계 (RDMA 우회)─────────────────
[커널 net stack]   socket → TCP → IP    ← 일반 통신은 여기 거침
                       ↓
[커널 디바이스]    veth / bridge / NIC   ← Calico가 만들고 사용
─────────────────namespace 경계─────────────────────────
[net namespace]    Pod별 격리             ← CNI가 관리
─────────────────────────────────────────────────────
[하드웨어]         물리 NIC, IB Switch   ← Host-Device로 직접 제어 가능
```

---

## 11. 면접 답변 템플릿

### Q. "K8s 네트워킹을 커널 레벨에서 설명해보세요"

> "Pod의 본질은 리눅스 커널의 net namespace 하나입니다. CNI 플러그인이 호출되면 veth pair를 만들어 한쪽은 호스트 bridge에 꽂고 다른 한쪽을 Pod namespace로 옮긴 뒤, 그 안에서 IP와 라우팅을 설정합니다. Pod 간 통신은 결국 veth, bridge, iptables를 거치는 호스트 커널의 TCP/IP 스택을 통해 이루어집니다.
>
> 우리 환경처럼 RDMA가 필요한 경우 이 경로의 오버헤드가 문제가 됩니다. 그래서 두 가지 우회를 합니다. 첫째, Host-Device CNI로 물리 IB 인터페이스를 통째로 Pod namespace로 옮겨 veth/bridge를 건너뜁니다. 둘째, RDMA 자체가 libibverbs를 통해 NIC 하드웨어를 직접 제어하므로 커널 TCP/IP 스택을 완전히 우회합니다. 이 두 우회 덕분에 GPU 메모리에서 다른 노드의 GPU 메모리로 거의 1μs 수준 지연으로 데이터가 흐를 수 있고, 이게 멀티노드 분산 학습 성능의 핵심입니다."

이 정도면 네트워크/시스템 깊이 있는 면접관도 만족합니다.

---

## 12. 참고 - 직접 실험해보고 싶다면

이 문서의 1~3장(namespace, veth, bridge)은 Linux가 깔린 어떤 머신에서도 직접 해볼 수 있습니다. K8s 없이도 됩니다.

```bash
# 미니 실험 시나리오
sudo ip netns add pod1
sudo ip netns add pod2
sudo ip link add br0 type bridge
sudo ip link set br0 up

# pod1 ↔ br0
sudo ip link add v1h type veth peer name v1p
sudo ip link set v1h master br0
sudo ip link set v1h up
sudo ip link set v1p netns pod1
sudo ip netns exec pod1 ip addr add 10.0.0.1/24 dev v1p
sudo ip netns exec pod1 ip link set v1p up

# pod2 ↔ br0
sudo ip link add v2h type veth peer name v2p
sudo ip link set v2h master br0
sudo ip link set v2h up
sudo ip link set v2p netns pod2
sudo ip netns exec pod2 ip addr add 10.0.0.2/24 dev v2p
sudo ip netns exec pod2 ip link set v2p up

# 통신
sudo ip netns exec pod1 ping 10.0.0.2
```

직접 만들어본 사람은 절대 안 잊습니다. 면접 전 한 번 해보시길 권장합니다.
