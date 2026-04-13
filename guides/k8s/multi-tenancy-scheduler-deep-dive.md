# 멀티테넌시 & 스케줄러 딥다이브 — Kueue / Volcano / ResourceQuota / Kyverno

> **목적**: "GPU 8장을 3팀이 나눠 쓴다"가 어떻게 **K8s scheduler + quota + admission + job queue** 4축으로 풀리는지 설계. 현대차급 대규모(수백~수천 GPU)까지 스케일되는 그림.
> **선행**: [k8s-control-plane-deep-dive.md](k8s-control-plane-deep-dive.md), [security-deep-dive.md](security-deep-dive.md)
> **연결**: [mlops-stack-deep-dive.md](mlops-stack-deep-dive.md), [inference-serving-deep-dive.md](inference-serving-deep-dive.md)

---

## 0. 문제 정의

단일 클러스터에서 해결해야 할 것:

1. **배치 공정성** — 팀 A가 전부 점유해서 팀 B가 굶지 않도록.
2. **Gang Scheduling** — 분산학습 32 GPU를 "전부 또는 아무것도" 스케줄.
3. **우선순위 + 선점** — 프로덕션 Job이 실험 Job을 밀어낼 수 있도록.
4. **할당량 + 빌림** — 팀 quota를 초과하는 여유 자원 임시 빌려쓰기.
5. **정책 강제** — `nvidia.com/gpu` 없이 Pod 만드는 것 차단 등.

기본 K8s scheduler는 (1)(2)(4) 약함. 그래서 **Kueue / Volcano + ResourceQuota + Kyverno** 조합.

---

## 1. 기본 스케줄러 한계

### 1.1 Default kube-scheduler

- **Pod 단위**로 스케줄. 32개 Pod 중 31개만 fit되면 31개는 떠버림. Gang 의미 없음.
- **우선순위** — PriorityClass는 있지만 기본은 FIFO + first-fit.
- **Quota** — `ResourceQuota` 는 namespace 합계만 막음. 팀 간 공유/빌림 불가.

### 1.2 왜 이게 문제?

PyTorchJob `Worker 4` 중 2개만 뜨고 2개 Pending → 이미 뜬 2개는 **GPU를 잡은 채 무작정 대기**. 다른 Job도 시작 못하는 **자원 데드락**.

---

## 2. PriorityClass + Preemption (K8s 내장)

### 2.1 PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: prod-high
value: 100000
preemptionPolicy: PreemptLowerPriority
globalDefault: false
description: "Production inference workloads"
```

Pod에 `priorityClassName: prod-high` → scheduler가 해당 Pod을 **다른 Pod 제치고** 우선 배치. 리소스 부족하면 **lower priority Pod을 Evict** 하고 자리 확보.

### 2.2 PreemptionPolicy

- `PreemptLowerPriority` (default) — 자리 부족하면 낮은 Pod evict.
- `Never` — evict 안 함, 기다리기만.

**주의**: evict는 SIGTERM. **학습 Job이 checkpoint 없으면 손실** — Kubeflow PyTorchJob은 기본적으로 evict에 약함. priority 설계 시 학습/서빙 분리 핵심.

---

## 3. ResourceQuota + LimitRange

### 3.1 ResourceQuota (namespace 한도)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-slm-quota
  namespace: slm
spec:
  hard:
    requests.nvidia.com/gpu: "16"
    requests.cpu: "200"
    requests.memory: "2Ti"
    persistentvolumeclaims: "50"
    requests.storage: "10Ti"
```

**admission 시 합계 체크**. 초과하면 Pod create 거부.

**주의**: GPU는 `limits`와 `requests`가 같아야 함 (extended resource 규칙). `limits.nvidia.com/gpu`만 명시하면 request 자동 동일.

### 3.2 LimitRange (default + 상한)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      memory: "64Gi"
```

**역할**: 사용자가 resources 안 써도 기본값 주입. 과도한 요청 방지.

### 3.3 ResourceQuota의 한계

- namespace 단위만. **팀 간 빌림 불가**.
- scope가 admission 시점만 — 이미 뜬 Pod은 건드리지 않음.
- 우선순위 개념 없음.

→ Kueue 등장.

---

## 4. Kueue (Job Queueing)

### 4.1 개념

**Job을 submission 시점에 suspend** → Kueue가 quota 확인 → 자리 있으면 `.spec.suspend=false` 로 풀어줌.

**장점**:
- **gang admission** — Job 전체가 fit되어야 시작.
- **LocalQueue / ClusterQueue** 계층 — 팀 ↔ 전사.
- **Cohort + borrowing** — 팀 간 여유 빌림.
- **PriorityClass 연동**.
- Kubeflow PyTorchJob, JobSet, RayJob, MPIJob 등 지원.

### 4.2 리소스 계층

```
ResourceFlavor             ← 하드웨어 종류 (H100, A100, CPU-only)
       ▲
ClusterQueue               ← 전사 자원 풀
  quotas: { H100: 32 }
       ▲
LocalQueue (namespace)     ← 팀 큐
       ▲
Workload (Job/PyTorchJob)  ← 사용자 제출
```

### 4.3 ClusterQueue 예시

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: cq-h100-shared
spec:
  namespaceSelector: {}          # 모든 ns 허용
  resourceGroups:
  - coveredResources: ["nvidia.com/gpu"]
    flavors:
    - name: "h100"
      resources:
      - name: "nvidia.com/gpu"
        nominalQuota: 16          # 기본 할당
        borrowingLimit: 16        # 빌릴 수 있는 최대
  cohort: "research"              # 같은 cohort 내 빌림 가능
```

### 4.4 Cohort + Borrowing

- 같은 `cohort`에 속한 ClusterQueue 끼리 **사용 안 하는 quota를 빌려줌**.
- `borrowingLimit`: 빌릴 수 있는 최대량.
- **선점 대상**: 빌려간 쪽의 Workload는 원래 주인이 요구하면 evict 가능 (`preemption.reclaimWithinCohort`).

**예시**:
- ClusterQueue A: nominal 16, 사용 4.
- ClusterQueue B: nominal 16, 사용 28 (A에서 12 빌림).
- A가 갑자기 16 요구 → Kueue가 B의 빌린 Workload를 preempt.

### 4.5 LocalQueue

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  namespace: slm
  name: slm-h100
spec:
  clusterQueue: cq-h100-shared
```

사용자 PyTorchJob에 label:
```yaml
metadata:
  labels:
    kueue.x-k8s.io/queue-name: slm-h100
```

Kueue가 이 라벨 보고 해당 ClusterQueue의 quota와 대조.

### 4.6 Gang Admission 보장

Kueue는 Workload 단위로 quota 체크 → **Job 전체 fit 되기 전엔 단 하나의 Pod도 안 뜸**. PyTorchJob의 부분 실행 + 데드락 문제 해결.

---

## 5. Volcano (Gang Scheduler)

### 5.1 Kueue와 뭐가 다른가

Kueue는 **queue/quota 관리자** — 결정 후 실제 스케줄은 kube-scheduler.
Volcano는 **대체 scheduler** — kube-scheduler 대신 직접 gang/fair/SLA 정책 수행.

| 축 | Kueue | Volcano |
|----|-------|---------|
| 레이어 | Admission+Queue | Scheduler |
| Gang | gang admission (suspend 제어) | gang scheduling (Pod 배치 단계) |
| Quota | ClusterQueue (borrow 지원) | Queue + reclaim |
| 정책 | 간단, Kubernetes native | plugin 다양 (DRF, binpack, preempt) |
| 성숙도 | CNCF sandbox→incubating | CNCF incubating |

**최근 경향**: Kueue가 **kube-scheduler 호환**을 유지하며 기능을 넓히는 중. 둘 다 쓰거나, Kueue만 쓰는 조합이 단순.

### 5.2 Volcano PodGroup

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: job-1
spec:
  minMember: 4
  queue: research
  priorityClassName: medium
```

Job의 모든 Pod이 이 PodGroup 참조. `minMember` 충족될 때까지 **scheduling cycle에서 전체 유보**.

### 5.3 Volcano Plugin 조합

- `gang` — minMember 보장
- `predicates` — nodeAffinity 등
- `priority` — PriorityClass
- `proportion` — queue 간 비례 배분
- `drf` (Dominant Resource Fairness) — 다차원 공정성
- `binpack` — 자원 뭉치기 (GPU fragmentation 방지)
- `reclaim` — 빌려간 자원 회수

---

## 6. Kyverno (정책 강제)

### 6.1 역할

K8s native **Policy Engine**. ClusterPolicy CR로 admission 시점 규칙.

세 가지 mode:
- **validate** — 규칙 어기면 거부.
- **mutate** — 자동 보강 (patch 적용).
- **generate** — 다른 리소스 자동 생성.

### 6.2 GPU 관련 예시

**정책 1: GPU 요청 Pod은 반드시 toleration + nodeSelector**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-gpu-node-scheduling
spec:
  validationFailureAction: Enforce
  rules:
  - name: gpu-scheduling
    match:
      any:
      - resources:
          kinds: ["Pod"]
    preconditions:
      all:
      - key: "{{ request.object.spec.containers[].resources.limits.\"nvidia.com/gpu\" || `0` }}"
        operator: GreaterThan
        value: 0
    validate:
      message: "GPU Pod must have h100 nodeSelector and toleration"
      pattern:
        spec:
          nodeSelector:
            accelerator: h100
          tolerations:
          - key: nvidia.com/gpu
            operator: Exists
```

**정책 2: latest 이미지 금지**

```yaml
spec:
  rules:
  - name: disallow-latest-tag
    match:
      any:
      - resources:
          kinds: ["Pod"]
    validate:
      message: "An image tag is required; :latest is forbidden"
      pattern:
        spec:
          containers:
          - image: "!*:latest"
```

**정책 3: namespace 생성 시 기본 ResourceQuota 자동 generate**

```yaml
spec:
  rules:
  - name: default-quota
    match:
      any:
      - resources:
          kinds: ["Namespace"]
    generate:
      kind: ResourceQuota
      name: default-quota
      namespace: "{{request.object.metadata.name}}"
      data:
        spec:
          hard:
            requests.nvidia.com/gpu: "4"
```

### 6.3 회사 정책 예 (추정/설계)

- PyTorchJob 생성 시 **NodeSelector** 자동 주입 (mutate).
- prod namespace에 **`imagePullPolicy: IfNotPresent` + 이미지 레지스트리 허용 리스트** (validate).
- Kubeflow Profile 생성 시 default ResourceQuota + LimitRange (generate).

### 6.4 Validating vs Mutating Admission 체인

kube-apiserver admission 순서:
```
Mutating Admission → Validating Admission → etcd
```

Mutating에서 patch → Validating에서 검증. Kyverno는 둘 다 지원.

---

## 7. PSA (Pod Security Admission)

### 7.1 레벨

- **privileged** — 제한 없음
- **baseline** — 일반 워크로드 기본 보안
- **restricted** — 엄격 (runAsNonRoot, no privilege escalation, readOnlyRootFS 등)

### 7.2 namespace 라벨

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: serving-prod
  labels:
    pod-security.kubernetes.io/enforce: "restricted"
    pod-security.kubernetes.io/audit: "restricted"
    pod-security.kubernetes.io/warn: "restricted"
```

### 7.3 GPU Pod 주의

- `nvidia/cuda` 이미지는 root가 기본 → `restricted`에선 **runAsNonRoot** 위반.
- 해결: `runAsUser: 1000` 추가, 이미지 레이어에 nonroot user 세팅.
- 또는 `baseline` 적용.

---

## 8. 장애/시나리오

### 8.1 "32 GPU PyTorchJob이 Pending인 채 다른 Job들이 2개씩 뜸"

- Kueue 없이 제출됨 → partial scheduling으로 다른 팀 Job이 조금씩 먹음 → 32개 gang 영원히 대기.
- **해결**: 해당 namespace에 Kueue LocalQueue 연결 + PyTorchJob label.

### 8.2 "prod 서빙이 학습 때문에 느려짐"

- 같은 노드에 학습/서빙 공존.
- **해결**:
  - Node taint: `nvidia.com/workload=training:NoSchedule` / `serving:NoSchedule`.
  - Kyverno로 namespace별 toleration 자동 주입.

### 8.3 "팀 A quota 남는데 팀 B가 못 빌린다"

- Cohort 설정 안 됨. 같은 cohort로 묶으면 borrow 가능.

### 8.4 "PriorityClass evict로 학습이 checkpoint 없이 죽었다"

- PreemptLowerPriority 기본. 대책:
  - 학습용 PriorityClass를 prod과 같게 높임 + Kyverno로 강제.
  - 학습은 **별도 노드풀 + NoSchedule taint** → preempt 충돌 원천 차단.

---

## 9. 설계 템플릿 (팀 3개 + 환경 2개)

```
ResourceFlavor:
  - h100 (DGX 노드들)
  - cpu-only (일반 노드)

ClusterQueue:
  - cq-research (cohort=research, 16 GPU nominal, borrow 16)
  - cq-engineering (cohort=research, 12 GPU nominal, borrow 12)
  - cq-prod (cohort=prod, 4 GPU nominal, borrow 0)

LocalQueue:
  - ns: slm       → cq-research
  - ns: foundation → cq-research
  - ns: data-eng  → cq-engineering
  - ns: serving-prod → cq-prod

Kyverno policies:
  - mutate: GPU Pod에 nodeSelector + toleration 주입
  - validate: serving-prod ns는 image allow-list
  - generate: Profile 생성 시 default ResourceQuota

PriorityClasses:
  - prod-serving (100000) — 서빙 전용
  - research-high (50000) — 팀 리더 승인 학습
  - research-normal (10000) — 일반 학습
```

---

## 10. 면접 Q&A

### Q1 (기초). ResourceQuota 만으로 멀티팀 운영이 왜 부족한가?
A. (1) 한 namespace 합계만. (2) 팀 간 여유 **빌림 불가**. (3) Job 단위 gang 없음 — 부분 스케줄로 데드락. (4) 우선순위 없음. Kueue가 이 4개를 보강.

### Q2. Kueue와 Volcano, 같이 쓸 수 있나? 우리는 뭘 쓴다면?
A. 동시 사용 가능하지만 복잡. Kueue는 admission/queue 레이어, Volcano는 scheduler. 둘 다 쓰면 policy 충돌 디버깅 어려움. **Kubernetes native + kube-scheduler 유지** 원칙이면 Kueue만. 복잡 스케줄링 정책(DRF, binpack 등) 필요하면 Volcano 단독.

### Q3. Gang admission vs Gang scheduling 차이?
A. Admission은 **Job 제출 시점에 전체 fit되면 풀어줌** (Kueue). Scheduling은 **scheduler cycle에서 전체 Pod을 동시 배치** (Volcano). 결과는 비슷하지만, gang admission은 Job이 suspend 상태로 대기하므로 kube-scheduler 부하가 없음.

### Q4. Kyverno vs OPA Gatekeeper?
A. 둘 다 policy engine. Kyverno는 **K8s native YAML** — CRD로 정책, Rego 없음. Gatekeeper는 Rego 기반(OPA 재사용). **사내에 Rego 경험 있으면 Gatekeeper, K8s-only면 Kyverno 편함**. Kyverno는 mutate + generate까지 기본 지원 (Gatekeeper는 mutation은 별도 모듈).

### Q5. PreemptLowerPriority로 학습 Job 날아가는 것 어떻게?
A. 방법 여러개: (1) 학습 노드풀 taint 분리. (2) 학습 PriorityClass 값을 서빙과 비슷하게. (3) PyTorchJob에 **주기적 체크포인트 + elastic training**으로 재개 가능. (4) 서빙 Pod의 preemptionPolicy를 `Never`로.

### Q6. Cohort borrowing 설계 시 함정?
A. 빌려간 Pod은 원주인 요구 시 preempt되어 재시작. **stateful Pod(학습)에 치명적**. 대책: 빌림은 배치 Job에만, 서빙은 borrow 금지. `borrowingLimit: 0` 으로 명시.

### Q7. PSA `restricted`를 GPU Pod에 적용하려면?
A. (1) 이미지에 nonroot user 설정 (UID 1000 등). (2) `securityContext.runAsNonRoot: true`. (3) `readOnlyRootFilesystem: true` — 쓰기 경로는 emptyDir 마운트. (4) `capabilities: drop: [ALL]`. 커널 디바이스(`/dev/nvidia*`) 접근은 허용해야 하므로 `privileged`는 불가능하지만 GPU device plugin의 host mount가 작동하도록 configure.

### Q8. 팀 quota 16 GPU, 실제 사용 12. 나머지 4는 놀고 다른 팀이 아쉬워함. 설계?
A. 해당 팀 ClusterQueue를 **cohort 공유** + `borrowingLimit: 4`. 다른 팀 Workload가 cohort 한도 내에서 빌려감. 원팀이 16 요구하면 Kueue가 빌려간 쪽 Workload를 preempt하여 회수.

### Q9. Kyverno 정책이 갑자기 전사 Pod을 거부하는 사고. 방어책?
A. (1) 정책을 **`validationFailureAction: Audit`** 로 먼저 배포 → 감사 로그만 남기고 실제 거부 안 함. (2) 지표 확인 후 `Enforce`로 승격. (3) CI에 `kyverno test` 포함 — 정책 rollout 전 예제 Pod로 동작 확인. (4) Kyverno 자체가 죽으면 admission webhook `failurePolicy: Ignore` 설정으로 klaster 락 방지.

### Q10. 노드 20대, GPU 160장. 사이즈 커지면 어디가 먼저 아프나?
A. (1) etcd — Pod + Workload 수 증가, compaction 중요. (2) scheduler — 큰 gang Job은 decision 시간 증가. (3) Kyverno — admission 체인 지연. (4) Kueue controller — Workload 수 많으면 reconcile 병목. 대응: etcd shard/성능 튜닝, scheduler `percentageOfNodesToScore` 조정, Kyverno replicas ↑.

---

## 11. 체크리스트

- [ ] ResourceQuota 한계 4가지
- [ ] Kueue ClusterQueue / LocalQueue / Workload 계층
- [ ] Cohort borrowing + preemption 메커니즘
- [ ] Volcano PodGroup + minMember
- [ ] Kyverno mutate/validate/generate 구분
- [ ] Mutating → Validating admission 순서
- [ ] PSA 3레벨 + GPU 이미지 주의점
- [ ] PriorityClass + PreemptionPolicy 학습 보호 패턴
- [ ] 팀 3개 설계 템플릿 (§9) 말로 재현

---

## 12. 참고

- Kueue: https://kueue.sigs.k8s.io/
- Volcano: https://volcano.sh/
- Kyverno: https://kyverno.io/
- 관련: [mlops-stack-deep-dive.md](mlops-stack-deep-dive.md), [security-deep-dive.md](security-deep-dive.md), [inference-serving-deep-dive.md](inference-serving-deep-dive.md)
