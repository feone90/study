# 로그 파이프라인 딥다이브 — Fluent Bit / Loki / Elasticsearch / OpenSearch

> **목적**: 컨테이너 로그가 **stdout → kubelet → 파일 → agent → backend → 쿼리** 로 가는 과정을 계층별로. Prometheus(메트릭)와의 분업, Loki와 Elasticsearch 트레이드오프, cardinality 지뢰까지.
> **선행**: [observability-deep-dive.md](observability-deep-dive.md)
> **연결**: [security-deep-dive.md](security-deep-dive.md)

---

## 0. 큰 그림

```
 Pod container
   │  write to stdout/stderr
   ▼
 containerd → /var/log/pods/<ns>_<pod>_<uid>/<container>/<n>.log
   │
   ▼  (tail, 노드별)
 Fluent Bit / Fluentd / Vector (DaemonSet)
   │  parse / enrich (labels) / filter
   ▼
 Loki  또는  Elasticsearch/OpenSearch
   │
   ▼
 Grafana (Loki) / Kibana (ES)
```

**면접 프레임**: 로그 스택 선택은 **(1) 쿼리 패턴, (2) 저장 비용, (3) 카디널리티, (4) 팀 친숙도** 4축으로 정한다.

---

## 1. 컨테이너 로그 경로

### 1.1 kubelet / containerd 규약

- 컨테이너가 stdout/stderr로 쓰면 containerd가 **JSON-lines** 또는 **CRI 로그 포맷**으로 파일에 기록:
  ```
  /var/log/pods/<namespace>_<pod>_<uid>/<container>/<restart>.log
  ```
- 호스트에서 `kubectl logs`는 kubelet이 같은 파일을 tail.

### 1.2 CRI 로그 포맷

```
2026-04-14T10:23:45.123456789Z stdout F Some log line here
```
- RFC3339 timestamp
- stream (`stdout`/`stderr`)
- tag (`F` = full, `P` = partial, **splitting**)
- content

partial tag `P`는 **16KB 넘는 라인**이 chunk 분할됐을 때. 재조립은 agent 책임.

### 1.3 로테이션

kubelet의 `containerLogMaxSize`, `containerLogMaxFiles`. 기본 10Mi × 5 파일. 초과 시 오래된 파일 삭제.

**주의**: agent가 tail 중 파일 rotate → inode 변경 → 마지막 몇 라인 유실 가능. Fluent Bit은 `Rotate_Wait` 옵션.

---

## 2. Fluent Bit

### 2.1 왜 Fluent Bit인가

| 축 | Fluent Bit | Fluentd | Vector |
|----|-----------|---------|--------|
| 언어 | C | Ruby + C plugins | Rust |
| 메모리 | 수십 MB | 수백 MB | 수십 MB |
| 플러그인 | 중간 | 매우 많음 | 증가 중 |
| K8s native | DaemonSet 표준 | 무거움 | 가능 |
| 라이선스 | Apache | Apache | MPL |

**결론**: 에지(노드)에는 Fluent Bit, 집계 서버에는 Fluentd/Vector가 전통. 최근엔 Fluent Bit만으로도 충분.

### 2.2 파이프라인 구조

```
[INPUT] → [PARSER] → [FILTER] → [OUTPUT]
```

각 단계 plugin.

### 2.3 샘플 설정 (ConfigMap)

```ini
[SERVICE]
    Flush         1
    Log_Level     info
    Parsers_File  parsers.conf

[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            cri
    Tag               kube.*
    Refresh_Interval  5
    Rotate_Wait       30
    Mem_Buf_Limit     50MB
    Skip_Long_Lines   On
    DB                /var/log/fluent-bit.db

[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Merge_Log           On
    Keep_Log            Off
    K8S-Logging.Parser  On
    K8S-Logging.Exclude On

[OUTPUT]
    Name   loki
    Match  kube.*
    Host   loki.monitoring.svc
    Port   3100
    Labels $kubernetes['namespace_name'], $kubernetes['pod_name'], $kubernetes['container_name']
```

### 2.4 kubernetes 필터가 하는 일

- `/var/log/containers/<pod>_<ns>_<container>-<hash>.log` 경로에서 메타데이터 파싱.
- apiserver에 질의해 label, annotation 보강.
- `kubernetes.pod_name` 등 필드 추가.

### 2.5 Backpressure / Buffering

- `Mem_Buf_Limit` — in-memory 버퍼 한도. 초과 시 Pause.
- `storage.type filesystem` — 디스크 spool. 재시작 후에도 유지.

---

## 3. Loki

### 3.1 개념

Grafana Labs의 **로그 전용 TSDB**. Prometheus 스타일 "label index + chunk store".

**핵심 철학**: "로그 본문은 인덱싱 안 한다, **라벨만 인덱싱**". 덕분에 **저장 비용 저렴**.

### 3.2 구성요소

- **Distributor** — write path, label-based routing.
- **Ingester** — 메모리 버퍼링 + chunk 생성 → 오브젝트 스토리지.
- **Querier** — 질의 처리, chunk 풀백.
- **Compactor** — chunk 압축/merge.
- **Storage** — 오브젝트 스토리지(S3, GCS, **Ceph RGW**).

**monolithic / simple scalable / microservices** 3가지 배포 모드.

### 3.3 LogQL

```
{namespace="slm", pod=~"pytorch-.*"} |= "NCCL" | json | latency > 100
```

- `{...}` — label selector.
- `|=`, `|~`, `!=` — line filter.
- `| json`, `| logfmt` — 파서.
- aggregation: `rate(... [5m])`, `count_over_time(... [5m])`.

### 3.4 Cardinality 함정

라벨 값이 너무 다양하면 **chunk 폭발** → 메모리/쿼리 성능 망함.

**Do**:
- `namespace`, `pod`, `container`, `node` — 안정적인 값.

**Don't**:
- `request_id`, `trace_id`, `user_id` — 고유 값.
- `path` — wildcard 수천.

고유 값은 **로그 본문 안으로** (LogQL이 line filter로 찾음). 라벨로 올리면 Loki가 무너진다.

### 3.5 Chunk 저장

- Chunk = 특정 label set의 로그 블록. gzip 압축.
- Ceph RGW에 `s3://loki-chunks/<tenant>/...`.
- `chunk_idle_period` 넘기면 flush.

---

## 4. Elasticsearch / OpenSearch

### 4.1 개념

**full-text 역색인**. 로그 본문도 인덱싱 → 빠른 검색/집계.

### 4.2 구성요소

- **Master** node — cluster state.
- **Data** node — 샤드 저장.
- **Coordinator** — 쿼리 분배.
- **Ingest** — 파이프라인.

### 4.3 Index Lifecycle

- daily index: `logs-2026-04-14`.
- ILM policy: **hot → warm → cold → delete**.
  - Hot (NVMe) : 최근 7일.
  - Warm (SSD) : 30일.
  - Cold (HDD, searchable snapshot) : 90일.
  - Delete: 이후.

### 4.4 Mapping 함정

동적 매핑으로 필드가 폭발 — "mapping explosion". JSON 필드 수천 개의 pod은 cluster state 확장 → master가 OOM.

**대책**: explicit mapping, `dynamic: strict`, `ignore_malformed`.

### 4.5 샤드 설계

- 너무 많은 샤드 → 오버헤드.
- 너무 큰 샤드 → 재배치 느림, recovery 리스크.
- 권장: 샤드 크기 10~50GB.

---

## 5. Loki vs Elasticsearch 비교

| 축 | Loki | ES/OpenSearch |
|----|------|---------------|
| 본문 인덱싱 | X (라벨만) | O (역색인) |
| 저장 비용 | 매우 낮음 (오브젝트 스토리지) | 높음 (로컬 SSD/NVMe) |
| 쿼리 속도 (라벨 기반) | 빠름 | 비슷 |
| Full-text 쿼리 | 느림 (grep 느낌) | 매우 빠름 |
| 집계/분석 | 제한적 | 강력 (Kibana Discover/Visualize) |
| 운영 난이도 | 낮음-중간 | 높음 |
| 다중 테넌시 | tenant header | index pattern + role |
| K8s 친화 | Grafana + Prom 생태 | Kibana 별도 |

**선택 가이드**:
- 메트릭 중심 + 가벼운 로그 조회 → Loki.
- 보안/감사 + 복잡 검색 → ES/OpenSearch.
- 둘 다 → 로그 "hot path"는 Loki, 장기/감사는 ES로 이중화.

---

## 6. OpenTelemetry 로그

OTel Collector가 로그 파이프라인도 표준화 중:

```
[receiver: filelog / k8slog]
  → [processor: batch, resource, attributes]
    → [exporter: loki, elasticsearch, otlp]
```

**장점**: 메트릭/트레이스/로그를 같은 agent로 처리. **상관관계** (trace_id) 가 로그에 자동 주입.

**현실**: 회사는 보통 Fluent Bit + OTel Collector (metric/trace만) 병행.

---

## 7. 보안

### 7.1 민감 정보 마스킹

- Fluent Bit filter `modify` 또는 Lua script로 PII 패턴 제거.
- 예: JWT는 `eyJ...` 시작 → `[REDACTED]`로.

### 7.2 Multi-tenancy

- Loki: HTTP header `X-Scope-OrgID`. 테넌트별 저장/쿼리 격리.
- ES: index pattern + role-based access.

### 7.3 로그 tamper-evidence

- 금융/감사 요구: WORM (Write Once Read Many). S3 object lock, ES frozen tier.

---

## 8. 회사/클러스터 관점

### 8.1 예상 구성 (dnotitia 규모)

- Fluent Bit DaemonSet on all nodes.
- Loki (simple scalable mode) + Ceph RGW SSD 버킷.
- Grafana로 Prometheus + Loki 통합 대시보드.
- ES는 별도 운영 없음 (cardinality/저장 비용 불리).

### 8.2 Airflow/MLflow 로그 주입

- Airflow Pod 로그 (Task 별 Pod): stdout → Fluent Bit → Loki.
- MLflow artifact는 S3(RGW)에 저장되고 로그 자체는 stdout → Loki.
- Kubeflow PyTorchJob: Master/Worker 로그 분리, Loki 라벨로 `training.kubeflow.org/job-name`.

### 8.3 dashboard 예

- **GPU node 에러 스캔**: `{node="dgx04"} |~ "XID|NVRM"`.
- **NCCL timeout 패턴**: `{container="pytorch"} |= "NCCL" |~ "timeout|unhandled"`.
- **Ceph 관련 fatal**: `{namespace="rook-ceph"} |~ "slow ops|stuck"`.

---

## 9. 실습

### 9.1 Fluent Bit 로컬 테스트

```bash
docker run --rm -v $(pwd)/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf \
  fluent/fluent-bit:latest
```

### 9.2 Loki 질의

```bash
logcli query '{namespace="slm"} |= "NCCL"' --since=1h --limit=100
```

### 9.3 Loki 라벨 카디널리티 확인

```bash
logcli series '{namespace="slm"}' --since=1h
curl http://loki:3100/loki/api/v1/labels
```

라벨 값 수 폭주하면 alert.

### 9.4 Grafana alert (에러 폭증)

```
rate({namespace="serving-prod"} |= "ERROR" [5m]) > 10
```

---

## 10. 장애/시나리오

### 10.1 "로그가 사라진다"

- Fluent Bit `Mem_Buf_Limit` 초과 → `Pause_On_Chunks_Overlimit`. 디스크 spool 전환.
- kubelet log rotate → Fluent Bit tail lost (Rotate_Wait 설정).
- Loki Ingester OOM → 쓰기 거부.

### 10.2 "Loki 쿼리 너무 느림"

- 라벨 selector가 너무 광범위 (`{job=~".*"}`).
- Long time range + line regex.
- 대책: 라벨 강화(cardinality 주의), `| json` 파서 뒤로.

### 10.3 "ES cluster RED"

- 샤드 복제 실패. `GET _cluster/health`, `GET _cat/shards?v`.
- 대책: node 추가, ILM으로 구식 index delete.

### 10.4 "CAP 쓰기가 계속 실패"

- Ceph RGW bucket 용량 부족 → Loki ingester가 S3 put 실패. Ceph 용량 알림(ML-21) 중요성.

---

## 11. 면접 Q&A

### Q1 (기초). Loki가 "Prometheus 같은 로그 DB"라고 불리는 이유?
A. Prometheus가 TSDB metric을 label + timestamp로 인덱싱하듯, Loki도 로그를 **라벨만 인덱싱**하고 본문은 gzip chunk로. 결과적으로 같은 label selector 스타일 LogQL + 저렴한 저장.

### Q2. 로그 라벨에 `request_id` 넣으면 왜 안 되나?
A. 모든 요청이 고유 값 → chunk가 요청당 하나씩 → index 폭발, ingester 메모리 폭발, 쿼리 지연. 고유 값은 **로그 본문**에 두고 `| json | request_id=...` 로 조회.

### Q3. Fluent Bit vs Fluentd?
A. Fluent Bit은 C, 가볍고 에지용. Fluentd는 Ruby, 플러그인 풍부하지만 무겁고 GC 영향. 현재 K8s DaemonSet 표준은 Fluent Bit, aggregator가 필요하면 Vector/Fluentd.

### Q4. 로그 파이프라인 OOM을 방어하는 5가지?
A. (1) `Mem_Buf_Limit` 제한. (2) 디스크 spool. (3) backpressure → pause. (4) output 쪽에 Kafka 버퍼. (5) ingester autoscale. (6) 과다 로그 파드 감지 → 격리.

### Q5. Loki에 Ceph RGW 쓸 때 주의?
A. (1) **S3 path-style**. (2) bucket lifecycle — compactor와 ILM 충돌 주의. (3) 용량 알림(ML-21 교훈). (4) RGW 장애 = 쓰기 차단 → ingester 백업.

### Q6. ES 샤드 설계 기준?
A. 샤드 크기 10-50GB. primary shard 수는 노드 수 이상. 너무 많으면 cluster state 커짐, 너무 적으면 병렬성 손실. **index per day + rollover** 패턴.

### Q7. OTel Collector로 전환 고려해야 할 시점?
A. (1) trace/metric/log **상관관계** 필요. (2) 현재 여러 agent(Fluent Bit + DCGM + OTel)가 혼재해 운영 복잡. (3) 벤더 중립성. 단점: 플러그인 성숙도 Fluent Bit 대비 낮음, 마이그레이션 리스크.

### Q8. Loki의 multi-tenant header?
A. `X-Scope-OrgID: <tenant>`. Distributor가 이 헤더로 쓰기/질의 격리. Grafana datasource에서 tenant별 UI 분리.

### Q9. 로그에 PII/토큰이 섞여 나감. 대응?
A. (1) Fluent Bit filter에서 정규식 마스킹. (2) 애플리케이션 쪽 로거 설정 수정 (근본 해결). (3) Loki/ES 쪽 IAM으로 특정 label 접근 제한. (4) 이력 삭제 — Loki는 retention 기반 삭제, 긴급하면 chunk 수동 제거.

### Q10. K8s 로그가 JSON이 아니다. 어떻게 처리?
A. (1) Fluent Bit `Parser_Firstline` 로 multi-line 병합. (2) logfmt/json 이외는 `| regexp` 로 LogQL에서 추출. (3) 이상적으론 애플리케이션이 JSON 로거 쓰게 — 운영 표준화 제안.

---

## 12. 체크리스트

- [ ] 컨테이너 로그 파일 경로 및 CRI 포맷
- [ ] Fluent Bit 파이프라인 4단계 + kubernetes filter
- [ ] Loki 구성요소 5개
- [ ] Cardinality 함정 + 대표 DO/DON'T
- [ ] ES vs Loki 비교 4축
- [ ] Log rotation edge case (rotate_wait)
- [ ] Multi-tenant 방식 (Loki header vs ES role)
- [ ] OTel Collector 통합 로드맵
- [ ] Fluent Bit backpressure 방어 (Mem_Buf_Limit, spool)

---

## 13. 참고

- Fluent Bit: https://docs.fluentbit.io/
- Loki: https://grafana.com/docs/loki/latest/
- Elastic ILM: https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html
- OTel Collector: https://opentelemetry.io/docs/collector/
- 관련: [observability-deep-dive.md](observability-deep-dive.md), [security-deep-dive.md](security-deep-dive.md)
