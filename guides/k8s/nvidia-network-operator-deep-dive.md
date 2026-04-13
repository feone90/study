# NVIDIA Network Operator Deep Dive

> 회사 DGX 클러스터에서 IB/RDMA를 K8s Pod에게 붙여주는 모든 컴포넌트를 한 번에 깔아주는 Operator.
> 이 문서를 다 읽으면 `values.yaml`의 옵션 하나하나가 왜 그렇게 켜져 있고 꺼져 있는지, 그리고 Pod이 `/dev/infiniband`를 받기까지 내부적으로 무슨 일이 일어나는지 전부 설명할 수 있게 된다.

---

## 0. 큰 그림 — 왜 Operator가 필요한가

"그냥 IB 쓰게 해주세요"를 위해 수동으로 해야 하는 일들:

1. **MOFED (Mellanox OFED) 드라이버 설치** — 커널 모듈(`mlx5_core`, `mlx5_ib`, `ib_core`, `ib_uverbs`) 로드, 유저스페이스 라이브러리(`libibverbs`, `librdmacm`) 설치
2. **SR-IOV VF 생성 (쓸 경우)** — `echo N > /sys/class/net/ibX/device/sriov_numvfs`
3. **RDMA Subsystem 모드 설정** — `rdma system set netns exclusive` (네트워크 네임스페이스별 RDMA 장치 격리)
4. **Multus CNI 배포** — kubelet이 여러 CNI를 호출할 수 있게 하는 meta-plugin
5. **CNI 플러그인 바이너리 배치** — host-device, sriov, ipoib, bridge 등을 `/opt/cni/bin`에 복사
6. **Device Plugin 배포** — `/dev/infiniband/uverbsX`, `/dev/infiniband/rdma_cm`을 K8s resource로 노출
7. **IPAM 배포** — secondary network용 IP 할당 (whereabouts)
8. **NFD (Node Feature Discovery)** — 어느 노드에 Mellanox NIC이 있는지 자동 라벨링
9. **PKey/GUID 관리 (IB subnet 분리 시)** — UFM 연동
10. **업그레이드 전략** — 드라이버 교체 시 노드 drain, 재부팅, rollback

이걸 한 Helm chart로 묶은 게 **NVIDIA Network Operator**. CRD `NicClusterPolicy`에 원하는 구성을 선언하면 Operator가 알아서 DaemonSet들을 깐다.

**회사 특이점**: MOFED는 MDS(벤더)가 호스트 OS에 직접 설치했기 때문에 `ofedDriver.deploy: false`. Operator는 "이미 준비된 호스트 드라이버" 위에 K8s 레이어만 얹는다.

---

## 1. 컴포넌트 지도

```
┌─────────────────────────────────────────────────────────────────┐
│                   NVIDIA Network Operator (Pod)                 │
│                       ↓ reconciles                              │
│                   NicClusterPolicy (CR)                         │
└─────────────────────────────────────────────────────────────────┘
                         │ 생성
       ┌─────────────────┼─────────────────┬─────────────────┐
       ↓                 ↓                 ↓                 ↓
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ MOFED        │  │ RDMA Shared  │  │ SR-IOV       │  │ Multus CNI   │
│ DaemonSet    │  │ Device Plugin│  │ Device Plugin│  │ DaemonSet    │
│ (드라이버)    │  │ DaemonSet    │  │ DaemonSet    │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ CNI Plugins  │  │ Whereabouts  │  │ ib-kubernetes│  │ IPoIB CNI    │
│ (바이너리 복사)│  │ (IPAM)       │  │ (PKey 관리)  │  │              │
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
```

**회사에서 실제 켜진 것 (values.yaml 기준)**:

| 컴포넌트 | 상태 | 이유 |
|---|---|---|
| `ofedDriver` | `deploy: false` | MDS가 호스트에 이미 설치 |
| `nvPeerDriver` | `deploy: false` | 커널 5.x+ `nvidia-peermem` 모듈로 대체됨 |
| `rdmaSharedDevicePlugin` | `deploy: true` | Pod에게 HCA를 공유(shared) 모드로 노출 |
| `sriovDevicePlugin` | `deploy: true` | Host-Device 리소스(`hostdev`) 노출용 |
| `ibKubernetes` | `deploy: false` | UFM 없음 → PKey 자동관리 불필요 |
| `secondaryNetwork.multus` | `deploy: true` | 멀티 NIC 핵심 |
| `cniPlugins` | `deploy: true` | host-device 등 바이너리 배치 |
| `ipoib` | `deploy: false` | IPoIB 대신 Host-Device 직접 할당 |
| `whereabouts` | `deploy: true` | IPAM |
| `nfd` | `enabled: false` | 외부 NFD 또는 수동 라벨 사용 |
| `sriovNetworkOperator` | `enabled: false` | SR-IOV VF 동적 관리 안함 |

→ 한 줄 요약: **"MOFED는 호스트, K8s는 Multus + Host-Device CNI + RDMA Shared DP로 HCA를 Pod에 붙여주는 구성"**.

---

## 2. NicClusterPolicy CRD — 단일 진실의 원천

Operator의 reconcile 대상은 `NicClusterPolicy`라는 클러스터 스코프 CR 한 개. Helm values → NicClusterPolicy → DaemonSet 순으로 펼쳐진다.

```yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy   # 클러스터에 딱 하나
spec:
  ofedDriver: {...}           # MOFED 컨테이너 스펙
  rdmaSharedDevicePlugin:
    config: |
      {"configList":[{"resourceName":"rdma_shared_device_a","rdmaHcaMax":63,
       "selectors":{"vendors":["15b3"]}}]}
  sriovDevicePlugin:
    config: |
      {"resourceList":[{"resourceName":"hostdev","selectors":{"vendors":["15b3"]}}]}
  secondaryNetwork:
    multus: {...}
    cniPlugins: {...}
    ipamPlugin: {...}
```

**관찰 방법**:
```bash
kubectl get nicclusterpolicies
kubectl describe nicclusterpolicy nic-cluster-policy
kubectl get pods -n nvidia-network-operator -o wide
```

---

## 3. 각 컴포넌트 심층 해부

### 3.1 MOFED DaemonSet (회사에선 OFF지만 원리는 알아야 함)

**하는 일**: 호스트 커널에 Mellanox OFED 커널 모듈(`mlx5_core`, `mlx5_ib`, `ib_umad`, `ib_uverbs`, `rdma_cm`)을 주입.

**주입 방식**:
- Privileged 컨테이너 + `/lib/modules`, `/usr/src`, `/run/mellanox/drivers` hostPath 마운트
- 컨테이너 안에서 `depmod` → `modprobe mlx5_ib` 실행 → 호스트 커널에 모듈 로드
- 호스트 OS는 건드리지 않는 척하지만 실제로는 커널 모듈 공간을 공유함 (컨테이너는 프로세스 격리일 뿐 커널은 하나)

**업그레이드 시**: `upgradePolicy.drain.enable: true` → `kubectl drain` → 모듈 언로드 → 새 버전 로드 → uncordon. `maxParallelUpgrades: 1`이면 롤링으로 한 노드씩.

**회사가 MDS 호스트 설치를 택한 이유**:
- DGX OS는 커스텀 커널/펌웨어. Operator가 실수로 잘못된 MOFED 버전을 올리면 노드 전체 IB가 죽음 → IDC 출동
- 펌웨어(`mlxfwmanager`)는 Operator 범위 밖 → 어차피 벤더가 해야 함

**확인 명령**:
```bash
lsmod | grep mlx5      # mlx5_core, mlx5_ib
ofed_info -s           # OFED 버전
ibv_devinfo            # HCA 상태
```

---

### 3.2 RDMA Shared Device Plugin (`rdmaSharedDevicePlugin.deploy: true`)

**핵심 질문**: "노드에 HCA 4장인데 Pod 10개가 각자 RDMA 쓰고 싶다면?"

두 가지 모드:
- **Shared**: 여러 Pod이 **같은 HCA device file을 공유**. 각 Pod이 `/dev/infiniband/uverbs0`을 동시에 open. 리소스 카운트는 논리적(예: 63).
- **Exclusive (SR-IOV)**: VF 하나당 Pod 하나. 커널이 Virtual Function을 만들어 물리적으로 분리.

**회사 config**:
```yaml
resources:
  - name: rdma_shared_device_a
    vendors: [15b3]       # Mellanox PCI vendor ID
```
→ K8s resource name: `rdma/rdma_shared_device_a`. Pod이 `requests: rdma/rdma_shared_device_a: 1`로 요청하면 Plugin이 `/dev/infiniband/*`, `/dev/infiniband/rdma_cm`을 컨테이너에 bind-mount.

**동작 메커니즘** (kubelet ↔ device plugin gRPC):
1. Plugin이 `/var/lib/kubelet/device-plugins/kubelet.sock`에 Register RPC
2. kubelet이 `ListAndWatch` 구독 → Plugin이 "device ID 리스트"를 스트리밍 (shared 모드는 같은 HCA에 대해 여러 가짜 ID 생성, 기본 `rdmaHcaMax: 63`)
3. Pod 스케줄 시 kubelet이 `Allocate(device_ids)` 호출 → Plugin이 `{Mounts, Devices, Envs}` 반환
4. runc가 OCI spec에 반영 → 컨테이너 안에 `/dev/infiniband/*` 등장

**주의**: Shared 모드는 RDMA CM 충돌 가능성 (같은 GID/QP 공간 공유). 대규모 멀티테넌트엔 SR-IOV 권장. 회사는 노드당 Pod 수가 적고 신뢰 가능한 워크로드만 돌려서 Shared로 충분.

---

### 3.3 SR-IOV Device Plugin (회사는 Host-Device 용도로만 사용)

**일반적 용도**: PF(Physical Function) 하나에서 VF(Virtual Function) N개를 만들어 Pod별 배타 할당.

**회사 config**:
```yaml
resources:
  - name: hostdev
    vendors: [15b3]
```
→ K8s resource: `nvidia.com/hostdev`. 이름은 `sriov-device-plugin`이지만 `selectors`에 `pfNames`/`rootDevices`를 안 적고 `vendors`만 썼기 때문에 **VF가 아니라 전체 PCI device(= 물리 HCA 자체)**를 리소스로 노출.

**왜 이렇게 쓰나**: Host-Device CNI가 Pod의 netns로 물리 인터페이스를 **통째로 이동**시키는데, 이때 K8s 스케줄러에게 "이 노드에 HCA가 몇 개 남았는지" 알려줄 카운터가 필요. SR-IOV DP를 카운터로 재활용.

→ **1 Node = 1 Pod = 1 NAD** 제약의 뿌리: Host-Device는 PF를 옮기므로 한 PF는 한 netns에만 존재 가능.

---

### 3.4 Multus CNI — 멀티 네트워크의 메타 플러그인

**문제**: 표준 K8s Pod은 NIC 1개(eth0)만 가짐. IB + Ethernet 둘 다 붙이려면?

**해결**: Multus는 CNI **위에서** 동작하는 meta-plugin. kubelet은 Multus를 primary CNI로 호출 → Multus가 Pod annotation을 읽고 **여러 delegate CNI**를 순차 호출.

**배포 구조**:
- DaemonSet이 모든 노드에 `/opt/cni/bin/multus`, `/etc/cni/net.d/00-multus.conf` 배치
- `00-` prefix → kubelet이 사전순으로 첫 번째 conf를 메인 CNI로 인식 → Multus가 메인이 됨
- Multus의 `delegates` 필드에 실제 메인 CNI(회사: Calico 또는 Cilium) 지정 → eth0는 평소대로 생김

**Pod에 IB 붙이는 플로우**:
```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: ibp64s0-net  # NAD 이름
spec:
  containers:
  - resources:
      limits:
        nvidia.com/hostdev: 1
        rdma/rdma_shared_device_a: 1
```

1. kubelet → Multus `ADD` 호출
2. Multus가 primary delegate (Calico) 호출 → eth0 생성
3. Multus가 annotation 파싱 → `ibp64s0-net` NAD 조회
4. NAD의 `config` JSON (type: host-device, device: ibp64s0) 로 **Host-Device CNI** 호출
5. Host-Device CNI: `ip link set ibp64s0 netns <pod_netns>` → Pod 안에 `net1` 으로 등장
6. RDMA subsystem (`rdma system` = exclusive mode면) 해당 RDMA device도 같은 netns로 이동
7. IPAM (whereabouts) 호출 → IP 할당
8. Multus가 결과 합쳐 kubelet에 반환

---

### 3.5 CNI Plugins (`containernetworking/plugins`)

`cniPlugins.deploy: true`는 표준 CNI 플러그인 바이너리들(`host-device`, `bridge`, `macvlan`, `ipvlan`, `loopback`, `portmap`, `tuning`)을 `/opt/cni/bin`에 DaemonSet으로 복사. Multus의 delegate로 호출됨.

**Host-Device CNI 내부**:
```c
// 의사코드
device = find_link_by_name(conf.device);    // "ibp64s0"
netlink_setns(device, container_netns_fd);  // 커널 syscall: setns
rename_link(device, "net1");
bring_up(device);
```
→ **물리 인터페이스가 Pod netns로 이사. 호스트에선 `ip link`에서 사라짐**. Pod 삭제 시 복원.

---

### 3.6 Whereabouts — Cluster-wide IPAM

**문제**: DHCP 없는 IB 네트워크에서 Pod별 IP 어떻게 할당?

**Whereabouts**: 클러스터 전역 IPAM. IP 할당 상태를 CRD(`IPPool`, `OverlappingRangeIPReservation`)에 저장 → 어느 노드에서 호출해도 중복 없음.

NAD 예시:
```yaml
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "type": "host-device",
      "device": "ibp64s0",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.100.0/24",
        "exclude": ["192.168.100.1/32"]
      }
    }
```
→ etcd에 락 기반으로 할당 기록, Pod 삭제 시 해제.

---

### 3.7 IPoIB CNI (회사는 OFF)

**IPoIB**: InfiniBand 위에 IP 레이어를 얹어 TCP/IP가 동작하게 하는 상위 프로토콜. `ib0` 같은 인터페이스가 그것.

회사는 Host-Device로 IB 장치 자체를 옮기기 때문에 IPoIB 인터페이스도 Pod로 같이 따라감 → 별도 CNI 불필요.

IPoIB CNI는 **하나의 IPoIB 링크에서 sub-interface를 여러 Pod에 나눠주는** 시나리오에서 사용. 회사는 "한 PF = 한 Pod" 원칙이라 필요 없음.

---

### 3.8 ib-kubernetes & UFM (회사는 OFF)

**UFM (Unified Fabric Manager)**: NVIDIA의 IB 패브릭 관리 SW. PKey(Partition Key)로 IB 트래픽을 가상 네트워크처럼 격리.

**ib-kubernetes**: UFM과 연동해 Pod이 요청한 PKey 기반으로 HCA GUID 할당.

회사가 OFF인 이유: 멀티테넌트 격리가 필요 없는 단일 팀 클러스터 + UFM 미도입. 전체 IB가 default partition(0x7fff)으로 동작.

---

### 3.9 NFD (Node Feature Discovery)

노드의 PCI/CPU 기능을 자동 탐지해 라벨 부착: `feature.node.kubernetes.io/pci-15b3.present=true` 등. Operator는 이 라벨로 Mellanox NIC 있는 노드에만 DaemonSet Pod을 스케줄.

회사는 `nfd.enabled: false`. 외부 NFD 또는 수동 라벨(`node-role.kubernetes.io/worker`) 사용 추정.

---

### 3.10 SR-IOV Network Operator (회사는 OFF)

별도 Operator로 SR-IOV VF를 동적으로 생성/관리. `SriovNetworkNodePolicy` CR로 "이 노드 이 PF에 VF N개 만들어라" 선언. 회사는 SR-IOV 안 쓰므로 OFF.

---

## 4. Pod이 RDMA 장치를 받기까지 — End-to-End 플로우

```
1. 사용자: PyTorchJob 생성 (pod annotation: k8s.v1.cni.cncf.io/networks: ibp64s0-net)
                  ↓
2. kube-scheduler: 리소스 체크
   - nvidia.com/gpu: 8  (GPU Device Plugin)
   - rdma/rdma_shared_device_a: 1  (RDMA Shared DP)
   - nvidia.com/hostdev: 1  (SR-IOV DP as counter)
   → 조건 만족하는 노드에 배치
                  ↓
3. kubelet: Allocate RPC to each Device Plugin
   - RDMA Shared DP → {Devices: [/dev/infiniband/uverbs0, /dev/infiniband/rdma_cm]}
   - SR-IOV DP      → {Envs: PCIDEVICE_NVIDIA_COM_HOSTDEV=0000:64:00.0}
                  ↓
4. kubelet: containerd에 CRI CreateContainer
                  ↓
5. containerd → runc → OCI spec에 device/mount 반영
                  ↓
6. kubelet: CNI ADD (Multus가 메인)
   a) Multus → Calico (delegate) → eth0 생성
   b) Multus → NAD lookup → Host-Device CNI 호출
   c) Host-Device CNI: ibp64s0를 Pod netns로 이동 → net1
   d) RDMA subsystem: rdma link를 해당 netns로 이동
   e) Whereabouts: IP 할당
                  ↓
7. Pod 시작 → 컨테이너 안에서:
   $ ls /dev/infiniband/    # uverbs0, rdma_cm
   $ ibv_devinfo            # HCA 보임
   $ ip link                # eth0, net1 (ibp64s0)
   $ rdma link              # RDMA device
                  ↓
8. PyTorch NCCL init → ibv_open_device → RDMA verbs → IB 스위치 거쳐 다른 노드와 AllReduce
```

---

## 5. 회사 values.yaml 라인별 해석

```yaml
nfd.enabled: false                    # 외부 NFD 또는 수동 라벨 쓰는 중
psp.enabled: false                    # PSP deprecated (K8s 1.25+) → PSS 사용
sriovNetworkOperator.enabled: false   # SR-IOV VF 동적 관리 안 씀

ofedDriver.deploy: false              # MDS가 호스트에 MOFED 설치함
ofedDriver.version: 23.04-0.5.3.3.1  # deploy=false라 실제론 미사용, 메타정보
ofedDriver.upgradePolicy.drain.enable: true  # 켜져있어도 deploy=false면 무의미

nvPeerDriver.deploy: false            # 최신 커널은 nvidia-peermem 내장, 구형 드라이버 불필요

rdmaSharedDevicePlugin.deploy: true   # Pod이 RDMA 쓸 수 있게 하는 핵심
  resources:
    - name: rdma_shared_device_a      # Pod requests: rdma/rdma_shared_device_a
      vendors: [15b3]                 # Mellanox PCI vendor ID만 필터

sriovDevicePlugin.deploy: true        # Host-Device 카운터 역할로 재활용
  resources:
    - name: hostdev                   # Pod requests: nvidia.com/hostdev
      vendors: [15b3]

ibKubernetes.deploy: false            # UFM 없음 → PKey 자동관리 불필요

secondaryNetwork.deploy: true
  cniPlugins.deploy: true             # host-device 바이너리 배치
  multus.deploy: true                 # 멀티 NIC 핵심
  ipoib.deploy: false                 # Host-Device로 대체
  ipamPlugin (whereabouts).deploy: true  # IB 네트워크 IP 할당
```

**이 조합의 의미**: "MOFED는 벤더가 깔았고, K8s는 Multus + Host-Device CNI로 PF를 통째로 Pod에 넘기며, RDMA Shared DP로 /dev/infiniband를 공유 노출한다. SR-IOV는 안 쓰고, UFM도 없다."

---

## 6. NetworkAttachmentDefinition (NAD) 실전

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ibp64s0-net
  namespace: team-a
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "ibp64s0-net",
      "type": "host-device",
      "device": "ibp64s0",
      "ipam": {
        "type": "whereabouts",
        "range": "192.168.101.0/24"
      }
    }
```

**핵심 포인트**:
- `device: ibp64s0` — **PCI 경로 기반 네이밍** (`ibp<bus>s<slot>`). 노드가 바뀌어도 같은 슬롯이면 같은 이름 → 여러 노드 공용 가능.
- NAD는 namespace-scoped. 팀별로 격리.
- 동일 PF를 두 NAD에 쓰면 충돌. "1 PF = 1 NAD" 유지.

**Pod 요청**:
```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: ibp64s0-net
spec:
  containers:
  - name: trainer
    resources:
      limits:
        nvidia.com/gpu: 8
        rdma/rdma_shared_device_a: 1
        nvidia.com/hostdev: 1
```

---

## 7. 트러블슈팅 체크리스트

### Pod 안에서 RDMA 장치가 안 보일 때

```bash
# 1. 호스트에 드라이버 있나
ssh dgx-node-1 'lsmod | grep mlx5; ofed_info -s'

# 2. HCA 인식되나
ssh dgx-node-1 'ibv_devinfo; ibstat'

# 3. Device Plugin 등록됐나
kubectl get node dgx-node-1 -o json | jq '.status.allocatable' | grep -E 'rdma|hostdev'

# 4. Operator DaemonSet 건강한가
kubectl get pods -n nvidia-network-operator -o wide

# 5. NicClusterPolicy 상태
kubectl get nicclusterpolicy -o yaml | grep -A5 state

# 6. Multus conf 배치됐나
ssh dgx-node-1 'ls /etc/cni/net.d/; cat /etc/cni/net.d/00-multus.conf'

# 7. CNI 바이너리
ssh dgx-node-1 'ls /opt/cni/bin/ | grep -E "host-device|multus|whereabouts"'

# 8. Pod에서 직접
kubectl exec -it <pod> -- sh -c 'ls /dev/infiniband/; ibv_devinfo; ip link; rdma link'
```

### 자주 나오는 에러

| 증상 | 원인 | 해결 |
|---|---|---|
| `failed to find plugin "host-device"` | `cniPlugins.deploy: false` 이거나 바이너리 복사 실패 | Operator Pod 로그, DaemonSet 재시작 |
| `no IP addresses available in range` | Whereabouts pool 소진, 삭제된 Pod의 IP 미반환 | `kubectl get ippools -A`, 수동 정리 |
| `device or resource busy` | Host-Device: 같은 PF를 두 Pod이 노렸음 | NAD/스케줄링 재검토 |
| NCCL timeout / hang | PKey 불일치, MTU 불일치, subnet manager down | `ibstat`, `opensm` 상태 확인 |
| Pod scheduling pending | Device Plugin 리소스 카운트 0 | Plugin 로그, `kubectl describe node` |

### Operator/Plugin 로그

```bash
kubectl logs -n nvidia-network-operator deploy/network-operator
kubectl logs -n nvidia-network-operator ds/rdma-shared-dp-ds
kubectl logs -n nvidia-network-operator ds/kube-multus-ds
kubectl logs -n nvidia-network-operator ds/sriov-device-plugin
```

---

## 8. 커널 레벨 백그라운드 (보너스)

### Network namespace와 RDMA
- 리눅스 커널 5.3+ RDMA subsystem은 namespace 인식(`rdma system set netns exclusive`).
- Host-Device CNI가 netlink로 `IFLA_NET_NS_FD` 보내면 커널이 해당 link와 관련 RDMA device를 타깃 netns로 이동.
- 이게 **Shared 모드에서도 Pod 격리가 일부 작동하는 이유**. 다만 같은 HCA의 uverbs device file을 여러 namespace에서 열면 GID/QP 네임스페이스가 공유될 수 있음 → 보안이 중요하면 SR-IOV + exclusive.

### Device Plugin이 할당하는 것의 실체
- `/dev/infiniband/uverbs0` = **character device** (`c 231 192`). verbs API가 mmap/ioctl로 커널 드라이버와 대화.
- `/dev/infiniband/rdma_cm` = Connection Manager. QP 설정, 경로 해석.
- bind-mount 시 cgroup `devices` controller로 access 허용.

### GPUDirect RDMA와의 연결
- `nv_peer_mem` (구형) 또는 `nvidia-peermem` (신형) 커널 모듈이 `ib_register_peer_memory_client`로 등록 → RDMA 드라이버가 GPU 메모리를 pinned memory처럼 취급 → HCA가 GPU VRAM에 **PCIe P2P DMA**.
- `dmesg | grep nvidia-peermem` 로 확인.
- Network Operator의 `nvPeerDriver`는 구형용. 회사는 `false`로 두고 커널 내장 모듈 사용.

---

## 9. 면접 Q&A (예상)

**Q1. NVIDIA Network Operator가 하는 일을 한 문장으로 설명해보세요.**
A: Mellanox MOFED, Multus, CNI plugins, RDMA/SR-IOV Device Plugin, Whereabouts IPAM 등 IB/RDMA를 K8s에서 쓰기 위한 모든 컴포넌트를 `NicClusterPolicy` CR로 선언적으로 배포·관리하는 Operator입니다.

**Q2. 회사에서 `ofedDriver.deploy: false`인 이유는?**
A: MDS(벤더)가 DGX OS 호스트에 MOFED를 직접 설치했기 때문입니다. Operator가 컨테이너로 덮어쓰면 DGX 커스텀 커널/펌웨어와 충돌 위험이 있어 호스트 설치가 안전합니다.

**Q3. Multus 없이는 왜 IB를 Pod에 붙일 수 없나요?**
A: 표준 K8s는 Pod당 CNI 하나만 호출해 eth0 하나만 만듭니다. Multus는 meta-plugin으로 여러 delegate CNI를 순차 호출해 추가 인터페이스(net1=IB)를 붙입니다.

**Q4. RDMA Shared Device Plugin과 SR-IOV Device Plugin의 차이는?**
A: Shared는 하나의 PF를 여러 Pod이 `/dev/infiniband`를 공유해 사용하고(논리적 리소스 카운트), SR-IOV는 PF에서 VF를 만들어 Pod마다 배타적 VF를 할당합니다. 격리/보안이 중요하면 SR-IOV, 신뢰 워크로드엔 Shared가 간단합니다.

**Q5. Host-Device CNI에서 "1 Node = 1 Pod = 1 NAD"가 되는 이유는?**
A: Host-Device는 물리 PF를 Pod netns로 이동시킵니다. 한 PF는 한 시점에 한 netns에만 존재할 수 있어 PF 하나 = Pod 하나가 원칙이고, NAD가 PF를 지목하므로 동일 PF를 다른 NAD가 또 쓸 수 없습니다.

**Q6. Whereabouts가 왜 필요하죠? 기본 host-local IPAM 쓰면 안 되나요?**
A: host-local은 노드별 독립 할당이라 여러 노드가 같은 대역을 쓰면 IP 충돌이 납니다. Whereabouts는 클러스터 전역 IPPool CRD로 상태를 공유해 중복을 방지합니다.

**Q7. `nvPeerDriver.deploy: false`인데 GPUDirect RDMA는 어떻게 동작하나요?**
A: 최신 커널(5.x+)은 NVIDIA 드라이버에 `nvidia-peermem` 모듈이 포함돼 있어 구형 `nv_peer_mem`이 불필요합니다. `lsmod | grep peermem`으로 확인 가능.

**Q8. `NicClusterPolicy` 리소스가 Stuck 상태일 때 어떻게 디버깅하나요?**
A: 1) Operator Pod 로그, 2) NicClusterPolicy `.status.appliedStates` 확인, 3) 각 DaemonSet의 Pod 상태, 4) 노드 라벨/taint, 5) CNI conf 및 바이너리 배치 여부 순서로 내려갑니다.

**Q9. IPoIB CNI를 안 쓰고 Host-Device를 택한 이유는?**
A: 회사는 Pod당 HCA를 통째로 주는 단일 테넌시 모델입니다. IPoIB CNI는 하나의 IPoIB 링크를 여러 Pod에 나눠주는 용도라 이 모델과 안 맞습니다.

**Q10. SR-IOV Network Operator와 NVIDIA Network Operator의 관계는?**
A: NVIDIA Network Operator가 SR-IOV Network Operator를 서브차트로 포함할 수 있습니다(`sriovNetworkOperator.enabled`). SR-IOV VF 동적 생성/관리가 필요할 때 켭니다. 회사는 VF 안 써서 `false`.

---

## 10. 한 줄 요약

**NVIDIA Network Operator = `NicClusterPolicy` CR 하나로 "MOFED + Multus + Host-Device/SR-IOV CNI + RDMA/SR-IOV Device Plugin + Whereabouts IPAM"을 선언적으로 배포해 Pod이 IB/RDMA를 쓸 수 있게 해주는 Operator. 회사는 MOFED만 호스트에 두고 나머지는 Operator로 관리한다.**
