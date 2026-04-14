# cgroup 완전 정복 - 컨테이너 자원 격리의 다른 절반

> **목적**: 컨테이너 = namespace(보이는 것 격리) + cgroup(자원량 제한). namespace는 CNI 문서에서 다뤘으니, 이 문서는 cgroup의 모든 것을 커널 레벨까지 본다.

---

## 0. 한 줄 정리부터

> **namespace가 "무엇을 볼 수 있는가"를 격리한다면, cgroup은 "얼마나 쓸 수 있는가"를 제한한다.**

```
컨테이너 = namespace + cgroup + (chroot/overlayfs 같은 파일시스템 격리)
            ─────      ─────
            "보이는 것"  "쓸 수 있는 양"
```

---

## 1. cgroup이 없으면 어떻게 되는가

namespace만 있고 cgroup이 없는 컨테이너를 상상해봅시다.

```
호스트 (CPU 8코어, 메모리 64GB)
├── Pod A (격리는 됨)
│   └── 무한 루프 도는 코드
│       → CPU 800% 점유 (8코어 다 씀)
│       → 다른 Pod도, 호스트도 마비
└── Pod B
    └── 메모리 leak
        → 64GB 다 먹음
        → OOM으로 시스템 다운
```

**격리만으로는 부족합니다.** 자원도 제한해야 안전합니다. cgroup이 그 일을 합니다.

---

## 2. cgroup이란 무엇인가

**control group**의 약자. 리눅스 커널의 자원 관리 메커니즘.

- 프로세스들을 그룹으로 묶음
- 그룹 단위로 CPU/메모리/IO/네트워크 등을 **제한, 측정, 격리**

```
[cgroup: my-pod]
    ├── CPU: 최대 2코어
    ├── 메모리: 최대 4GB
    ├── 디스크 IO: 최대 100MB/s
    └── 소속 프로세스: PID 1234, 5678, 9012
```

---

## 3. cgroup v1 vs v2

리눅스에는 두 버전이 있습니다. **v2가 표준**이지만 v1도 여전히 많이 보입니다.

### 3.1 v1 - 자원별 별도 계층 (legacy)

```
/sys/fs/cgroup/
├── cpu/        ← CPU 제어 전용 트리
│   └── my-pod/
├── memory/     ← 메모리 제어 전용 트리
│   └── my-pod/
├── blkio/      ← 블록 IO 전용 트리
│   └── my-pod/
└── ...
```

문제점:
- 자원별로 트리가 따로 있어서 일관성 떨어짐
- 한 프로세스가 cpu에서는 그룹A, memory에서는 그룹B에 속할 수 있음 (혼란)

### 3.2 v2 - 통합 계층 (unified hierarchy)

```
/sys/fs/cgroup/
└── my-pod/
    ├── cpu.max
    ├── memory.max
    ├── io.max
    └── ...     ← 모든 자원이 한 곳에서 통합 관리
```

- 모든 자원이 같은 트리에서 관리됨
- K8s도 v2로 전환 중 (Ubuntu 22.04+, RHEL 9+ 기본 v2)

### 3.3 어떻게 확인하나

```bash
# v2면 cgroup2 마운트가 있음
mount | grep cgroup
# cgroup2 on /sys/fs/cgroup type cgroup2 ...   ← v2

# 시스템 전체 확인
stat -fc %T /sys/fs/cgroup/
# cgroup2fs   ← v2
# tmpfs       ← v1
```

---

## 4. cgroup이 제어하는 자원들

### 4.1 CPU

| 파일 (v2) | 의미 |
|----------|------|
| `cpu.max` | 사용 가능한 최대 CPU 시간. `100000 100000` = 1코어 |
| `cpu.weight` | 상대적 가중치 (1~10000, 기본 100) |
| `cpu.stat` | 사용 통계 (usage_usec, throttled_usec 등) |

**cpu.max 동작 예시**:
```bash
# cpu.max = "200000 100000"
# 100ms 주기당 200ms 사용 가능 → 2코어
```

핵심 메커니즘: **CFS bandwidth control**
- 100ms 주기마다 정해진 시간만큼만 CPU 할당
- 초과하면 다음 주기까지 **throttle** (강제 대기)

### 4.2 메모리

| 파일 (v2) | 의미 |
|----------|------|
| `memory.max` | 하드 리밋. 초과 시 OOM Kill |
| `memory.high` | 소프트 리밋. 초과 시 회수 압박 |
| `memory.current` | 현재 사용량 |
| `memory.events` | OOM 발생 횟수 등 |

**OOM Killer**:
- `memory.max` 초과 시 cgroup 안의 프로세스 중 하나를 죽임
- `oom_score` 가 높은 프로세스 우선
- K8s에서 Pod이 갑자기 죽으면 99% 이거

### 4.3 Block IO

| 파일 | 의미 |
|------|------|
| `io.max` | 디바이스별 read/write 대역폭/IOPS 제한 |
| `io.stat` | 통계 |

```
# /dev/sda 디바이스에 대해
io.max = "8:0 rbps=10485760 wbps=10485760"
       → 10MB/s read, 10MB/s write
```

### 4.4 PID

```
pids.max = 100   # 이 cgroup 안의 프로세스 수 제한
```

fork bomb 방어용.

### 4.5 cpuset (★ K8s CPU Manager)

| 파일 (v2) | 의미 |
|----------|------|
| `cpuset.cpus` | 이 cgroup이 쓸 수 있는 CPU 리스트 (예: `0-7,16-23`) |
| `cpuset.mems` | 쓸 수 있는 NUMA 노드 |

- `kubelet --cpu-manager-policy=static` + Guaranteed Pod + 정수 CPU → 해당 Pod cgroup에 `cpuset.cpus`로 전용 코어 할당.
- Topology Manager와 결합해 GPU/NIC NUMA alignment.

### 4.6 rdma (★ DGX 관심)

```
/sys/fs/cgroup/<grp>/rdma.max
mlx5_0 hca_handle=16 hca_object=16384
```

RDMA QP/PD 등 객체 수 제한. 공유 HCA에서 QP 고갈로 이웃 Pod 실패 방지.

### 4.7 PSI (Pressure Stall Information) — v2 진단 필수

```bash
cat /sys/fs/cgroup/my-pod/memory.pressure
# some avg10=5.23 avg60=3.11 avg300=1.50 total=...
# full avg10=0.42 ...
```

- `some`: 하나 이상의 task가 리소스 대기 중인 시간 비율
- `full`: 모든 task가 stall된 시간 비율 (cpu.pressure에는 full 없음)
- `cpu.stat throttled_usec`보다 정밀. node-exporter PSI collector로 노출.

### 4.8 디바이스, 네트워크 등

- `devices` - 어떤 디바이스 파일 접근 가능한지
- 네트워크는 cgroup이 아니라 **tc (traffic control)** 등으로 처리

---

## 5. 직접 만들어보기

cgroup도 namespace처럼 직접 만질 수 있습니다.

### 5.1 cgroup 생성

```bash
# v2 기준
sudo mkdir /sys/fs/cgroup/my-test

# 생성된 파일들 확인
ls /sys/fs/cgroup/my-test/
# cpu.max  cpu.stat  cpu.weight
# memory.max  memory.current  memory.events
# io.max  io.stat
# cgroup.procs  cgroup.threads
# ...
```

폴더만 만들었는데 파일들이 자동 생성됩니다. **cgroupfs** (가상 파일시스템)이 그렇게 동작.

### 5.2 제한 설정

```bash
# CPU 0.5코어 제한
echo "50000 100000" | sudo tee /sys/fs/cgroup/my-test/cpu.max

# 메모리 100MB 제한
echo "104857600" | sudo tee /sys/fs/cgroup/my-test/memory.max
```

### 5.3 프로세스를 cgroup에 넣기

```bash
# 새 셸을 cgroup에 넣음
sudo bash -c "echo $$ > /sys/fs/cgroup/my-test/cgroup.procs && exec bash"

# 이제 이 셸과 모든 자식 프로세스가 cgroup에 속함
# CPU 부하 테스트
yes > /dev/null  # 무한 루프
# top으로 보면 CPU 50%만 씀 (1코어가 아니라 0.5코어)
```

### 5.4 실험: OOM 직접 트리거

```bash
# 메모리 100MB 제한 cgroup에서
python3 -c "x = 'a' * (200 * 1024 * 1024)"
# Killed   ← OOM Killer가 죽임

# events 확인
cat /sys/fs/cgroup/my-test/memory.events
# oom 1
# oom_kill 1
```

---

## 6. K8s requests/limits → cgroup 매핑 (★ 면접 단골)

### 6.1 K8s의 requests / limits

```yaml
resources:
  requests:
    cpu: "500m"      # 0.5코어
    memory: "256Mi"
  limits:
    cpu: "2"         # 2코어
    memory: "1Gi"
```

### 6.2 이게 cgroup으로 어떻게 변환되나

**CPU**:
- `requests.cpu` → `cpu.weight` (스케줄러 우선순위)
- `limits.cpu` → `cpu.max` (하드 리밋)

```
limits.cpu: 2
  ↓
cpu.max = "200000 100000"   (100ms 주기당 200ms = 2코어)
```

**메모리**:
- `requests.memory` → 스케줄러 결정에만 사용 (cgroup 없음)
- `limits.memory` → `memory.max` (하드 리밋)

```
limits.memory: 1Gi
  ↓
memory.max = 1073741824
```

### 6.3 QoS Class

K8s는 requests/limits 조합으로 QoS 등급을 매김:

| QoS | 조건 | OOM 시 우선순위 |
|-----|------|----------------|
| **Guaranteed** | requests = limits (모든 자원) | 가장 늦게 죽음 |
| **Burstable** | requests < limits | 중간 |
| **BestEffort** | requests/limits 모두 없음 | 먼저 죽음 |

OOM 발생 시 어떤 Pod이 죽을지가 여기서 결정됩니다.

### 6.4 실제 cgroup 경로 확인

```bash
# Pod의 컨테이너 ID 확인
kubectl get pod mypod -o jsonpath='{.status.containerStatuses[0].containerID}'
# containerd://abc123...

# 노드에 들어가서 cgroup 확인
ls /sys/fs/cgroup/kubepods.slice/
# kubepods-besteffort.slice/
# kubepods-burstable.slice/
# kubepods-pod<UID>.slice/

cat /sys/fs/cgroup/kubepods.slice/.../memory.max
```

---

## 7. GPU는 왜 cgroup으로 제한 안 되는가 (★ 중요)

**일반 자원**: cgroup으로 커널이 직접 제어
- CPU: 스케줄러가 시간 분배
- 메모리: page allocator가 추적
- IO: block layer가 throttle

**GPU**:
- NVIDIA 드라이버가 GPU 메모리/연산을 관리
- 커널 cgroup 메커니즘은 GPU에 손이 안 닿음
- → **GPU는 cgroup이 아니라 device plugin이 처리**

```yaml
resources:
  limits:
    nvidia.com/gpu: 4   # ← cgroup 안 거침
                        # NVIDIA Device Plugin이 GPU 4장을 컨테이너에 노출
```

이게 NVIDIA Device Plugin이 별도로 필요한 이유입니다. RDMA Device Plugin도 동일한 이유.

**MIG (Multi-Instance GPU)**:
- A100/H100에서 GPU 1장을 하드웨어 레벨로 7개로 쪼갬
- 각 instance를 별도 디바이스처럼 취급
- 여전히 cgroup 아님 — NVIDIA 드라이버가 격리

---

## 8. cgroup과 systemd

**systemd**가 cgroup의 주요 관리자입니다.

```bash
# 현재 cgroup 트리 보기
systemd-cgls

# 자원 사용량 보기
systemd-cgtop
```

systemd unit을 만들면 자동으로 cgroup이 생성됨:

```ini
# /etc/systemd/system/myapp.service
[Service]
ExecStart=/usr/bin/myapp
MemoryMax=1G
CPUQuota=200%
```

이렇게 unit 파일에 적으면 systemd가 cgroup으로 변환해서 적용.

K8s도 노드 OS에 따라 **systemd cgroup driver** 또는 **cgroupfs driver** 사용 (containerd 설정).

---

## 9. 실무 시나리오

### 9.1 "Pod이 이유 없이 OOM Kill 됐다"

```bash
# 노드에서 dmesg 확인
dmesg | grep -i "killed process"
# Memory cgroup out of memory: Killed process 1234

# 또는 Pod 이벤트
kubectl describe pod mypod | grep -A 5 "Last State"
# Reason: OOMKilled
# Exit Code: 137  (= 128 + SIGKILL)
```

원인 찾기:
- Pod의 `memory.events`에서 oom_kill 카운트 확인
- 메모리 사용 패턴이 limit 가까이 갔는지 확인
- limit을 올리거나, 메모리 leak 수정

### 9.2 "Pod이 느린데 CPU 사용률은 낮다"

→ **CPU throttling**

```bash
cat /sys/fs/cgroup/.../cpu.stat
# usage_usec 1234567890
# throttled_usec 9876543210   ← throttle 당한 시간
# nr_throttled 12345
```

`throttled_usec`이 크면 limit을 너무 낮게 설정한 것. 늘리거나 제거 고려.

K8s에서 `cpu.limit` 설정이 미묘한 이유가 이것 — 평균 사용량은 낮아도 burst 시 throttle 걸려 응답 느려짐.

### 9.3 "노드에서 어떤 Pod이 메모리 많이 쓰나"

```bash
# 모든 Pod cgroup의 메모리 사용량
for f in /sys/fs/cgroup/kubepods.slice/*/memory.current; do
  echo "$f: $(cat $f)"
done | sort -t: -k2 -n
```

---

## 10. 우리 회사 환경에서

DGX 노드에서 cgroup 관점으로 보면:

```yaml
# 학습 Pod 예시
resources:
  requests:
    cpu: "16"
    memory: "128Gi"
    nvidia.com/gpu: 8           # ← cgroup 안 거침, NVIDIA device plugin
    rdma/hca_shared_devices_a: 1 # ← cgroup 안 거침, RDMA device plugin
  limits:
    cpu: "32"                    # → cpu.max = "3200000 100000"
    memory: "256Gi"              # → memory.max = 274877906944
    nvidia.com/gpu: 8
    rdma/hca_shared_devices_a: 1
```

**주의점**:
- DGX는 보통 노드 1개를 학습 Pod 1개가 전유 → CPU/메모리 limit을 노드 거의 전체로 둠
- 너무 빡빡하게 설정하면 데이터 로딩 워커들이 OOM
- GPU 작업 중 host 메모리에 prefetch도 하니 메모리는 여유 있게

---

## 11. 면접 예상 질문

### Q1. "컨테이너의 정의를 한 문장으로?"
> "리눅스 커널의 namespace로 시야를 격리하고, cgroup으로 자원량을 제한하며, OverlayFS 같은 유니온 파일시스템으로 이미지 레이어를 마운트한 프로세스 그룹입니다."

### Q2. "K8s memory limit은 cgroup으로 어떻게 적용되나요?"
> "Pod 컨테이너 cgroup의 memory.max에 limits.memory 값이 그대로 들어갑니다. 사용량이 이 값을 넘으면 커널 OOM Killer가 cgroup 안의 프로세스를 죽이고, 컨테이너는 exit code 137로 종료되며 K8s는 OOMKilled 상태로 표시합니다."

### Q3. "CPU limit을 걸면 왜 응답이 느려질 수 있나요?"
> "limits.cpu는 cgroup의 cpu.max로 매핑되어 100ms 주기당 사용 가능한 CPU 시간을 강제합니다. 평균 사용량이 낮아도 burst 순간에 한도를 넘으면 다음 주기까지 throttle되어 latency가 튑니다. cpu.stat의 throttled_usec으로 진단할 수 있습니다."

### Q4. "GPU는 왜 cgroup으로 제한 안 되나요?"
> "cgroup은 커널이 직접 관리하는 자원(CPU, 메모리, 블록IO 등)에만 적용됩니다. GPU는 NVIDIA 드라이버가 자체적으로 메모리/연산을 관리하므로 커널 cgroup 영역 밖입니다. 그래서 K8s에서는 NVIDIA Device Plugin이 별도로 GPU 디바이스 파일을 컨테이너에 노출하는 방식으로 처리합니다."

### Q5. "QoS 클래스가 왜 중요한가요?"
> "노드 메모리가 부족하면 kubelet이 Pod을 evict하는데, BestEffort → Burstable → Guaranteed 순으로 죽입니다. 핵심 워크로드는 Guaranteed로 두어야 살아남습니다. requests = limits로 설정하면 Guaranteed가 됩니다."

---

## 11.5 연계 문서

- 직전: [linux-fundamentals-deep-dive.md](linux-fundamentals-deep-dive.md) — OOM Killer 원리, QoS와 `oom_score_adj`.
- 다음: [../k8s/container-runtime-deep-dive.md](../k8s/container-runtime-deep-dive.md) — containerd가 cgroupfs/systemd driver로 cgroup을 실제 만드는 경로.
- [../k8s/multi-tenancy-scheduler-deep-dive.md](../k8s/multi-tenancy-scheduler-deep-dive.md) — ResourceQuota/Kueue가 requests/limits에 어떻게 묶이는지.
- [../k8s/security-deep-dive.md](../k8s/security-deep-dive.md) — cgroup은 격리가 아닌 "한도" 메커니즘임을 구분.

---

## 12. 한 줄 요약

> **"cgroup은 컨테이너가 실제로 쓸 수 있는 CPU/메모리/IO를 커널 차원에서 강제하는 메커니즘이다. K8s의 requests/limits는 결국 cgroupfs의 파일에 값을 쓰는 일이고, OOMKilled나 CPU throttling 같은 운영 이슈는 모두 이 메커니즘의 결과물이다. GPU/RDMA처럼 커널이 직접 다루지 않는 자원은 별도의 device plugin이 처리한다."**
