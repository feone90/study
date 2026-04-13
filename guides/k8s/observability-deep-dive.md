# Observability 완전 정복 - 메트릭/로그/트레이싱

> **목적**: 운영자의 절반의 일은 "지금 뭐가 어떻게 돌아가고 있나"를 보는 것. Prometheus, Loki, OpenTelemetry, eBPF 기반 관측까지.

---

## 0. 관측의 3대 축 (Three Pillars)

| 축 | 답하는 질문 | 도구 |
|----|------------|------|
| **Metrics** | "얼마나, 몇 번, 빠른가?" | Prometheus, Grafana |
| **Logs** | "정확히 무슨 일이 있었나?" | Loki, ELK |
| **Traces** | "요청이 어떤 경로로 흘렀나?" | Jaeger, Tempo |

거기에 추가로:
- **Profiling**: "왜 느린가? 어디서 시간을 쓰는가?" (Pyroscope)
- **Events**: "K8s에서 무슨 일이 일어났나?" (kube-events)

---

## 1. Metrics - Prometheus

### 1.1 데이터 모델

```
metric_name{label1="v1", label2="v2"} value @timestamp
```

예:
```
http_requests_total{method="GET", path="/api", status="200"} 1234 @1700000000
gpu_utilization{node="dgx01", gpu="0"} 95.5 @1700000000
```

핵심:
- **label**으로 차원 분리 (다차원 시계열 DB)
- 모든 메트릭은 시간에 따른 값의 시퀀스

### 1.2 메트릭 타입

| 타입 | 설명 | 예 |
|------|------|-----|
| **Counter** | 단조 증가 | 총 요청 수, 에러 수 |
| **Gauge** | 임의 값 (오르락내리락) | 메모리 사용량, 온도 |
| **Histogram** | 값의 분포 (버킷) | 응답 시간 분포 |
| **Summary** | 분위수 계산 | latency p50/p99 |

### 1.3 Pull vs Push

Prometheus는 **pull** 방식.

```
[Prometheus]
   ↓ HTTP GET /metrics (15초마다)
[Application 또는 Exporter]
   ↓
metric1 ...
metric2 ...
metric3 ...
```

장점:
- 어디서 누가 수집하는지 명확
- target 헬스 체크 자연스러움
- 단순 (HTTP만 알면 됨)

단점:
- 짧게 사는 잡(Job)에는 부적합 → **Pushgateway**로 보완

### 1.4 PromQL

Prometheus의 쿼리 언어.

```promql
# 최근 5분간 HTTP 에러율
rate(http_requests_total{status=~"5.."}[5m])
  /
rate(http_requests_total[5m])

# GPU 사용률 95% 이상인 노드들
DCGM_FI_DEV_GPU_UTIL > 95

# Pod별 메모리 사용량 top 10
topk(10, container_memory_usage_bytes{namespace="slm"})

# 5분 평균 + 비교 (지난주 같은 시간 대비)
avg_over_time(metric[5m]) /
avg_over_time(metric[5m] offset 1w)
```

핵심 함수:
- `rate()`: counter의 초당 증가율
- `irate()`: 마지막 두 데이터 포인트 기준 (변동 큼)
- `histogram_quantile(0.99, ...)`: p99 계산
- `sum by (label) (...)`: label 기준 그룹 합산

### 1.5 Exporter 패턴

내장 메트릭 없는 시스템에서 Prometheus 형식으로 변환해주는 사이드카.

| Exporter | 노출 메트릭 |
|----------|------------|
| node-exporter | OS/하드웨어 메트릭 |
| dcgm-exporter | NVIDIA GPU |
| infiniband-exporter | IB 포트 |
| ceph-exporter | Ceph 클러스터 |
| postgres-exporter | PostgreSQL |
| blackbox-exporter | HTTP 헬스 체크 |

각자 자기 시스템 상태를 `/metrics` 엔드포인트로 노출. Prometheus가 긁어감.

### 1.6 K8s에서 ServiceMonitor

Prometheus Operator가 추가하는 CRD.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
spec:
  selector:
    matchLabels: { app: my-app }
  endpoints:
  - port: metrics
    interval: 15s
```

`my-app` Service의 metrics 포트를 자동으로 Prometheus가 긁어감. 직접 Prometheus 설정 안 건드려도 됨.

---

## 2. Grafana - 시각화

### 2.1 역할

- 다양한 데이터 소스(Prometheus, Loki, Tempo, ...)에서 데이터 가져와 대시보드로
- 알람 설정도 가능

### 2.2 대시보드 구성 예 (DGX 클러스터)

```
[전체 클러스터 개요]
├── 노드 상태 / GPU 사용률 / IB 트래픽 (overview)

[노드별 뷰]
├── CPU/메모리/디스크
├── GPU 사용률, 메모리, 온도, 전력
├── IB 포트별 rx/tx, 에러 카운터
└── NVLink 트래픽

[학습 잡 뷰]
├── PyTorchJob 별 GPU 사용률, 메모리
├── NCCL 통신 시간 비율
└── 데이터 로딩 throughput

[Ceph 뷰]
├── OSD 상태, 사용량
└── IOPS, 대역폭
```

### 2.3 Grafana 변수

`$node`, `$namespace` 같은 변수로 인터랙티브 대시보드 만들 수 있음.

```promql
DCGM_FI_DEV_GPU_UTIL{node="$node"}
```

---

## 3. Logs - Loki

### 3.1 Loki란

Grafana Labs의 로그 시스템. **"Prometheus for logs"** 컨셉.

특징:
- **인덱스는 메타데이터(라벨)만**, 본문은 압축 저장 → 저렴
- LogQL: PromQL과 비슷한 쿼리 언어
- Grafana와 자연스럽게 통합

### 3.2 vs ELK

| 항목 | Loki | ELK (Elasticsearch) |
|------|------|---------------------|
| 인덱싱 | 라벨만 | 본문 풀텍스트 |
| 비용 | 저렴 | 비쌈 (디스크/메모리) |
| 검색 | 라벨 기반 빠름, 풀텍스트 grep | 풀텍스트 빠름 |
| 적합 | 클라우드 네이티브, K8s | 본격 검색 시스템 |

K8s 환경에서는 Loki가 점점 표준.

### 3.3 LogQL

```logql
# slm 네임스페이스의 모든 컨테이너 로그
{namespace="slm"}

# 에러만
{namespace="slm"} |= "ERROR"

# 정규식
{namespace="slm"} |~ "level=(ERROR|FATAL)"

# 메트릭처럼 집계 (분당 에러 수)
sum by (pod) (rate({namespace="slm"} |= "ERROR" [1m]))
```

### 3.4 로그 수집기

Loki에 로그 보내는 에이전트들:

- **Promtail**: Loki 공식 (DaemonSet)
- **Fluent Bit**: 가볍고 빠름
- **Fluentd**: 더 무겁지만 강력한 변환
- **Vector**: 차세대 (Rust 기반)

K8s에서 일반 동작:
```
[각 노드: DaemonSet]
   ↓ /var/log/containers/*.log 읽음
[Promtail / Fluent Bit]
   ↓ K8s 메타데이터(namespace, pod, container) 자동 추가
[Loki]
```

---

## 4. Traces - 분산 트레이싱

### 4.1 문제

마이크로서비스 환경에서 한 요청이 여러 서비스를 거침.

```
[클라이언트] → [API Gateway] → [Auth Service] → [Order Service] → [Inventory Service] → [DB]
                                                                                    ↓
                                                                            [캐시 Service]
```

어디서 느린지? 어디서 에러 났는지? 단순 로그로는 추적 불가.

### 4.2 분산 트레이싱

요청에 **trace ID**와 **span ID** 부여, 모든 서비스가 이걸 공유.

```
trace_id = abc123
├── span: api-gateway (50ms)
│   └── span: auth-service (10ms)
└── span: order-service (200ms)
    ├── span: inventory-service (50ms)
    └── span: cache (3ms)
```

→ Jaeger/Tempo 같은 도구가 시각화.

### 4.3 OpenTelemetry (OTel)

벤더 중립적 표준.

```
[Application]
   ↓ OTel SDK (Python/Go/Java)
[OTel Collector]
   ↓ 다양한 백엔드로 전송
[Jaeger / Tempo / Datadog / 등]
```

이제 메트릭/로그/트레이스 모두 OTel로 통합되는 추세.

### 4.4 자동 계측 (auto-instrumentation)

```bash
# Python (Flask 예시)
opentelemetry-instrument --traces_exporter otlp \
  --metrics_exporter otlp \
  --service_name myservice \
  python app.py
```

코드 변경 없이 HTTP 요청, DB 쿼리 등 자동 트레이싱. ML 환경보다는 마이크로서비스에서 진가 발휘.

---

## 5. Profiling - 어디가 느린가

### 5.1 종류

- **CPU**: 어떤 함수가 CPU 시간을 먹나
- **메모리**: 누가 할당하나, leak 어디서
- **GPU**: NVIDIA Nsight, PyTorch profiler

### 5.2 Pyroscope / Parca

**continuous profiling** - 운영 중인 시스템을 항상 프로파일링.

```
[애플리케이션] → eBPF로 stack sampling → [Pyroscope] → flame graph
```

언제든 "지금 CPU를 누가 먹나" 확인 가능. 이슈 발생 후가 아니라 항상.

### 5.3 PyTorch Profiler

```python
from torch.profiler import profile, ProfilerActivity

with profile(activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA]) as prof:
    model(input)

print(prof.key_averages().table(sort_by="cuda_time_total"))
```

학습 잡의 CPU/GPU/메모리 타임라인. TensorBoard로 시각화 가능.

---

## 6. K8s Events

종종 잊히는 관측 데이터.

```bash
kubectl get events --all-namespaces --sort-by=.lastTimestamp
# 50s    Warning   FailedScheduling   pod/mypod   0/4 nodes have GPU
# 30s    Normal    Scheduled          pod/mypod   Successfully assigned to dgx01
# 25s    Normal    Pulling            pod/mypod   Pulling image
```

K8s가 자체적으로 발생시키는 이벤트. **트러블슈팅의 첫 단계는 항상 events 확인**.

도구: kube-events-exporter로 Prometheus 메트릭화 가능.

---

## 7. eBPF 기반 관측 (차세대)

### 7.1 컨셉

커널에 안전하게 코드를 넣어서 모든 시스템 활동을 관찰.

```
일반 모니터링: 앱이 메트릭을 export → exporter → Prometheus
eBPF: 커널이 syscall, 네트워크 패킷 등을 직접 관측 → 메트릭화
```

장점:
- 앱 변경 불필요 (블랙박스 관측)
- 매우 낮은 오버헤드
- 모든 syscall, 네트워크 패킷 관측 가능

### 7.2 도구

- **Cilium Hubble**: 네트워크 관측 (Pod 간 통신 모두 시각화)
- **Pixie**: K8s 자동 관측 (HTTP, MySQL, gRPC 자동 디코드)
- **Falco**: 보안 이벤트 (이상한 syscall 감지)
- **bpftrace**: ad-hoc 트레이싱

### 7.3 예: Hubble

```
hubble observe --pod slm/training-master-0
# 실시간으로 그 Pod의 모든 네트워크 통신 표시
# (어느 Pod에 어느 포트로, HTTP method까지)
```

---

## 8. 알람 (Alerting)

### 8.1 Prometheus Alertmanager

```yaml
# Prometheus 룰
groups:
- name: gpu
  rules:
  - alert: GPUTempHigh
    expr: DCGM_FI_DEV_GPU_TEMP > 85
    for: 5m
    annotations:
      summary: "GPU {{ $labels.gpu }} on {{ $labels.node }} is too hot"
```

5분 이상 조건 만족 → Alertmanager로 → Slack/PagerDuty/Email로 전달.

### 8.2 SLI / SLO

| 용어 | 의미 |
|------|------|
| **SLI** (Indicator) | 측정 가능한 신호 (예: 가용성 %) |
| **SLO** (Objective) | 목표치 (예: 99.9% 가용성) |
| **SLA** (Agreement) | 계약상 약속 (위반 시 페널티) |

알람 설계의 원칙:
- "CPU 90% 넘음" 보다 "사용자 경험 영향" 기반 알람
- "에러율 5분간 1% 초과 → 알람" 식

### 8.3 Alert fatigue

알람 너무 많으면 무시됨. 신중하게:
- 정말 사람이 깨야 하는 것만
- 자동 복구 가능한 건 자동화
- 매주 알람 회고 → 노이즈 제거

---

## 9. 우리 회사 환경 적용

### 9.1 추천 스택

```
[Metrics]
  Prometheus + Prometheus Operator
  ├── node-exporter (DaemonSet)
  ├── dcgm-exporter (GPU)
  ├── infiniband_exporter (IB)
  ├── ceph-exporter
  └── kube-state-metrics (K8s 객체 상태)
  ↓
  Grafana

[Logs]
  Loki + Promtail (DaemonSet)
  ↓
  Grafana

[Events]
  kube-events-exporter (이벤트 메트릭화)

[Profiling] (optional)
  Pyroscope + eBPF
```

### 9.2 핵심 대시보드

1. **클러스터 헬스**: 노드/Pod/etcd 상태
2. **GPU 사용률**: DGX 노드별
3. **IB 트래픽**: 포트별 rx/tx, 에러
4. **Ceph**: OSD 상태, IOPS, latency
5. **학습 잡**: PyTorchJob별 진행 상황, 자원 사용

### 9.3 핵심 알람

```
- GPU 온도 85°C 이상 5분 → 경고
- IB 링크 다운 → 즉시
- Ceph HEALTH_WARN 또는 ERR → 즉시
- 노드 NotReady 5분 이상 → 즉시
- etcd 멤버 다운 → 즉시
- 디스크 90% 이상 → 경고
```

---

## 10. 트러블슈팅에서 관측 활용

### 10.1 학습 잡이 느림

```
1. PyTorchJob 메트릭 확인
   - GPU 사용률 낮음? → 데이터 로딩 또는 통신 병목
2. IB 사용률 확인
   - 높음? → 통신 병목 → 모델/배치 조정
   - 낮은데 GPU도 낮음? → 데이터 로딩 의심
3. Ceph 메트릭
   - 학습 데이터 read latency 높음?
4. dcgm SM Active
   - 낮음? → GPU 자체가 활성화 안 됨
```

### 10.2 노드가 갑자기 NotReady

```
1. K8s Events 확인
2. node-exporter 메트릭 (디스크, 메모리)
3. kubelet 로그 (Loki)
4. dmesg (eBPF로 가능, 또는 직접 노드 접근)
```

### 10.3 사용자가 "Pod이 OOMKilled"

```
1. 그 Pod의 메모리 메트릭 확인
   - limit 가까이 갔는지
2. cgroup memory.events
   - oom_kill 카운터
3. 패턴 분석
   - 점진적 증가 (leak)? 갑작스런 spike?
```

---

## 11. 면접 예상 질문

### Q1. "Observability와 Monitoring의 차이는?"
> "Monitoring은 알려진 문제(known unknowns)를 감시합니다 — 'CPU가 80% 넘으면 알람'처럼. Observability는 모르는 문제(unknown unknowns)를 탐색할 수 있는 능력입니다. 메트릭/로그/트레이스가 충분히 풍부해야 사후에 '왜 그런 일이 일어났나'를 추적할 수 있습니다."

### Q2. "Prometheus가 pull 방식인 이유는?"
> "타겟 디스커버리와 헬스 체크를 자연스럽게 통합할 수 있고, push 폭주로 인한 메트릭 시스템 다운을 방지할 수 있으며, 클라이언트 구현이 단순합니다. 단점은 짧게 사는 잡에 부적합한데 이건 Pushgateway로 보완합니다."

### Q3. "DCGM vs IB 모니터링?"
> "DCGM은 NVIDIA 드라이버 레벨에서 GPU 메트릭만 수집합니다. IB는 Mellanox 드라이버 영역이라 DCGM 범위 밖이고, infiniband_exporter로 별도 수집해야 합니다. 보통 둘을 같이 띄워 'GPU는 노는데 IB가 포화' 같은 상관 분석을 합니다."

### Q4. "Loki와 ELK 차이는?"
> "Loki는 라벨만 인덱스해서 비용이 매우 저렴하고 K8s 메타데이터 통합이 자연스럽습니다. ELK는 본문 풀텍스트 인덱스라 검색이 강력하지만 디스크/메모리가 비쌉니다. K8s 환경에서 일반적인 로그 보기/추적용은 Loki, 본격 검색/분석은 ELK가 적합합니다."

### Q5. "알람 노이즈 줄이는 방법?"
> "사용자 경험 기반 SLO를 정하고 그 위반에만 알람합니다. 자원 임계치(CPU 90%) 같은 것보다 '에러율 1% 5분 지속' 같은 식. for 절로 짧은 spike 무시, severity 분리(P0~P3), 자동 복구 가능한 건 알람 대신 자동화로 처리합니다."

---

## 12. 한 줄 요약

> **"Observability = 메트릭(Prometheus)으로 추세를, 로그(Loki)로 사실을, 트레이스(OTel/Jaeger)로 경로를 본다. K8s 환경에서는 Operator로 자동 디스커버리(ServiceMonitor)를 설정하고, eBPF 기반 관측(Cilium Hubble, Pixie)으로 앱 변경 없이 더 깊이 본다. 우리 환경에서는 GPU(DCGM), IB(infiniband_exporter), Ceph 메트릭의 상관 분석이 핵심이다."**
