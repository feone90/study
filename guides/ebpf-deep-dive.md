# eBPF 완전 정복 - 차세대 인프라의 키워드

> **목적**: "eBPF가 뭐냐"는 면접 질문에 5분간 막힘없이 답하고, 왜 모든 인프라 회사가 이걸 외치는지 이해. 차세대 네트워크/관측/보안의 공통 기반.

---

## 0. 한 줄로

> **"커널을 재컴파일하거나 모듈 로드하지 않고, 안전하게 검증된 작은 프로그램을 커널 안에서 실행시키는 메커니즘."**

이게 왜 혁명적인지는 1장에서.

---

## 1. 왜 eBPF가 필요했나

### 1.1 전통적인 방법의 한계

리눅스 커널 기능을 확장하려면:

**옵션 A: 커널 패치**
- 코드 수정 → 커널 재컴파일 → 재부팅
- 메인라인 채택까지 수년
- 너무 무겁고 위험

**옵션 B: 커널 모듈**
- 동적 로드 가능
- 그러나 커널 권한으로 동작 → 버그 = 시스템 크래시
- 커널 버전마다 호환성 문제

### 1.2 eBPF의 답

```
[유저 공간]
  ↓ eBPF 프로그램 작성 (C 또는 고수준 언어)
  ↓ 컴파일 → eBPF 바이트코드
  ↓ bpf() syscall로 커널에 로드
[커널]
  ↓ Verifier가 안전성 검증
  ↓ JIT 컴파일 → 네이티브 머신 코드
  ↓ 정해진 hook 지점에서 실행
```

특징:
- **재부팅 불필요**
- **검증된 안전성** (verifier가 무한 루프, 메모리 위반 등 차단)
- **거의 네이티브 성능** (JIT)
- **커널을 안 건드림**

---

## 2. eBPF 작동 원리

### 2.1 hook 지점

eBPF 프로그램은 커널의 특정 이벤트가 발생할 때 실행됩니다.

| 종류 | 언제 실행 | 용도 |
|------|-----------|------|
| **kprobe** | 임의 커널 함수 진입/종료 | 트레이싱 |
| **uprobe** | 유저 공간 함수 | 앱 트레이싱 |
| **tracepoint** | 미리 정의된 커널 이벤트 | 안정적 트레이싱 |
| **XDP** | NIC에서 패킷 수신 직후 | 초고속 패킷 처리 |
| **TC** | 네트워크 큐 디스시플린 | 패킷 라우팅/필터 |
| **socket filter** | 소켓 IO | 네트워크 필터 |
| **LSM** | 보안 관련 hook | 보안 정책 |
| **perf_event** | 성능 이벤트 (CPU 사이클 등) | 프로파일링 |

### 2.2 verifier - 안전성의 핵심

eBPF 프로그램이 커널에 로드되기 전, **verifier**가 검사:

- 모든 메모리 접근이 유효한 범위 안인가
- 무한 루프 없는가 (모든 분기가 종료에 도달하나)
- 사용한 helper 함수가 적절한가
- 스택 overflow 가능성 없는가

검사 통과 못하면 로드 거부. **이래서 커널 안전이 보장됨**.

### 2.3 BPF map - 데이터 공유

eBPF 프로그램끼리, 또는 eBPF ↔ 유저공간 데이터 공유 메커니즘.

| 종류 | 용도 |
|------|------|
| `BPF_MAP_TYPE_HASH` | key-value 저장 |
| `BPF_MAP_TYPE_ARRAY` | 배열 |
| `BPF_MAP_TYPE_PERF_EVENT_ARRAY` | 이벤트 스트림 (유저로 푸시) |
| `BPF_MAP_TYPE_RINGBUF` | 링 버퍼 (효율적 이벤트) |
| `BPF_MAP_TYPE_LRU_HASH` | 자동 LRU eviction |

```
[eBPF 프로그램] ─── write ───→ [BPF map]
                                  ↑
[유저공간 모니터링 앱] ── read ───┘
```

---

## 3. eBPF 프로그램 예시

### 3.1 가장 단순한 예: 시스템 콜 카운트

```c
// count_open.c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, u32);
    __type(value, u64);
    __uint(max_entries, 1);
} count_map SEC(".maps");

SEC("tracepoint/syscalls/sys_enter_openat")
int trace_open(void *ctx) {
    u32 key = 0;
    u64 *val = bpf_map_lookup_elem(&count_map, &key);
    if (val) (*val)++;
    return 0;
}

char LICENSE[] SEC("license") = "GPL";
```

```bash
# 컴파일 & 로드
clang -target bpf -O2 -c count_open.c -o count_open.o
sudo bpftool prog load count_open.o /sys/fs/bpf/count_open

# 결과 보기
sudo bpftool map dump name count_map
```

매번 `openat` syscall이 호출될 때마다 카운터 증가. 시스템 영향 거의 없음.

### 3.2 bpftrace - eBPF의 awk

ad-hoc eBPF 프로그래밍을 쉽게.

```bash
# 1초마다 syscall TOP 5
sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_* {
    @[probe] = count();
  }
  interval:s:1 {
    print(@, 5);
    clear(@);
  }
'

# 어떤 프로세스가 어떤 파일 여는지
sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_openat {
    printf("%s -> %s\n", comm, str(args->filename));
  }
'

# TCP 연결 latency 분포
sudo bpftrace -e '
  kprobe:tcp_connect { @start[tid] = nsecs; }
  kretprobe:tcp_connect /@start[tid]/ {
    @latency = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
  }
'
```

C 안 짜고도 강력한 관측 가능.

---

## 4. eBPF 활용 영역

### 4.1 네트워킹 (Cilium)

**XDP** (eXpress Data Path): NIC에서 패킷이 커널에 들어오기 직전에 처리.

```
[NIC] → [드라이버] → [XDP eBPF] → 커널 네트워크 스택
                          ↓
                      [DROP / PASS / REDIRECT]
```

처리량:
- iptables: ~수 Gbps에서 한계
- XDP: 라인 레이트 (수십 Gbps)

**Cilium**: eBPF 기반 K8s CNI.
- kube-proxy 대체 (iptables 안 씀, 더 빠름)
- Network Policy를 eBPF로
- L7 (HTTP, gRPC) 정책도 가능

### 4.2 관측 (Observability)

**Hubble** (Cilium 자매): 네트워크 흐름 시각화
```
hubble observe
# Apr 13 12:34:56  default/pod-a → default/pod-b  HTTP/1.1 GET /api 200
```

**Pixie**: K8s 자동 관측
- HTTP, MySQL, Redis 등 프로토콜 자동 디코드
- 코드 변경 없음

**Pyroscope/Parca**: continuous profiling
- 모든 프로세스 stack sampling
- flame graph 자동 생성

### 4.3 보안 (Security)

**Falco**: 런타임 보안
```
규칙: "/etc/passwd 읽으려는 시도"
이벤트: 컨테이너 X가 /etc/passwd open()
→ 알람!
```

**Tracee**, **Tetragon**: 비슷한 컨셉.

eBPF로 모든 syscall을 관측 → 이상 행위 탐지.

### 4.4 성능

**bpftrace**, **bcc**: 시스템 디버깅
- CPU profiling
- IO latency 분포
- TCP 재전송 추적
- 함수 호출 추적

기존에는 strace, perf 등으로 분산되어 있던 도구들을 eBPF가 통합.

---

## 5. eBPF가 게임 체인저인 이유

### 5.1 vs 전통적 모니터링

```
전통:
  앱이 메트릭 export → exporter → Prometheus
  → 앱 변경 필요, latency 영향

eBPF:
  커널이 모든 syscall, 패킷 관측 → 메트릭화
  → 앱 변경 무관, 거의 0 오버헤드
```

### 5.2 vs iptables

```
iptables:
  규칙 1만 개 → 매 패킷마다 모두 평가 → 느려짐

eBPF (Cilium):
  hash map lookup → O(1) → 라인 레이트
```

### 5.3 vs 커널 모듈

```
커널 모듈:
  버그 → 시스템 panic
  
eBPF:
  verifier가 안전성 검증 → 안전
```

### 5.4 vs 유저 공간 도구

```
유저 공간:
  syscall로 데이터 가져옴 → context switch 비용

eBPF:
  커널 안에서 실행 → context switch 없음
```

---

## 6. eBPF 생태계

### 6.1 도구

| 도구 | 용도 |
|------|------|
| **bpftrace** | ad-hoc 트레이싱 (awk 같음) |
| **bcc** | Python으로 eBPF 프로그래밍 |
| **libbpf** | C 라이브러리 (현대적) |
| **bpftool** | eBPF 프로그램/맵 관리 |
| **CO-RE (Compile Once, Run Everywhere)** | 커널 버전 호환성 |

### 6.2 K8s 프로젝트

| 프로젝트 | 영역 |
|---------|------|
| **Cilium** | CNI, kube-proxy 대체 |
| **Hubble** | 네트워크 관측 |
| **Tetragon** | 보안 (Cilium의 자매) |
| **Pixie** | 자동 관측 |
| **Falco** | 보안 모니터링 |
| **Pyroscope/Parca** | continuous profiling |

### 6.3 회사

대부분의 대형 인프라 회사가 eBPF에 투자:
- Cloudflare (트래픽 처리)
- Meta/Facebook (데이터센터 네트워크)
- Netflix (성능 분석)
- Google (보안)

---

## 7. CO-RE (Compile Once, Run Everywhere)

### 7.1 문제

eBPF 프로그램은 커널 자료구조(struct task_struct 등)를 참조.
커널 버전마다 자료구조가 미묘하게 다름 → 매 커널마다 재컴파일 필요했음.

### 7.2 해결

- **BTF (BPF Type Format)**: 커널이 자기 자료구조 정보를 포함
- **libbpf의 CO-RE**: 컴파일 시 참조를 "추상적"으로, 런타임에 실제 오프셋 patch

→ 한 번 빌드한 eBPF가 다양한 커널 버전에서 동작.

이게 eBPF가 본격적으로 production-ready가 된 이유.

---

## 8. K8s에서 eBPF 적용 시나리오

### 8.1 Cilium으로 CNI 교체

```
Calico (전통적 CNI) → Cilium (eBPF)

이점:
- kube-proxy 제거 (iptables 없음)
- Network Policy 빠름
- L7 정책 (HTTP method/path 단위)
- Hubble로 모든 통신 시각화
```

단점:
- 학습 곡선 가파름
- eBPF 친화적 커널 필요 (5.x+)

### 8.2 관측 강화

```
기존 Prometheus + Loki + Jaeger 위에
+ Hubble (네트워크 플로우)
+ Pyroscope (continuous profiling)
+ Falco (보안)
```

각각이 eBPF로 거의 0 오버헤드.

### 8.3 우리 환경 적용 가능성

- Calico 쓰는 회사라면 → Cilium 검토 가치
- 다만 IB는 RDMA라 eBPF 영역 밖 (커널 우회)
- 하지만 일반 통신(eth0)은 eBPF 혜택 가능
- **관측/보안 도구**는 즉시 도입 가능 (Hubble, Falco 등)

---

## 9. 한계와 주의점

### 9.1 verifier 제약

- 프로그램 크기 제한 (이전 4096 instruction, 이제 1M)
- 무한 루프 못 만듦 (bounded loop만)
- 모든 메모리 접근에 bounds check 필요

복잡한 로직은 어려움. 보통 데이터 수집에 집중하고 처리는 유저공간에서.

### 9.2 커널 의존성

- 5.x+ 커널 권장
- BTF 지원 여부 확인 (CO-RE 위해)
- 일부 helper 함수는 새 커널에만 있음

### 9.3 디버깅 어려움

- 커널 안에서 실행 → printf 디버깅 한정적 (`bpf_printk`)
- verifier 거부 메시지 해석 어려움

---

## 10. 실습: 시스템 호출 관측

```bash
# bpftrace 설치
sudo apt install bpftrace

# 가장 많이 호출되는 syscall TOP 10
sudo bpftrace -e '
  tracepoint:raw_syscalls:sys_enter {
    @[comm] = count();
  }
  interval:s:5 {
    printf("=== top 10 ===\n");
    print(@, 10);
    clear(@);
  }
'
# 5초마다 출력

# 어떤 프로세스가 디스크 IO를 일으키나
sudo bpftrace -e '
  tracepoint:block:block_rq_issue {
    @[comm] = sum(args->bytes);
  }
'

# TCP 재전송 발생 추적
sudo bpftrace -e '
  kprobe:tcp_retransmit_skb {
    @[kstack] = count();
  }
'
```

직접 해보면 eBPF의 위력이 와닿음.

---

## 11. 면접 예상 질문

### Q1. "eBPF를 한 문장으로 설명해보세요."
> "재부팅이나 커널 모듈 없이, verifier로 안전성이 검증된 작은 프로그램을 커널 hook 지점에서 실행시키는 메커니즘입니다. 거의 네이티브 성능으로 네트워킹/관측/보안 영역에서 커널을 안전하게 확장할 수 있어, Cilium·Falco·Pyroscope 같은 차세대 인프라의 공통 기반이 되었습니다."

### Q2. "Cilium이 Calico보다 좋은 점은?"
> "Calico는 iptables 기반이라 규칙이 많아질수록 패킷마다 성능이 떨어집니다. Cilium은 eBPF로 hash map lookup이라 O(1)이고, kube-proxy까지 대체해 iptables를 거의 안 씁니다. L7 정책(HTTP method/path)도 가능하고, Hubble로 모든 Pod 통신을 시각화할 수 있습니다. 단점은 학습 곡선과 5.x+ 커널 요구입니다."

### Q3. "eBPF가 안전한 이유는?"
> "커널에 로드되기 전 verifier가 모든 메모리 접근의 bounds check, 무한 루프 없음, 사용 가능한 helper 함수만 호출하는지를 검증합니다. 통과 못하면 로드 거부됩니다. 그래서 검증된 eBPF 프로그램은 커널 모듈과 달리 시스템 panic을 일으킬 수 없습니다."

### Q4. "관측 도구로 eBPF를 쓰는 이점은?"
> "전통적 관측은 앱이 메트릭을 export하거나 sidecar가 트래픽을 가로채야 합니다 — 코드 변경 또는 성능 영향이 있습니다. eBPF는 커널 hook에서 직접 syscall과 패킷을 관측해 앱은 전혀 모르고 오버헤드도 거의 없습니다. Pixie 같은 도구는 K8s에 설치만 하면 모든 HTTP/gRPC 호출이 자동으로 보입니다."

### Q5. "RDMA 환경에서 eBPF 한계는?"
> "RDMA는 커널 네트워크 스택을 우회하므로 eBPF가 그 트래픽을 못 봅니다. eBPF는 커널 hook 지점에서만 동작하는데, RDMA 패킷은 NIC이 직접 GPU/유저 메모리로 DMA합니다. 그래서 IB 트래픽 모니터링은 infiniband_exporter나 NIC 카운터로, eBPF는 일반 이더넷 영역(eth0)과 syscall 수준 관측에 활용해야 합니다."

---

## 12. 한 줄 요약

> **"eBPF = 검증된 작은 프로그램을 커널에 안전하게 주입해 네트워킹·관측·보안을 거의 0 오버헤드로 확장하는 기술. Cilium(CNI), Hubble(관측), Falco(보안), Pyroscope(프로파일링) 같은 차세대 K8s 인프라의 공통 기반이며, RDMA처럼 커널을 우회하는 영역만 빼면 모든 곳에 적용 가능하다."**
