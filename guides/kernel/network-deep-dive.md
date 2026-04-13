# 네트워크 깊이 - BGP, VXLAN, MTU, 그리고 K8s/IB 환경의 실제

> **목적**: K8s 네트워킹과 데이터센터 네트워킹의 핵심 개념. Calico의 BGP, VXLAN 오버레이, MTU 튜닝, IB와 이더넷의 차이까지.

---

## 0. K8s 네트워킹의 4대 요구사항

K8s 네트워크 모델:

1. **모든 Pod이 모든 Pod와 직접 통신 가능** (NAT 없이)
2. **모든 노드가 모든 Pod와 통신 가능**
3. **Pod이 자기 IP를 자기 IP로 인식** (호스트와 다르게 보이지 않음)
4. Service는 가상 IP, 로드밸런싱

이걸 어떻게 구현하느냐는 CNI마다 다름. 여기서 BGP, VXLAN 같은 기술이 등장.

---

## 1. 라우팅 vs 오버레이

### 1.1 두 가지 접근

**라우팅 기반 (Calico BGP)**:
- 각 노드가 자기 Pod 대역을 라우터처럼 광고
- 패킷은 평범한 IP 라우팅으로 흐름
- 빠름, 단순, 디버깅 쉬움
- 단, 네트워크 인프라가 BGP 지원 필요

**오버레이 기반 (Flannel VXLAN, Calico IPIP)**:
- 패킷을 다른 패킷으로 감싸서 (encapsulation) 전송
- 네트워크 인프라 무관
- MTU 손실, 약간의 오버헤드

### 1.2 그림으로

```
라우팅:
  Pod A (10.244.1.5) → 노드 라우팅 테이블
                    → 물리 네트워크
                    → Pod B (10.244.2.6)
  
오버레이:
  Pod A → 노드가 패킷을 UDP/IP로 감쌈
       → 외부 IP는 노드 IP
       → 다른 노드 도착, 풀고 → Pod B
```

---

## 2. BGP (Border Gateway Protocol)

### 2.1 BGP는 인터넷의 라우팅 프로토콜

원래 ISP 간 경로 광고에 쓰임. 인터넷 자체가 BGP 위에 동작.

### 2.2 K8s에서 BGP

**Calico**가 각 노드에서 BGP 데몬(BIRD)을 실행해 자기 Pod CIDR을 광고.

```
[노드 A] "10.244.1.0/24는 나한테 와"  ─광고→  [TOR 스위치]
[노드 B] "10.244.2.0/24는 나한테 와"  ─광고→  [TOR 스위치]
                                                ↓
                                          모든 노드가
                                          서로 알게 됨
```

### 2.3 BGP peer 종류

- **iBGP** (Internal): 같은 AS 안 (보통 클러스터 내부)
- **eBGP** (External): 다른 AS 사이 (TOR 스위치와 piring)

### 2.4 Route Reflector

노드 100개 → 각자 99개와 BGP peer 맺어야 함 → 폭발.

해결: **Route Reflector** (BGP의 hub).

```
모든 노드 → RR과만 peer
RR이 정보 중계
```

### 2.5 Calico의 BGP 모드

```yaml
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  asNumber: 64512
  serviceClusterIPs:
  - cidr: 10.96.0.0/12
```

BGP 안 쓰면 IPIP/VXLAN 오버레이로 fallback.

### 2.6 BGP 확인 명령

```bash
# Calico 노드에서
calicoctl node status
# +--------------+-------------------+-------+----------+
# | PEER ADDRESS | PEER TYPE         | STATE | SINCE    |
# +--------------+-------------------+-------+----------+
# | 10.0.0.2     | node-to-node mesh | up    | 12:34:56 |
```

---

## 3. VXLAN (Virtual eXtensible LAN)

### 3.1 무엇

L2(이더넷) 프레임을 L3(UDP/IP)로 감싸는 터널링.

```
원본 패킷:
  [Pod A의 이더넷 프레임]

VXLAN 캡슐화:
  [원본 프레임]
  + VXLAN 헤더 (8 bytes, VNI 포함)
  + UDP 헤더 (8 bytes)
  + 외부 IP 헤더 (20 bytes)
  + 외부 이더넷 헤더 (14 bytes)
  
총 50 bytes 추가!
```

### 3.2 동작

```
[Pod A] → [노드 A의 vxlan0 인터페이스]
            ↓ 캡슐화
          [노드 A의 eth0] → 물리 네트워크
            ↓
          [노드 B의 eth0]
            ↓ 디캡슐화
          [노드 B의 vxlan0]
            ↓
          [Pod B]
```

물리 네트워크는 평범한 UDP 트래픽으로 봄. Pod IP를 몰라도 됨.

### 3.3 VNI (Virtual Network Identifier)

24비트 = 1600만 개 가상 네트워크 분리 가능. (VLAN은 12비트, 4096개 한계)

### 3.4 장단점

장점:
- 물리 네트워크와 분리 (어떤 인프라에서도 동작)
- L2 over L3 (멀리 떨어진 노드도 같은 L2 broadcast 도메인)

단점:
- 50 bytes 오버헤드 → MTU 손실
- 캡슐화/디캡슐화 CPU 사용
- 디버깅 어려움 (외부에서 보면 UDP만)

---

## 4. MTU (Maximum Transmission Unit)

### 4.1 무엇

한 번에 전송 가능한 패킷 크기.

```
이더넷 기본: 1500 bytes
점보 프레임: 9000 bytes
InfiniBand: 2048~4096 bytes (또는 더 큼)
```

### 4.2 MTU 미스매치 문제

```
Pod이 1500 바이트 패킷 보냄
  ↓
VXLAN 캡슐화로 +50 = 1550 바이트
  ↓
물리 NIC MTU 1500 → 단편화 또는 drop
  ↓
"connection hangs" 또는 "느림"
```

### 4.3 해결책

**옵션 1: Pod MTU 낮추기**
```
물리 MTU 1500
- VXLAN 50
= Pod MTU 1450
```

**옵션 2: 점보 프레임**
```
물리 MTU 9000 (점보 프레임)
- VXLAN 50
= Pod MTU 8950
```

성능 좋아지지만 모든 스위치/NIC가 점보 지원해야.

### 4.4 ML 환경에서의 MTU

- 노드 간 큰 데이터(gradient sync) 자주 → 점보 프레임 적극 추천
- IB는 MTU 4096이 흔함
- 데이터센터 내부는 점보 프레임이 표준화 추세

### 4.5 MTU 확인

```bash
ip link show eth0
# eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500

# Path MTU 발견 (실제 사용 가능 MTU)
tracepath -n 8.8.8.8

# MTU 강제로 패킷 보내기 (DF 비트로 단편화 금지)
ping -M do -s 8972 target  # 8972 + 28 = 9000
```

---

## 5. IB vs 이더넷

### 5.1 비교 표

| 항목 | 이더넷 | InfiniBand |
|------|--------|------------|
| 표준 | IEEE 802.3 | InfiniBand Trade Association |
| 패킷 | Ethernet frame | IB packet (다름) |
| 라우팅 | IP, BGP, OSPF | Subnet Manager가 중앙 관리 |
| 흐름 제어 | TCP가 처리 | 하드웨어 link-level credit |
| 손실 | drop 발생 가능 | lossless (credit-based) |
| L2 | broadcast/MAC | LID (Local ID) |
| L3 | IP | GID (Global ID, 128-bit) |

### 5.2 RoCE (RDMA over Converged Ethernet)

이더넷 위에서 RDMA를 구현. InfiniBand 프로토콜을 이더넷으로 옮긴 것.

```
RoCEv1: 이더넷 위 (같은 L2 도메인 안에서만)
RoCEv2: UDP/IP 위 (라우팅 가능)
```

장점: 기존 이더넷 인프라 활용
단점: lossless 보장이 약함 → DCB(데이터센터 브리징) 설정 필요

NCCL의 `NCCL_IB_GID_INDEX=3`이 RoCE 환경 설정.

### 5.3 IB Subnet Manager (opensm)

InfiniBand는 BGP 같은 게 없음. 대신 **중앙 집중식 SM**이 모든 경로 결정.

```
[opensm]
   ↓ 모든 IB 디바이스 발견
   ↓ 토폴로지 학습
   ↓ 경로 계산 (multipath, ECMP)
   ↓ 각 NIC에 경로 정보 push
[모든 노드가 어디로 보낼지 안 상태]
```

opensm 죽으면 새 연결은 못 만들지만 기존은 유지. 보통 redundant SM 운영.

### 5.4 우리 회사 환경

- **노드 간 학습 트래픽**: IB native (NCCL, RDMA)
- **K8s 제어 트래픽**: 이더넷 (Calico)
- **Ceph 트래픽**: IB native (RDMA)

각각 다른 물리 네트워크를 통해.

---

## 6. Service & 로드밸런싱

### 6.1 ClusterIP

```yaml
spec:
  type: ClusterIP
  selector: { app: api }
  ports:
  - port: 80
```

가상 IP. kube-proxy가 iptables/IPVS 규칙으로 백엔드 Pod에 분배.

### 6.2 NodePort

각 노드의 특정 포트로 노출.
```
http://node-ip:30080 → Service → Pod
```

### 6.3 LoadBalancer

클라우드 제공자(AWS ELB, GCP LB)가 외부 LB 생성. 온프레미스에선:

- **MetalLB**: BGP 또는 ARP로 외부 IP 광고
- **kube-vip**: 비슷한 컨셉

### 6.4 Ingress

L7 라우팅 (HTTP path/host).

```yaml
kind: Ingress
spec:
  rules:
  - host: api.company.com
    http:
      paths:
      - path: /v1
        backend: { service: api-v1 }
      - path: /v2
        backend: { service: api-v2 }
```

뒤에 Ingress Controller (nginx, traefik, contour) 필요.

### 6.5 Service Mesh

L7+ 트래픽 관리, 보안, 관측. 대표: Istio, Linkerd.

```
[Pod] ← sidecar (Envoy) ← Service Mesh
            ↓
        모든 트래픽 가로채서:
        - mTLS 자동 적용
        - retry, timeout
        - canary deployment
        - 분산 트레이싱
```

**주의**: RDMA 트래픽은 Envoy가 못 봄 → IB Pod은 Istio 비활성화 필요 (우리 회사 환경 그대로).

---

## 7. CNI별 비교

| CNI | 라우팅 방식 | NetworkPolicy | 특이점 |
|-----|------------|--------------|--------|
| **Flannel** | VXLAN 오버레이 | 미지원 | 단순, 가벼움 |
| **Calico** | BGP (또는 IPIP/VXLAN) | 강력 | 가장 인기, 우리 회사 사용 |
| **Cilium** | eBPF | L7까지 | 차세대 |
| **Weave** | 메시 + 암호화 | 지원 | 자동 메시 |
| **OVN-Kubernetes** | OpenVSwitch | 강력 | RHEL 계열 표준 |

---

## 8. DNS

### 8.1 CoreDNS

K8s 안의 DNS 서버. 모든 Pod이 자동으로 사용.

### 8.2 Service DNS

```
<service>.<namespace>.svc.cluster.local

예: my-api.default.svc.cluster.local → Service IP
```

같은 namespace는 짧게:
```
my-api → 자동으로 my-api.default.svc.cluster.local로 해석
```

### 8.3 Pod DNS (StatefulSet)

```
<pod>.<service>.<namespace>.svc.cluster.local
```

`pytorch-master-0.training.slm.svc.cluster.local` 같은 형태로 학습 잡에서 사용.

### 8.4 NCCL과 DNS

PyTorch 분산 학습은 `MASTER_ADDR`을 환경 변수로 받는데, K8s에서는 보통 Service DNS 이름.

```python
os.environ["MASTER_ADDR"] = "pytorch-master-0.training-svc"
```

---

## 9. 트러블슈팅 도구

### 9.1 기본

```bash
# 인터페이스
ip a / ip link

# 라우팅
ip route / ip rule

# ARP
ip neigh

# 연결 상태
ss -tunap            # 모든 소켓
netstat -i           # 인터페이스 통계

# 패킷 캡처
tcpdump -i eth0 host 10.0.0.1 -w capture.pcap

# 라우팅 추적
traceroute / mtr target
```

### 9.2 K8s 안에서

```bash
# Pod 안에 임시 네트워크 도구 컨테이너
kubectl debug -it mypod --image=nicolaka/netshoot

# 노드의 네트워크 namespace 디버깅
sudo nsenter -t <pid> -n ip a
```

### 9.3 IB 도구

```bash
ibstatus              # IB 포트 상태
ibv_devinfo           # 디바이스 정보
ibping                # IB ping
ib_write_bw           # 대역폭 측정
perfquery             # 포트 카운터
```

---

## 10. 면접 예상 질문

### Q1. "K8s 네트워크 모델의 4가지 요구사항은?"
> "모든 Pod이 NAT 없이 직접 통신 가능, 노드와 Pod 양방향 통신 가능, Pod이 자기 IP를 자기 IP로 인식, Service가 가상 IP로 로드밸런싱. 이를 어떻게 구현하느냐는 CNI 선택의 문제이며, BGP 라우팅(Calico)이나 VXLAN 오버레이(Flannel) 등이 있습니다."

### Q2. "BGP와 VXLAN 중 뭘 쓰나요?"
> "BGP는 평범한 IP 라우팅으로 흐르므로 빠르고 디버깅 쉽지만 네트워크 인프라가 BGP를 지원해야 합니다. VXLAN은 인프라 무관하게 동작하지만 50바이트 캡슐화 오버헤드가 있고 MTU 손실이 발생합니다. 통제된 데이터센터 환경이면 BGP, 아니면 VXLAN 권장합니다."

### Q3. "MTU 미스매치는 어떻게 진단하나요?"
> "tracepath나 ping -M do -s <size>로 실제 가능한 MTU를 측정합니다. VXLAN 환경이면 노드 MTU에서 50바이트 빼야 Pod MTU가 됩니다. 패킷이 단편화되거나 drop되면 'connection hangs' 또는 '큰 요청만 실패' 같은 증상이 나타납니다. 우리 ML 환경처럼 큰 데이터 통신이 잦으면 점보 프레임(9000) 추천입니다."

### Q4. "InfiniBand의 Subnet Manager는 무엇이고 왜 필요한가요?"
> "이더넷에는 BGP 같은 분산 라우팅 프로토콜이 있지만 IB는 중앙집중식 Subnet Manager(opensm)가 모든 경로를 계산해 각 NIC에 push합니다. SM이 토폴로지를 학습하고 multipath 경로를 결정합니다. SM이 죽어도 기존 연결은 유지되지만 새 연결을 위해 보통 redundant SM을 운영합니다."

### Q5. "왜 Service Mesh(Istio)를 IB Pod에 못 쓰나요?"
> "Istio의 Envoy sidecar는 TCP/IP 트래픽을 가로채는데, RDMA는 커널과 이더넷 스택을 우회해 NIC을 직접 제어합니다. Envoy 입장에서 RDMA 트래픽은 보이지 않고, 그러면서도 net1 인터페이스에 개입 시도하면 통신이 hang 또는 깨집니다. 그래서 IB Pod은 sidecar.istio.io/inject: false로 명시 비활성화합니다."

### Q6. "CoreDNS가 느리거나 응답 안 함?"
> "CoreDNS Pod 상태와 메트릭(요청 수, 실패율) 확인. 흔한 원인: ndots 설정으로 검색 도메인 5개 시도하다 느려짐 (Pod의 /etc/resolv.conf), upstream DNS 응답 느림, CoreDNS Pod 자체 자원 부족. autopath 플러그인 활성화나 NodeLocal DNSCache 도입으로 해결합니다."

---

## 11. 한 줄 요약

> **"K8s 네트워킹 = CNI가 BGP 라우팅(Calico) 또는 VXLAN 오버레이(Flannel)로 Pod-to-Pod 직접 통신을 구현. MTU 튜닝과 점보 프레임이 ML 워크로드 성능에 직결. RDMA는 이더넷/IP 스택을 우회하므로 IB native에선 Subnet Manager가, RoCE에선 DCB가 무손실을 보장. Service Mesh는 RDMA 트래픽 못 보므로 IB Pod에서 비활성화 필수."**
