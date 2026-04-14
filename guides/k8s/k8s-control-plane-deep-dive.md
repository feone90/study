# Kubernetes Control Plane 완전 정복

> **목적**: K8s를 "사용"이 아닌 "운영"하는 사람의 시각으로 본다. etcd, API Server, Scheduler, Controller, kubelet의 내부 동작과 Operator 패턴까지.

---

## 0. K8s의 본질

> **"분산 시스템에서 declarative하게 원하는 상태(desired state)를 선언하면, 그것을 현재 상태(current state)로 만들어주는 reconcile loop들의 집합"**

이 한 문장이 K8s의 모든 것입니다.

```
[사용자] "Pod 3개 있어야 함" 선언 (desired)
[K8s] 현재 0개 → 3개 만들어야 함 (current → desired로 수렴)
[누가 죽으면] 다시 부족 → 만듦 (계속 수렴)
```

---

## 1. 컨트롤 플레인 컴포넌트

```
[Control Plane Node]
├── kube-apiserver    ← 모든 통신의 중심
├── etcd              ← 유일한 영구 상태 저장소
├── kube-scheduler    ← Pod을 어느 노드에 둘지 결정
├── kube-controller-manager  ← 각종 컨트롤러 모음
└── cloud-controller-manager (옵션)

[Worker Node]
├── kubelet           ← 노드 위 Pod 관리자
├── kube-proxy        ← Service 네트워킹
└── containerd        ← 컨테이너 런타임
```

---

## 2. etcd - K8s의 유일한 진실

### 2.1 무엇

**분산 key-value store**. K8s의 모든 상태(Pod, Service, ConfigMap, Secret 등)가 여기 저장됨.

```bash
# etcd에 저장된 키 보기 (제한적 접근)
ETCDCTL_API=3 etcdctl get / --prefix --keys-only | head
# /registry/pods/default/mypod
# /registry/services/default/myservice
# /registry/configmaps/...
# /registry/secrets/...
```

K8s 객체 = etcd의 한 줄. **etcd가 죽으면 K8s 죽음**.

### 2.2 Raft 합의 알고리즘

etcd는 **다중 노드**로 배포 (보통 3개 또는 5개, 홀수).

```
[etcd-1: leader]   ← 모든 쓰기는 여기로
[etcd-2: follower]
[etcd-3: follower]

쓰기:
1. 클라이언트 → leader
2. leader가 followers에 복제 명령
3. 과반수가 commit ack
4. leader가 클라이언트에 ack
```

장애 시:
- leader 죽으면 followers 중 새 leader 선출
- 과반수만 살아 있으면 동작 (quorum)
- 3노드 → 1대 죽어도 OK, 2대 죽으면 마비

### 2.3 watch 메커니즘

K8s의 핵심 패턴.

```
[Controller] 
   ↓ etcd.watch("/registry/pods/")
[etcd]
   ↓ Pod 변경 시 즉시 이벤트 푸시
[Controller가 반응]
```

폴링 안 함. 이벤트 기반. 효율적.

### 2.4 compaction & defrag (★ 운영 필수)

etcd는 모든 revision을 유지 → 시간 지나면 DB 커짐, 쓰기 느려짐.

```bash
# 현재 revision 확인
rev=$(etcdctl endpoint status --write-out=json | jq -r '.[0].Status.header.revision')

# 5분 전 revision까지 compact
etcdctl compact $rev

# 공간 회수 (db 파일 shrink)
etcdctl defrag --cluster
```

- K8s apiserver에 `--etcd-compaction-interval=5m`로 자동 compaction.
- defrag는 멤버별 순차 실행 (leader 영향 있음).
- DB size 8GB 이상 시 apiserver alarm `NOSPACE` 발생 → 쓰기 차단.

### 2.5 백업이 절대 중요

```bash
# etcd 백업
ETCDCTL_API=3 etcdctl snapshot save backup.db

# 복원
ETCDCTL_API=3 etcdctl snapshot restore backup.db
```

etcd 데이터 잃으면 클러스터의 모든 것 사라짐. 반드시 정기 백업.

---

## 3. kube-apiserver

### 3.1 역할

**모든 통신의 단일 진입점**. 어떤 컴포넌트든 K8s 상태를 보거나 바꾸려면 API Server를 통해서.

```
kubectl
controller-manager
scheduler
kubelet (각 노드)
   ↓ 모두
[kube-apiserver]
   ↓
[etcd]
```

### 3.2 요청 처리 흐름

```
[1] HTTP 요청 도착 (TLS)
   ↓
[2] Authentication (인증: 누구냐)
    - x509 인증서, Bearer token, OIDC, ServiceAccount token
   ↓
[3] Authorization (인가: 뭘 할 수 있냐)
    - RBAC, ABAC, Webhook
   ↓
[4] Mutating Admission (변형: 자동 수정)
    - 예: 자동 sidecar 주입(Istio), 기본 리소스 채움
   ↓
[5] Object Validation (객체 자체 유효성)
    - 필수 필드, 타입 등
   ↓
[6] Validating Admission (검증: 정책 위반 체크)
    - 예: PodSecurityPolicy, OPA Gatekeeper
   ↓
[7] etcd에 저장
   ↓
[8] watch 구독자들에게 이벤트 송신
```

각 단계마다 거부 가능. 거부되면 바로 응답.

### 3.3 API Priority and Fairness (APF, v1.29 stable)

대량 watch/list가 apiserver를 압도 → 이전엔 `--max-requests-inflight`로 단순 제한 → 중요 요청까지 drop.

APF는 `FlowSchema` + `PriorityLevelConfiguration`로 요청을 분류·가중 처리:

```yaml
# 시스템 리더 요청은 최우선
FlowSchema: system-leader-election → priorityLevel: leader-election
# 컨트롤러 workqueue는 workload-high
# 사용자 요청은 workload-low (fair queueing)
```

429 `Too Many Requests` + `X-Kubernetes-PF-FlowSchema-UID` 헤더로 어느 schema가 throttle 됐는지 진단.

### 3.4 Leader Election

controller-manager, scheduler는 HA를 위해 다중 인스턴스 배포 → 하나만 active.

```
leader: lease 객체(coordination.k8s.io/v1) 주기적 renew
follower: lease TTL 만료 감시 → 만료 시 takeover 시도
```

`--leader-elect=true --leader-elect-lease-duration=15s --leader-elect-renew-deadline=10s`.

### 3.5 Admission Controller (확장점)

```
Mutating Admission Webhook → 리소스 수정 가능
Validating Admission Webhook → 거부만 가능
```

K8s의 강력한 확장 메커니즘. Istio의 sidecar 자동 주입, OPA의 정책 강제 모두 이거.

---

## 4. Controller Pattern (★ K8s의 영혼)

### 4.1 컨셉

> **"desired state를 보고, current state와 다르면 같아지게 만든다"** 를 무한 반복.

```python
# 의사코드
while True:
    desired = api.get_desired_state()
    current = api.get_current_state()
    if desired != current:
        take_action_to_reconcile(desired, current)
```

### 4.2 예: ReplicaSet Controller

```
ReplicaSet "myapp" 선언: replicas: 3
   ↓
컨트롤러 watch:
   - 현재 Pod 0개 → 3개 생성
   - 누군가 Pod 1개 죽임 → 1개 추가 생성
   - replicas 3 → 5로 변경 → 2개 추가 생성
   - replicas 5 → 2로 변경 → 3개 삭제
```

### 4.3 K8s 빌트인 컨트롤러들

`kube-controller-manager` 안에 다 들어 있음:

- **ReplicaSet Controller**
- **Deployment Controller** (rolling update)
- **StatefulSet Controller**
- **DaemonSet Controller**
- **Job Controller**
- **Node Controller** (노드 헬스 체크)
- **ServiceAccount Controller**
- **Endpoint Controller** (Service ↔ Pod 매핑)
- **PersistentVolume Controller**

### 4.4 Informer / WorkQueue 패턴

컨트롤러는 효율을 위해 다음 패턴을 사용:

```
[API Server]
   ↓ watch
[Informer] (로컬 캐시 유지 + 이벤트 디스패치)
   ↓ Add/Update/Delete 이벤트
[Event Handler]
   ↓ 변경된 객체 key를 큐에 추가
[Work Queue]
   ↓ rate limit, dedup
[Worker Goroutine]
   ↓ key 받아 reconcile()
[Reconcile 로직]
```

핵심 이점:
- Informer가 로컬 캐시 유지 → API Server 부하 감소
- WorkQueue가 같은 객체 중복 처리 방지
- Rate limit으로 API Server 보호

이게 모든 K8s 컨트롤러/Operator의 공통 구조.

---

## 5. Scheduler

### 5.1 역할

새로 생성된 Pod (NodeName이 비어 있는)을 어느 노드에 배치할지 결정.

### 5.2 동작

```
[새 Pod 발견]
   ↓
[Filter 단계] 후보 노드 추리기
   - 자원 충분? (CPU/메모리/GPU)
   - taint/toleration 맞는가?
   - nodeSelector/affinity 만족?
   - PVC 마운트 가능?
   ↓
[Score 단계] 후보 노드 점수 매기기
   - 자원 여유 비율
   - 이미지 캐시 보유
   - inter-pod affinity
   ↓
[최고 점수 노드 선택]
   ↓
[Pod.spec.nodeName 업데이트]
   ↓
[해당 노드의 kubelet이 watch로 감지 → 생성]
```

### 5.3 affinity / anti-affinity

```yaml
# Pod이 GPU 노드에만 가도록
nodeSelector:
  hardware: gpu

# 같은 학습 잡의 Pod들은 같은 zone에 (성능)
podAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    podAffinityTerm:
      labelSelector: { matchLabels: { job: training } }
      topologyKey: topology.kubernetes.io/zone

# DB Pod끼리는 다른 노드에 (HA)
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
  - labelSelector: { matchLabels: { app: db } }
    topologyKey: kubernetes.io/hostname
```

### 5.4 taint / toleration

노드를 "거부"하는 메커니즘.

```yaml
# 노드에 taint 부여
kubectl taint nodes dgx01 dedicated=ml:NoSchedule
# → 이 노드에는 toleration 있는 Pod만 들어옴

# Pod이 toleration
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: ml
    effect: NoSchedule
```

DGX 노드를 ML 워크로드 전용으로 만들 때 사용.

---

## 6. kubelet

### 6.1 역할

각 노드에서 도는 에이전트. **노드 위 모든 Pod의 라이프사이클 책임**.

### 6.2 핵심 루프

```
[1] API Server를 watch (자기 노드 Pod만)
[2] 새 Pod이나 변경 감지
[3] CRI(containerd)에 컨테이너 생성/시작/정지 요청
[4] CNI 호출 (네트워크 설정)
[5] Volume mount
[6] Liveness/Readiness probe 실행
[7] 컨테이너 상태/로그/메트릭을 API Server에 보고
```

### 6.3 노드 헬스 보고

```
kubelet → API Server: "나 살아 있어" (10초마다)
     ↓
Node Controller 감지: heartbeat 늦으면 NotReady 마킹
     ↓
NoExecute taint 추가 → Pod evict
```

### 6.4 Pod Lifecycle

```
Pending → Running → Succeeded/Failed
                 ↓
              Crashed → Restart (정책 따라)
```

상태 변화 모두 kubelet이 관리해서 API Server에 보고.

---

## 7. kube-proxy & Service

### 7.1 Service란

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector: { app: myapp }
  ports:
  - port: 80
```

`myapp.default.svc.cluster.local`이라는 가상 endpoint 제공. 뒤의 실제 Pod이 바뀌어도 클라이언트는 이 이름만 알면 됨.

### 7.2 동작

```
[Endpoint Controller]
   ↓ Service의 selector에 매칭되는 Pod IP 목록 유지
   ↓ Endpoints / EndpointSlice 객체 업데이트
[kube-proxy]
   ↓ 각 노드에서 watch
   ↓ iptables/IPVS 규칙 생성
[클라이언트] 10.0.0.1 (Service IP)로 요청
   ↓ iptables가 실제 Pod IP로 DNAT
[Pod에 도달]
```

### 7.3 모드

- **iptables** (기본): 규칙 많아지면 느려짐
- **IPVS**: 커널 L4 로드밸런서, 빠름. 대규모 클러스터에 권장
- **eBPF (Cilium)**: 가장 빠름, kube-proxy 대체

---

## 8. CRD (Custom Resource Definition) & Operator

### 8.1 CRD

K8s의 객체 종류를 **사용자가 추가**할 수 있는 메커니즘.

```yaml
# 예: PyTorchJob CRD
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: pytorchjobs.kubeflow.org
spec:
  group: kubeflow.org
  versions: [...]
  names:
    kind: PyTorchJob
    plural: pytorchjobs
```

CRD 만들면 새 객체 타입이 K8s API에 추가됨.

### 8.2 Operator 패턴

> **"CRD + Custom Controller = Operator"**

```
[CRD 정의: PyTorchJob]
   ↓
[Custom Controller (Operator)]
   ↓ PyTorchJob을 watch
   ↓ PyTorchJob 객체 생기면:
     - Master Pod 생성
     - Worker Pod들 생성
     - 환경 변수(MASTER_ADDR 등) 주입
     - 상태 추적 / 정리
```

K8s의 controller pattern을 그대로 사용자 도메인에 적용한 것.

### 8.3 우리 환경의 Operator들

```
- NVIDIA Network Operator → IB 관련 자동 설치
- NVIDIA GPU Operator → 드라이버, device plugin 자동 설치
- Kubeflow Training Operator → PyTorchJob, TFJob 등 관리
- Ceph CSI 관련 operator
- Prometheus Operator → ServiceMonitor 등
```

K8s 운영 = Operator를 잘 활용/제작하는 일.

### 8.4 Operator 만들기

```bash
# kubebuilder로 scaffolding
kubebuilder init --domain mycompany.com
kubebuilder create api --group ml --version v1 --kind TrainingJob

# 코드 작성 (Go)
# Reconcile 함수 안에 비즈니스 로직
```

```go
func (r *TrainingJobReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var job mlv1.TrainingJob
    if err := r.Get(ctx, req.NamespacedName, &job); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // desired vs current 비교
    // 필요한 K8s 객체(Pod, Service 등) 생성/수정/삭제

    return ctrl.Result{}, nil
}
```

---

## 9. RBAC (Role-Based Access Control)

### 9.1 구조

```
[User / ServiceAccount]
   ↓ binding
[RoleBinding / ClusterRoleBinding]
   ↓ ref
[Role / ClusterRole]
   ↓ contains
[Rules: verbs + resources]
```

### 9.2 예시

```yaml
# 권한 정의
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: slm
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
# 사용자에게 부여
kind: RoleBinding
subjects:
- kind: User
  name: alice
roleRef:
  kind: Role
  name: pod-reader
```

### 9.3 ServiceAccount

Pod이 K8s API에 접근할 때 쓰는 정체성.

```yaml
spec:
  serviceAccountName: my-sa  # 이 SA의 권한으로 K8s API 접근
```

토큰이 자동으로 Pod에 마운트됨 (`/var/run/secrets/kubernetes.io/serviceaccount/token`).

---

## 10. 우리 회사 K8s 운영 관점

### 10.1 멀티 테넌트 격리

```
namespace: slm, semi-parametric, vlm, mlops, nia, ...
   ↓
- ResourceQuota: 팀별 GPU/CPU/메모리 한도
- LimitRange: Pod별 기본/최대 리소스
- NetworkPolicy: 팀 간 네트워크 격리
- RBAC: 팀별 권한
- NAD: 팀별 IB 포트 전속 할당
```

### 10.2 Operator 활용

이미 다음 Operator로 운영 자동화:
- NVIDIA Network Operator (IB 스택)
- Kubeflow Training Operator (PyTorchJob)

추가로 만들 수도 있는 것:
- 학습 Job 자동 정리, 비용 관리
- 사내 ML 플랫폼 전용 CRD (Experiment, Model 등)

### 10.3 etcd 운영 주의

- 정기 백업
- 디스크 IO 빠른 SSD
- 큰 ConfigMap/Secret 남용 금지 (etcd 부담)

---

## 11. 트러블슈팅 시나리오

### 11.1 "Pod이 Pending 상태로 멈춤"

```bash
kubectl describe pod mypod
# Events: 
# 0/4 nodes are available: 4 Insufficient nvidia.com/gpu
```

원인 후보:
- 자원 부족 (GPU/CPU/메모리)
- nodeSelector/affinity 매칭 안 됨
- taint에 대한 toleration 없음
- PVC 바인딩 실패
- 이미지 풀 실패

### 11.2 "API Server 응답 느림"

```bash
# etcd 상태
ETCDCTL_API=3 etcdctl endpoint status --cluster -w table

# leader가 자주 바뀌면 → 네트워크/디스크 문제
# DB size 큼 → compact 필요
```

### 11.3 "kubelet이 NotReady"

```bash
# 노드에서
journalctl -u kubelet -n 100

# 흔한 원인:
# - 디스크 가득 (containerd 이미지)
# - containerd 죽음
# - cgroup driver 불일치 (systemd vs cgroupfs)
# - CNI 플러그인 못 찾음
```

---

## 12. 면접 예상 질문

### Q1. "K8s를 한 문장으로 설명한다면?"
> "선언적으로 정의된 desired state를 다양한 컨트롤러들의 reconcile loop로 current state로 수렴시키는 분산 시스템입니다. 모든 상태는 etcd에 저장되고 API Server를 통해서만 접근 가능하며, kubelet이 각 노드에서 컨테이너 실행을 책임집니다."

### Q2. "Pod이 만들어지는 흐름을 처음부터 끝까지 설명해보세요."
> (Pod 생성 9단계 답안)

### Q3. "Operator 패턴을 왜 쓰나요?"
> "K8s의 빌트인 controller pattern(reconcile loop)을 사용자 도메인 객체에 적용하는 것입니다. CRD로 새 객체 타입을 정의하고, 그 객체를 watch하는 컨트롤러가 desired state를 K8s 리소스(Pod, Service 등)로 변환합니다. PyTorchJob처럼 도메인 특화 워크로드를 K8s 네이티브 방식으로 다룰 수 있게 해줍니다."

### Q4. "etcd가 죽으면 어떻게 되나요?"
> "K8s의 모든 상태가 etcd에 있으므로 API Server가 응답 못 합니다. 다만 이미 동작 중인 Pod들은 kubelet이 자체 캐시로 계속 실행합니다. 즉 클러스터는 'frozen' 상태가 됩니다 — 새로 만들거나 바꾸지는 못하지만 기존 워크로드는 동작. 그래서 etcd는 multi-node로 배포하고 정기 백업이 필수입니다."

### Q5. "Admission Controller는 무엇이고 언제 쓰나요?"
> "API Server가 etcd에 객체를 저장하기 직전 단계의 hook입니다. Mutating은 객체를 자동 수정(예: Istio sidecar 주입), Validating은 정책 검증(예: 'GPU Pod은 반드시 toleration 가져야 함')에 사용됩니다. Webhook으로 외부 서비스에 위임할 수 있어 강력한 확장 메커니즘입니다."

### Q6. "kube-scheduler의 동작은?"
> "Filter 단계에서 자원/affinity/taint 등으로 후보 노드를 거르고, Score 단계에서 점수를 매겨 최적 노드를 선택합니다. Pod.spec.nodeName을 업데이트하면 해당 노드의 kubelet이 watch로 감지해 컨테이너를 시작합니다. nodeSelector, affinity, taint/toleration이 스케줄링을 제어하는 주요 메커니즘입니다."

---

## 12.5 연계 문서

- 런타임 하단: [container-runtime-deep-dive.md](container-runtime-deep-dive.md), [../kernel/cgroup-deep-dive.md](../kernel/cgroup-deep-dive.md).
- 인증/인가 상세: [authn-authz-deep-dive.md](authn-authz-deep-dive.md).
- 스케줄러 확장: [multi-tenancy-scheduler-deep-dive.md](multi-tenancy-scheduler-deep-dive.md) (Kueue/Volcano).
- CNI와 Pod 네트워크: [../kernel/cni-kernel-deep-dive.md](../kernel/cni-kernel-deep-dive.md).

---

## 13. 한 줄 요약

> **"K8s 컨트롤 플레인 = etcd(상태) + API Server(게이트웨이) + 수많은 컨트롤러(reconcile loop)다. 사용자가 desired state를 선언하면 컨트롤러들이 informer-workqueue 패턴으로 current state를 desired로 수렴시킨다. CRD + 커스텀 컨트롤러 = Operator로 도메인 워크로드를 네이티브하게 확장한다."**
