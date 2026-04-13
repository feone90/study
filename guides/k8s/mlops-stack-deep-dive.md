# MLOps 스택 딥다이브 — Kubeflow / MPI / MLflow / Airflow 내부 구조

> **목적**: "Kubeflow 쓴다"라는 말 뒤에 숨은 **CRD 모양, reconcile 루프, 인증 경로, mutating webhook, 스토리지 경로**까지 설명할 수 있게 만든다. 회사 레포(`ml-platform/thirdparty/dn-manifest/mlops/`)의 실물을 1차 근거로 사용.
> **읽기 순서 선행**: [../k8s/k8s-control-plane-deep-dive.md](k8s-control-plane-deep-dive.md), [../integration/dgx-ib-multinode-training-guide.md](../integration/dgx-ib-multinode-training-guide.md)
> **후속 딥다이브**: [inference-serving-deep-dive.md](inference-serving-deep-dive.md), [multi-tenancy-scheduler-deep-dive.md](multi-tenancy-scheduler-deep-dive.md)

---

## 0. 큰 그림

사용자 관점:

```
[Data Scientist]
     │ 1) Notebook 띄우기         ┐
     │ 2) 분산학습 Job 제출        │
     │ 3) 실험 메트릭/모델 저장     ├─ Kubeflow + MLflow
     │ 4) 파이프라인 예약 실행     ┘
     │ 5) 모델 배포                 ─ KServe (다음 문서)
```

플랫폼 관점:

```
User → Istio Ingress (OIDC via Keycloak+Dex)
     → Kubeflow Central Dashboard
     → Notebook Controller / Training Operator / Pipelines
     → Pod 생성 (GPU, IB, PVC)
     → MLflow Tracking (Ceph RGW S3)
     → Airflow DAG (스케줄 실행)
```

각 컴포넌트는 **하나의 Operator + CRD**로 동작. "매니페스트 올리면 Pod 뜨는 마법"이 아니라 **reconcile 루프 + RBAC + webhook 체인**.

---

## 1. Kubeflow Training Operator

### 1.1 역할

여러 프레임워크의 분산 학습을 **K8s native CRD**로 표현. PyTorch/TF/MXNet/XGBoost/PaddlePaddle. 회사에선 `PyTorchJob`만 씀.

**단일 Operator 프로세스가 여러 CRD watch** (v1.7+ 통합). 이전 버전은 `pytorch-operator` / `tf-operator`가 각자 배포됐지만 현재는 `training-operator` 하나가 전부 처리.

### 1.2 CRD 모양 (PyTorchJob)

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: pytorch-dist-mnist-nccl-cwkim
  namespace: slm
spec:
  elasticPolicy:                 # PyTorch Elastic (torchrun) 지원
    rdzvBackend: c10d
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            k8s.v1.cni.cncf.io/networks: ib-ibp79s0-conf-slm   # Multus NAD
        spec:
          containers:
          - name: pytorch
            image: ...
            env:
            - name: NCCL_IB_HCA
              value: mlx5_3
            - name: NCCL_IB_GID_INDEX
              value: "3"
            resources:
              limits:
                nvidia.com/gpu: 8
                rdma/hca_shared_devices_a: 1
    Worker:
      replicas: 3
      # (동일 template)
```

**핵심 3가지**:

1. `pytorchReplicaSpecs.Master/Worker` — 역할별 Pod 템플릿.
2. `elasticPolicy` — torchrun의 rendezvous 설정. 없으면 정적 world_size.
3. **Pod annotation `k8s.v1.cni.cncf.io/networks`** — Multus가 Secondary NIC(IB) 붙임. Training Operator가 신경 안 씀. 사용자가 템플릿에 직접 써야 한다.

### 1.3 Reconcile 루프

```
[User] kubectl apply pytorchjob.yaml
   │
   ▼
apiserver → etcd 저장
   │
   ▼
Training Operator (watch on PyTorchJob CRD)
   │ 1. JobController.Reconcile()
   │ 2. 필요한 Pod spec 생성 (Master 1 + Worker N)
   │    - env 자동 주입: MASTER_ADDR, MASTER_PORT, WORLD_SIZE, RANK
   │    - Pod label: training.kubeflow.org/job-name, replica-type
   │ 3. Service 생성 (Master의 headless service, rendezvous용)
   │ 4. Pod 상태 집계 → JobStatus 업데이트
   ▼
kube-scheduler → Pod binding
   │
   ▼
kubelet → containerd → runc → torchrun
```

**자동 주입 env** (면접 빈출):

| env | 값 | 누가 넣나 |
|-----|-----|----------|
| `MASTER_ADDR` | Master Pod의 headless service DNS | Training Operator |
| `MASTER_PORT` | 기본 `23456` | Training Operator |
| `WORLD_SIZE` | replicas 합계 × `nvidia.com/gpu` | Training Operator |
| `RANK` | Pod의 인덱스 × GPU 수 | Training Operator |
| `NCCL_IB_HCA` 등 | 사용자가 spec.env에 명시 | 사용자 |

### 1.4 Worker 확장(2→3→4) 시 주의

회사 레포에 `...worker2.yaml`, `...worker3.yaml`이 존재하는 이유:

- `replicas: N` 늘리면 Master가 rendezvous를 기다림. **정적 world_size라서 Pod 수 맞아야 시작**.
- IB NAD는 namespace × NIC 전속 매핑. Worker 늘리려면 **해당 namespace가 점유한 IB 포트 수** 확인.
- `rdma/hca_shared_devices_a` 리소스가 부족하면 `Pending` 영원히.

### 1.5 장애 패턴

1. **Master만 Pending, Worker 뜸** → Master는 보통 더 큰 리소스 요구. Quota/nodeSelector 확인.
2. **Worker 1개 OOMKilled → 전체 Job 실패** → `restartPolicy: OnFailure` + `backoffLimit` 조정.
3. **Pod 다 떴는데 `NCCL timeout`** → MASTER_ADDR DNS 해결 안 됨. headless service 확인.
4. **`rdma: insufficient`** → RDMA Shared Device Plugin이 해당 노드에서 안 뜸.

---

## 2. MPI Operator (v2beta1)

### 2.1 PyTorchJob과의 차이

| 축 | PyTorchJob | MPIJob |
|----|------------|--------|
| 런처 모델 | 각 Pod에서 torchrun가 rendezvous | **Launcher 1 + Worker N** 구조 |
| 통신 시작 | NCCL P2P | **mpirun/hydra**가 SSH로 Worker 킥 |
| 적합 워크로드 | PyTorch DDP, FSDP | Horovod, OpenMPI 기반 (Tensorflow 분산, HPC 응용) |
| 인증 | 없음 (K8s service DNS) | **SSH 키 생성** — ConfigMap/Secret로 주입 |

### 2.2 CRD 모양

```yaml
apiVersion: kubeflow.org/v2beta1
kind: MPIJob
metadata:
  name: horovod-test
spec:
  slotsPerWorker: 8           # worker당 GPU 수
  runPolicy:
    cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
        spec:
          containers:
          - name: launcher
            command:
            - mpirun
            - --allow-run-as-root
            - -np
            - "32"            # 총 process
            - -bind-to
            - none
            - python
            - train.py
    Worker:
      replicas: 4             # slotsPerWorker × replicas = np
      # (sshd 떠있는 이미지)
```

### 2.3 SSH Key 흐름

1. MPI Operator가 **Launcher 전용 Secret** 생성 (id_rsa + known_hosts 템플릿).
2. **Worker에 authorized_keys 주입** (ConfigMap).
3. Worker Pod는 sshd를 22포트로 띄움.
4. Launcher가 `mpirun` 실행 → Hydra가 Worker에 SSH → 프로세스 fork.

**면접 포인트**: "왜 SSH?" → MPI 표준이 그렇게 정의되어 있고 Horovod/OpenMPI가 Hydra/ORTE 런처를 쓰기 때문. K8s-native는 아니지만 기존 HPC 자산을 그대로 활용할 수 있다는 트레이드오프.

### 2.4 회사에서 PyTorchJob을 주로 쓰는 이유

- PyTorch DDP는 NCCL backend를 직접 쓰므로 MPI 필요 없음.
- SSH key 관리, sshd 프로세스 추가, 보안 감사 대상 늘어남.
- LLM/SLM 팀 프레임워크가 PyTorch 중심.

→ **MPIJob은 Horovod/TF 레거시 요청이 오면 옵션으로 제공**하는 정도. Training Operator가 v1.7+부터 **둘 다 동일 컨트롤러**라 운영 부담 적음.

---

## 3. Kubeflow Central Dashboard + Profile

### 3.1 Profile이란

사용자별 **namespace + ServiceAccount + RBAC + 기본 리소스** 번들. Admin이 `kubectl apply -f profile.yaml` 하면:

```yaml
apiVersion: kubeflow.org/v1
kind: Profile
metadata:
  name: cwkim
spec:
  owner:
    kind: User
    name: cwkim@dnotitia.com
```

Profile Controller가:

1. namespace `cwkim` 생성 + label `app.kubernetes.io/part-of=kubeflow-profile`
2. RoleBinding: `namespaceAdmin` → user
3. ServiceAccount `default-editor`, `default-viewer`
4. **Istio AuthorizationPolicy** — 이 namespace는 해당 user만 접근
5. default PodDefault (공통 볼륨 마운트 등)

### 3.2 Mutating Admission Webhook (회사 커스텀)

회사 레포: `dn-manifest/mlops/kubeflow/mutating-admission-webhook/`.

**역할**: Profile 생성 시 혹은 Pod 생성 시 **자동으로 주입해야 할 것들**을 가로채서 inject.

**일반적인 주입 내용**:
- 공용 Ceph RGW credentials(Secret) 볼륨 마운트
- MLflow tracking URI env
- NodeSelector (GPU 노드 제한)
- Default Tolerations (GPU taint)

**흐름**:
```
kubectl apply pod.yaml
  ↓
apiserver MutatingAdmissionWebhook 호출
  ↓
webhook (Python Flask app.py) → JSONPatch 반환
  ↓
apiserver: Pod spec 수정본 저장
  ↓
scheduler → binding
```

**왜 ValidatingWebhook 아닌 MutatingWebhook?**: "거부"가 아니라 "보강"이 목적. 사용자가 잊은 것을 자동으로 채움.

### 3.3 PodDefault CRD

```yaml
apiVersion: kubeflow.org/v1alpha1
kind: PodDefault
metadata:
  name: add-mlflow-config
  namespace: cwkim
spec:
  selector:
    matchLabels:
      mlflow: "true"
  desc: "Attach MLflow tracking URI"
  env:
  - name: MLFLOW_TRACKING_URI
    value: http://mlflow.mlflow.svc:5000
  - name: MLFLOW_S3_ENDPOINT_URL
    value: http://ceph-rgw.rook-ceph.svc
```

Notebook Controller가 `poddefaults.kubeflow.org` label 매칭 검사 → Notebook에 라벨 붙으면 자동 주입.

---

## 4. MLflow (Tracking + Registry)

### 4.1 구성

```
                  ┌────────────────────┐
   SDK (python) ──▶ mlflow server (REST API :5000)
                  │   - tracking       │
                  │   - model registry │
                  └─────┬──────────────┘
                        │
      ┌─────────────────┼─────────────┐
      ▼                                ▼
  PostgreSQL                     S3-compatible
  (metadata: runs, params,       (artifacts: model
   metrics, tags)                 pickles, logs)
```

### 4.2 회사 구성

- DB: Postgres (in-cluster, `mlflow-postgresql`)
- Artifact: **Ceph RGW SSD pool** (S3 API 호환)
- 엔드포인트: `http://ceph-rgw.rook-ceph.svc` + bucket `mlflow-artifacts`
- 인증: Ceph RGW의 AWS-S3 style access key → Kubernetes Secret → Pod env

### 4.3 MinIO → Ceph RGW 마이그레이션 (STAR-3)

**왜 바꿨나**:
- MinIO는 SSD 전용 bucket을 별도 스토리지 풀로 관리할 수 없음.
- Ceph RGW는 **placement target**으로 SSD/HDD pool 분리 → artifact는 SSD, 백업은 HDD.
- 동일 Ceph 인프라에 통합 → MON/OSD 모니터링 일원화.

**절차** (Confluence 95584383):
1. 기존 MinIO bucket → rclone으로 RGW bucket 복사.
2. `mlflow server --default-artifact-root s3://mlflow-artifacts/` 로 환경변수 업데이트.
3. 기존 run의 artifact 경로는 DB에 절대경로로 저장되므로 **일괄 UPDATE**:
   ```sql
   UPDATE experiments SET artifact_location = REPLACE(artifact_location, 's3://minio/', 's3://rgw/');
   UPDATE runs SET artifact_uri = REPLACE(artifact_uri, 's3://minio/', 's3://rgw/');
   ```
4. HDD pool에 백업 bucket 별도 구성.

### 4.4 주의: S3 endpoint URL

AWS 공식 S3와 Ceph RGW의 차이:
- `MLFLOW_S3_ENDPOINT_URL` 반드시 설정 (`boto3`가 기본 AWS로 감).
- **path-style addressing** 필요: `AWS_S3_ADDRESSING_STYLE=path`. virtual-host style이면 DNS 변환 실패.
- HTTPS가 아니면 `AWS_S3_USE_SSL=false`.

---

## 5. Airflow

### 5.1 구성 (CeleryExecutor or KubernetesExecutor)

회사는 KubernetesExecutor 사용. DAG 실행 단위마다 **worker Pod 스폰**.

```
scheduler Pod  ← DAG 파싱 + 스케줄링
webserver Pod  ← UI (OIDC via Keycloak)
triggerer Pod  ← Deferrable operator
worker Pod     ← Task 1개당 1 Pod (KubernetesExecutor)
```

DAG 코드: GitSync sidecar로 git 저장소에서 주기 pull. Airflow main/dev/airflow 3개 환경 분리.

### 5.2 DB 스키마 + Deferred

- DB: Postgres (`airflow-postgresql`)
- 핵심 테이블: `dag`, `dag_run`, `task_instance`, `xcom`
- Deferrable operator: scheduler가 Sensor를 long-poll에서 빼서 **triggerer**에 위임. DB 부하 ↓.

### 5.3 Airflow 복구 (STAR-4, ML-30)

**증상**: Ceph 장애 여파로 Airflow webserver 500. 로그:
```
File ".../cachetools/_cachedmethod.py", line 56, in wrapper
File ".../fab_auth_manager.py", line 212, in deserialize_user
KeyError: 'user_id'
```

**원인 체인**:
1. Ceph I/O hang → DB(Postgres) PVC 응답 지연 → 일부 row 손상 가능성.
2. FAB(Flask-AppBuilder) auth manager가 session 역직렬화 중 `cachetools` TTL cache에서 stale user 읽음.
3. `airflow-webserver-secret`의 FERNET_KEY 불일치로 기존 session 복호화 실패.

**조치**:
1. `airflow-webserver` Pod 재시작 → cachetools 캐시 초기화.
2. 증상 지속 → Airflow **v3.1.7 업그레이드** (cachetools 의존성 제거한 auth manager).
3. health-monitor CronJob 추가: 5분마다 `/health` probe, 3회 실패 시 alert.
4. 최후의 수단 메모: *"이래도 안되면 모든 데이터 날리고 새롭게 설치해보아야할듯"* (ML-30).

**면접 꼬리질문 대비**: "왜 전체 재설치가 아니라 업그레이드를 택했나?" → DAG 히스토리/XCom을 잃으면 과거 실험 추적 불가. 업그레이드로 해결 가능성을 먼저 검증하는 게 비용 대비 효율.

### 5.4 Airflow + MLflow 통합 패턴

```
Airflow DAG
  └─ PythonOperator: train() 호출
        ├─ mlflow.start_run()
        ├─ model fit + log_params/metrics
        ├─ mlflow.log_model() → RGW
        └─ mlflow.register_model() → registry
  └─ PythonOperator: promote (stage → production)
```

Airflow는 스케줄링/의존성, MLflow는 실험 추적/모델 카탈로그. 둘은 **서로 모름**. 통합은 DAG 코드에서.

---

## 6. Harbor (컨테이너 레지스트리)

### 6.1 역할

- 사내 이미지 저장소. 외부 pull 캐시 프록시.
- `harbor-ext` namespace (레포 외부 접근 분리).
- 인증: Keycloak OIDC (Harbor가 OIDC provider로 Keycloak 등록).

### 6.2 Ceph RGW 연동

이미지 레이어 저장을 **Ceph RGW SSD pool**로. Harbor의 `registry` 컴포넌트가 S3 driver 사용.

```yaml
# Harbor values.yaml (발췌)
persistence:
  imageChartStorage:
    type: s3
    s3:
      bucket: harbor
      region: us-east-1
      regionendpoint: http://ceph-rgw.rook-ceph.svc
      accesskey: ...
      secretkey: ...
```

**Confluence 95912565** 마이그레이션 이력:
- 기존 내부 PVC → RGW SSD pool로 이관.
- Replication 기능으로 기존 이미지 복제(다운타임 최소화).
- 끝나고 원본 PVC 삭제.

### 6.3 Harbor CLI Secret

OIDC 사용자는 UI 로그인만 됨. `docker login`은 일반 패스워드 불가 → **CLI Secret** 발급받아 사용 (Harbor UI에서 생성).

---

## 7. Kubeflow Pipelines (KFP)

### 7.1 역할

Airflow와 비슷하지만 **ML 특화**. 각 Step은 Pod, 산출물은 Artifact로 추적.

컴포넌트:
- `ml-pipeline-apiserver` — API
- `ml-pipeline-persistenceagent` — run 상태 DB 기록
- `ml-pipeline-scheduledworkflow` — CronWorkflow
- `argo-workflows-controller` — 실제 Step 실행기

### 7.2 Airflow와 중첩되면?

- KFP: ML 실험 DAG, 단기. Artifact가 파이프라인 내부 리소스.
- Airflow: 광범위한 ETL, 장기 스케줄. 데이터엔지니어링 도구.

회사에선 **Airflow 위주**, KFP는 선택적으로 사용. 둘 다 운영하면 DAG 정의가 분산되므로 한쪽으로 몰아라.

---

## 8. 회사 특화 포인트 요약

| 영역 | 일반 | 이 회사 구성 |
|------|------|--------------|
| Training Operator 버전 | 1.5+ | **v1.7+**, 단일 컨트롤러 |
| 분산 학습 | PyTorchJob 또는 MPIJob | **PyTorchJob 전용**, MPI는 레거시 요청 대응 |
| MLflow artifact | MinIO/NFS | **Ceph RGW SSD pool** (마이그레이션 완료) |
| Airflow executor | Celery | **KubernetesExecutor**, DAG은 GitSync |
| Airflow 환경 분리 | 단일 | **main/dev/airflow 3개** namespace |
| Kubeflow 인증 | Dex만 | **Keycloak → Dex → Istio AuthService** |
| Pod mutation | 없음 | **커스텀 Mutating Webhook** (NodeSelector, MLflow env) |
| Harbor 저장소 | PVC | **Ceph RGW SSD**, OIDC |

---

## 9. 실습 (결과 해석 포함)

### 9.1 PyTorchJob 실행

```bash
kubectl apply -f multi-node-distributed-training-nccl-ib-mnist-sandbox.yaml
kubectl -n slm get pytorchjob pytorch-dist-mnist-nccl-cwkim -w
```

**봐야 할 것**:
- `Status: Running` 가기까지 Master → Worker 순서.
- `kubectl -n slm get pods -l training.kubeflow.org/job-name=pytorch-dist-mnist-nccl-cwkim` — Master 1 + Worker N.
- Master Pod 로그 초반 `NCCL INFO` 에 `NET/IB : Using [0]mlx5_3:1/IB` 보이면 IB 사용 중.

### 9.2 Training Operator 내부 로그

```bash
kubectl -n kubeflow logs deploy/training-operator | grep -i pytorchjob
```

**핵심 라인**:
- `Reconciling PyTorchJob`  — reconcile 시작
- `Creating pod for PyTorchJob` — Pod 생성
- `PyTorchJob is running` — 상태 전이

### 9.3 Profile + PodDefault 검증

```bash
kubectl get profile cwkim -o yaml
kubectl get poddefaults -n cwkim
kubectl get rolebindings -n cwkim
```

**기대 출력**:
- Profile 생성 시 Role: `namespaceAdmin`.
- PodDefault로 `add-mlflow-config` 존재.
- Istio `AuthorizationPolicy`가 해당 namespace에 자동 생성.

---

## 10. 면접 Q&A

### Q1 (기초). PyTorchJob은 왜 K8s Deployment가 아니라 CRD로 따로 만들어졌나?
A. Deployment는 "동일한 replicas가 전부 같은 역할"을 가정. 분산 학습은 **Master/Worker 역할 분리 + world_size 인지 + failure semantics**가 필요. 일반 Deployment의 rolling update는 분산 학습에 맞지 않음. CRD + 컨트롤러로 별도 reconcile 루프를 둠.

### Q1-꼬리. Master가 죽으면?
A. `restartPolicy: OnFailure`면 Operator가 Master를 재생성. 하지만 분산 학습의 rendezvous는 "모든 Pod가 처음부터 다시" 필요 → **사실상 Worker도 재시작** 해야 세션 복구. 체크포인트에서 재개하는 게 현실적.

### Q2. MPIJob은 왜 SSH를 쓰나? K8s 답지 않다는 비판은?
A. MPI(Horovod/OpenMPI)의 런처 Hydra/ORTE가 SSH/RSH 전제. K8s-native 통신(gRPC, Service DNS)로 대체하려면 프레임워크 자체 수정 필요. 대신 **PyTorchJob은 NCCL TCP bootstrap**이라 SSH 불요. 회사도 PyTorchJob 중심.

### Q3. Mutating Webhook이 실패하면?
A. Webhook 설정의 `failurePolicy`가 `Fail`이면 Pod 생성 전체 차단(admission 실패). `Ignore`면 주입 없이 통과. 우리는 **MLflow env 같이 없어도 사용자가 명시하면 되는 것**은 `Ignore`, **NodeSelector 같이 보안 관련**은 `Fail`. 양방향 설계.

### Q4. MLflow가 RGW를 쓰는 이유를 SSD pool 분리 외에 설명.
A. (1) 동일 Ceph 인프라 내 trust domain — 추가 자격 증명 관리 불필요. (2) Ceph RGW는 **versioning + lifecycle policy**를 오브젝트 단위로 지원 → 오래된 artifact 자동 HDD 이관 가능. (3) RGW의 **bucket notification**으로 artifact 업로드 시 이벤트 발행 → 후속 파이프라인 트리거.

### Q5. Airflow KubernetesExecutor와 CeleryExecutor 차이, 왜 K8sExecutor?
A. Celery는 worker가 상시 기동 → 유휴 비용 + 스케일 한계. KubernetesExecutor는 Task 1개당 Pod 1개 — **리소스는 필요할 때만**. 단점은 Pod 스폰 지연(수초). 우리 DAG은 task당 분~시간 단위라 스폰 지연이 무시 가능.

### Q5-꼬리. Task 스파이크 100개가 동시에 오면?
A. scheduler의 `parallelism` + namespace ResourceQuota로 상한. Kueue를 앞에 두면 queue 관리. KFP는 이런 스파이크에 Argo Workflows 방식이 더 유리.

### Q6. Kubeflow Profile이 그냥 namespace랑 뭐가 다른가?
A. Profile은 **namespace + RBAC + Istio AuthorizationPolicy + 기본 PodDefault**를 트랜잭션 단위로 묶은 번들. 사용자 onboarding 시 namespace 하나만 만들면 모든 것이 일관되게 적용됨. 삭제 시에도 하위 리소스 일괄 정리.

### Q7. PyTorchJob이 떴는데 NCCL timeout. 가장 먼저 볼 것?
A. 순서대로:
1. `kubectl describe pod master-pod` — NAD annotation, RDMA 리소스 할당 확인.
2. `exec` 해서 `ibstatus` — IB link 살아있나.
3. Worker Pod에서 `nslookup MASTER_ADDR` — headless service DNS 확인.
4. `NCCL_DEBUG=INFO`로 재시작 → 실패 위치(bootstrap/transport) 확인.

### Q8. Airflow v3.1.7로 업그레이드한 이유?
A. (1) FAB auth manager가 `cachetools` stale 문제 해결. (2) Deferrable operator 표준화 → sensor 부하 감소. (3) 3.x는 내부 스케줄러 재작성 + DAG parser 병렬화로 스케줄 지연 감소.

---

## 11. 체크리스트 (한 줄 요약)

- [ ] PyTorchJob의 자동 주입 env 4개 암기 (MASTER_ADDR/PORT, WORLD_SIZE, RANK)
- [ ] Training Operator v1.7+ 단일 컨트롤러 전환 설명
- [ ] MPIJob vs PyTorchJob 트레이드오프 (SSH, launcher)
- [ ] Profile 생성 시 자동 리소스 5가지
- [ ] MutatingWebhook의 `failurePolicy` 설계 (Fail vs Ignore)
- [ ] MLflow artifact S3 path-style 설정 이유
- [ ] Airflow KubernetesExecutor 선택 근거
- [ ] Airflow 복구 스토리 (v3.1.7 업그레이드 + health-monitor)

---

## 12. 참고

- 회사 레포: `ml-platform/thirdparty/dn-manifest/mlops/`
  - `kubeflow/` (training-operator, mutating-admission-webhook)
  - `mlflow/` (values.yaml, ceph-rgw secret)
  - `airflow/` (values.yaml, gitsync, health-monitor cronjob)
- Confluence: 11797049(Kubeflow 가이드), 95584383(MLflow 마이그레이션), 103253781(Airflow seminar), 95912565(Harbor 마이그레이션)
- Jira: ML-30(Airflow 복구)
- 관련 딥다이브: [inference-serving-deep-dive.md](inference-serving-deep-dive.md), [multi-tenancy-scheduler-deep-dive.md](multi-tenancy-scheduler-deep-dive.md), [../integration/dgx-ib-multinode-training-guide.md](../integration/dgx-ib-multinode-training-guide.md)
