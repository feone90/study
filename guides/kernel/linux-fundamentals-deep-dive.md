# Linux 기초 - DevOps/SRE의 진짜 시작점

> **목적**: 시스템 콜, /proc, /sys, systemd, IRQ affinity, ulimit 등 "리눅스 시스템을 진짜로 이해하는" 데 필요한 기초. DGX 같은 고성능 시스템 튜닝 시 반드시 알아야 할 것들.

---

## 0. 왜 이걸 알아야 하나

K8s, Docker, Ceph, eBPF — 모두 결국 **리눅스 위에서 도는 추상화**.
근본을 모르면:
- 트러블슈팅 시 추측만 함
- 성능 튜닝 못 함
- 새 도구가 나오면 처음부터 학습

리눅스를 이해하면 모든 인프라 도구가 "아 그래서 그렇구나" 됨.

---

## 1. 시스템 콜 (Syscall)

### 1.1 무엇

유저 공간 프로그램이 커널 기능을 호출하는 표준 인터페이스.

```
[App] (유저 공간)
   ↓ syscall(SYS_read, fd, buf, count)
[CPU mode switch: ring 3 → ring 0]
   ↓
[커널 syscall handler]
   ↓ 실제 작업 (디스크 IO 등)
   ↓ 결과 반환
[CPU mode switch: ring 0 → ring 3]
   ↓
[App에 결과 전달]
```

### 1.2 흔한 syscall들

| syscall | 용도 |
|---------|------|
| `read`, `write` | 파일/소켓 IO |
| `open`, `close` | 파일 디스크립터 관리 |
| `mmap` | 메모리 매핑 |
| `fork`, `clone`, `execve` | 프로세스 생성 |
| `socket`, `connect`, `bind`, `accept` | 네트워크 |
| `epoll_create`, `epoll_wait` | 이벤트 폴링 |
| `ioctl` | 디바이스 제어 |

전체 ~330개. `man 2 syscalls`로 목록.

### 1.3 strace - syscall 추적

```bash
strace ls
# execve("/usr/bin/ls", ["ls"], 0x...) = 0
# brk(NULL) = 0x...
# openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
# ...

# 특정 syscall만
strace -e trace=openat,read ls

# 시간 측정
strace -c ls
# % time     seconds  usecs/call    calls    errors syscall
# ...
```

트러블슈팅의 첫 단계: "이 프로세스가 뭘 하고 있나?" → strace.

### 1.4 vDSO와 vsyscall

자주 호출되는 일부 syscall(`gettimeofday` 등)은 mode switch 비용이 아까움.

→ 커널이 유저공간에 매핑한 코드(**vDSO**)로 처리. 진짜 syscall 안 함.

`ldd /bin/ls`에서 `linux-vdso.so.1`이 보이는 게 그것.

---

## 2. /proc과 /sys

### 2.1 /proc - 프로세스/커널 정보

가상 파일시스템. 디스크에 없고 커널이 동적 생성.

```bash
ls /proc/
# 1/    2/    1234/   self/   cpuinfo  meminfo  loadavg  ...
#         ↑      ↑
#       PID 디렉터리

cat /proc/cpuinfo
# CPU 정보

cat /proc/meminfo
# MemTotal:  131072000 kB
# MemFree:    50000000 kB
# ...

ls /proc/1234/
# cmdline   environ  fd/   maps   status   stat   ...

cat /proc/1234/status
# Name: nginx
# State: S
# Pid: 1234
# VmRSS: 50000 kB     ← 실제 메모리 사용량
# ...

ls /proc/1234/fd/
# 0 -> /dev/null
# 1 -> /var/log/nginx.log
# 2 -> /var/log/nginx.err
# 3 -> socket:[12345]   ← 열린 파일/소켓 모두 보임
```

### 2.2 /sys - 디바이스/커널 객체

```bash
ls /sys/class/
# net/   block/   gpu_drm/   infiniband/   ...

ls /sys/class/net/eth0/
# address (MAC)   mtu   speed   statistics/   ...

cat /sys/class/infiniband/mlx5_0/ports/1/state
# 4: ACTIVE       ← IB 포트 상태

cat /sys/class/infiniband/mlx5_0/ports/1/counters/port_rcv_data
# 1234567890     ← 수신 데이터 카운터 (RDMA 모니터링!)

ls /sys/fs/cgroup/
# cgroup 트리 (이미 봤음)
```

### 2.3 활용

운영 자동화 스크립트의 절대다수가 /proc, /sys를 읽음. node-exporter도 결국 여기 데이터를 메트릭화.

---

## 3. 파일 디스크립터 (FD)

### 3.1 무엇

프로세스가 열어둔 파일/소켓/파이프 등을 가리키는 정수.

```c
int fd = open("/etc/passwd", O_RDONLY);  // fd = 3
read(fd, buf, 100);
close(fd);
```

기본:
- 0: stdin
- 1: stdout
- 2: stderr
- 3+: open한 것들

### 3.2 FD limit

```bash
ulimit -n
# 1024     ← soft limit (기본 낮음)

ulimit -n 65536   # 늘림
```

K8s에서:
```yaml
# Pod 안에서는 컨테이너 런타임의 limit 적용
# /etc/security/limits.conf 또는 systemd 설정
```

대용량 트래픽 서버는 FD limit이 자주 병목. 모니터링 필수.

### 3.3 lsof - 열린 파일 보기

```bash
lsof -p 1234        # 프로세스 1234가 연 파일들
lsof -i :80         # 80포트 사용 중인 프로세스
lsof /var/log/app.log  # 이 파일 누가 열었나
```

"디스크 가득찼는데 df로 안 보임" → lsof로 unlinked 파일 잡기.

---

## 4. 프로세스와 스레드

### 4.1 차이

- **프로세스**: 독립된 메모리 공간 (fork)
- **스레드**: 같은 메모리 공유 (clone with CLONE_VM)

리눅스에선 둘 다 task_struct로 관리. 차이는 어떤 자원을 공유하느냐.

### 4.2 fork & exec

```c
pid_t pid = fork();      // 자식 프로세스 생성 (메모리 복사)
if (pid == 0) {
    execve("/bin/ls", ...);  // 자기 자신을 ls로 교체
}
```

새 프로세스 시작 = fork → exec.

### 4.3 ps 활용

```bash
ps aux                       # 모든 프로세스
ps -ef --forest              # 트리 형태
ps -o pid,user,%cpu,%mem,cmd # 특정 컬럼

# CPU 많이 쓰는 TOP 10
ps aux --sort=-%cpu | head -10
```

### 4.4 시그널

```bash
kill -SIGTERM 1234   # 부드럽게 종료
kill -SIGKILL 1234   # 강제 (= -9)
kill -SIGHUP  1234   # 보통 설정 reload

# 시그널 목록
kill -l
```

K8s가 Pod 종료 시:
1. SIGTERM 보냄
2. terminationGracePeriodSeconds(기본 30s) 대기
3. 안 죽으면 SIGKILL

앱이 SIGTERM 받으면 graceful shutdown 해야 함.

---

## 5. 메모리 관리

### 5.1 가상 메모리

각 프로세스는 자기만의 가상 주소 공간을 가짐.

- 32-bit: 4GB (유저 3GB + 커널 1GB, 일반적 split)
- 64-bit x86_64 (4-level paging): 유저 128TB + 커널 128TB
- 64-bit x86_64 (5-level paging, 커널 4.14+, `la57`): 유저 64PB
- ARM64도 비슷하며 `CONFIG_ARM64_VA_BITS`로 결정

```
[프로세스 가상 주소]
   ↓ MMU + 페이지 테이블
[실제 물리 메모리 또는 swap]
```

### 5.2 메모리 종류

```bash
cat /proc/1234/status | grep -i vm
# VmPeak: 가상 주소 사용 최대치
# VmSize: 현재 가상 주소
# VmRSS:  실제 물리 메모리 (Resident Set Size)
# VmData: 데이터 세그먼트
```

`top`/`ps`의 `RES`/`%MEM`이 VmRSS.

### 5.3 page cache vs anonymous

- **anonymous**: 프로세스가 malloc한 메모리
- **page cache**: 파일 IO 캐시

`free -h`:
```
              total   used   free   shared   buff/cache
Mem:          128G     50G    20G    1G       58G
```

`buff/cache`는 **언제든 해제 가능**한 캐시. "메모리 부족"으로 오인 금지.

### 5.4 swap

물리 메모리 부족 시 디스크로 페이지 내림. 매우 느림.

K8s에선 보통 swap 비활성화 (`swapoff -a`).
- 컨테이너 메모리 limit이 부정확해짐
- 성능 예측 어려움

### 5.5 Huge Pages

기본 페이지: 4KB. **HugePages**: 2MB 또는 1GB.

```bash
# 설정
echo 1024 > /proc/sys/vm/nr_hugepages
# 1024 * 2MB = 2GB hugepages
```

DB, GPU/RDMA 워크로드에서 TLB miss 줄여서 성능 향상. NCCL, RDMA가 활용.

### 5.6 THP (Transparent Huge Pages)

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never
```

커널이 투명하게 2MB로 묶어주는 기능. 장점: 앱 수정 없이 TLB hit 향상.
단점: **RDMA/DPDK/DB(Redis, MongoDB) 환경에선 잠재적 지연 스파이크** (khugepaged 압축, memory fragmentation).
→ DGX/GPU 노드는 보통 `madvise` 또는 `never`로 설정하고, 명시적 HugePages만 사용.

### 5.7 OOM Killer

물리 메모리 + swap 모두 고갈 시 커널이 프로세스를 죽여 시스템 보호.

```bash
# 각 프로세스의 OOM 점수 (높을수록 먼저 죽음)
cat /proc/1234/oom_score
cat /proc/1234/oom_score_adj   # -1000 ~ +1000, 사람이 조정

# OOM 절대 보호 (init/핵심 데몬)
echo -1000 > /proc/1234/oom_score_adj
```

- kubelet은 Pod QoS(Guaranteed / Burstable / BestEffort)에 따라 `oom_score_adj` 자동 설정 → BestEffort가 먼저 죽음.
- cgroup v2는 `memory.oom.group=1`이면 컨테이너 내 **전체 프로세스가 한 번에** 죽음 (좀비 방지).
- 진단: `dmesg | grep -i "killed process"` 또는 `journalctl -k`.

---

## 6. systemd

### 6.1 무엇

리눅스의 init 시스템. PID 1로 동작, 모든 서비스 관리.

### 6.2 unit

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network.target

[Service]
ExecStart=/usr/bin/myapp
Restart=on-failure
User=myuser
MemoryMax=2G          ← cgroup으로 변환됨
CPUQuota=200%

[Install]
WantedBy=multi-user.target
```

### 6.3 명령

```bash
systemctl start myapp
systemctl stop myapp
systemctl restart myapp
systemctl status myapp
systemctl enable myapp     # 부팅 시 자동 시작
systemctl daemon-reload    # unit 파일 변경 후

journalctl -u myapp        # 로그
journalctl -u myapp -f     # 실시간
journalctl -u myapp --since "1 hour ago"
```

### 6.4 systemd-cgls / systemd-cgtop

cgroup 트리와 자원 사용량 시각화.

```bash
systemd-cgtop
# Path                            Tasks  %CPU   Memory  Input/s
# /                                 2547  50.0    20.0G        -
# /system.slice                      300  10.0     2.0G        -
# /system.slice/containerd.service   200   8.0     5.0G        -
# /kubepods.slice                   2000  30.0    13.0G        -
```

K8s 노드에서 `systemd-cgls /kubepods.slice`로 Pod별 cgroup 트리 보기 가능.

---

## 7. IRQ (Interrupt Request)와 affinity

### 7.1 IRQ란

NIC, 디스크 등 디바이스가 CPU에 "할 일 생겼다" 알림.

```bash
cat /proc/interrupts
#            CPU0    CPU1    CPU2    ...
# 24:  100000     0      0          enp1s0
# 25:       0  50000      0         enp2s0
```

### 7.2 IRQ affinity

특정 IRQ를 특정 CPU에 고정.

```bash
# IRQ 24를 CPU 0,1에 고정
echo 3 > /proc/irq/24/smp_affinity   # 0b11

# 자동 균형 (irqbalance 데몬)
systemctl status irqbalance
```

### 7.3 왜 중요한가

NIC IRQ가 한 CPU에 몰리면:
- 그 CPU만 100%
- 패킷 처리 병목

해결: 멀티큐 NIC + IRQ affinity 분산.

DGX 같은 고성능 시스템:
- IB NIC 8개 → IRQ 8개 → CPU 8개에 분산
- NUMA 인식 (CPU와 NIC이 같은 NUMA 노드)

### 7.4 RPS / RSS

NIC 하나가 여러 큐 → 각 큐가 다른 CPU로 패킷 분배. 멀티코어 활용.

---

## 8. NUMA (Non-Uniform Memory Access)

### 8.1 무엇

멀티 소켓 서버에서 CPU별로 가까운 메모리/디바이스가 다름.

```
[Socket 0]        [Socket 1]
  CPU 0~31          CPU 32~63
  ↓                  ↓
RAM 0 (가까움)    RAM 1 (가까움)
GPU 0~3           GPU 4~7
HCA 0~3           HCA 4~7
```

CPU 0이 RAM 1에 접근하면 socket 간 통신 → 느림.

### 8.2 확인

```bash
numactl --hardware
# node 0 cpus: 0-31
# node 0 size: 128GB
# node 1 cpus: 32-63
# node 1 size: 128GB

lscpu | grep NUMA
```

### 8.3 NUMA-aware 실행

```bash
# 프로세스를 특정 NUMA 노드에 고정
numactl --cpunodebind=0 --membind=0 myapp
```

PyTorch/NCCL은 자동으로 NUMA 인식 (대부분).

DGX 환경:
- GPU와 IB NIC을 같은 NUMA 노드에 두어야 GPUDirect 효과 최대
- K8s에서 Topology Manager로 자동화 가능

---

## 9. ulimit / rlimit

```bash
ulimit -a
# core file size       0
# data seg size        unlimited
# file size            unlimited
# max locked memory    65536       ← RDMA 사용 시 unlimited 필요!
# max memory size      unlimited
# open files           1024        ← 자주 늘려야 함
# pipe size            8
# POSIX message queues 819200
# real-time priority   0
# stack size           8192
# cpu time             unlimited
# max user processes   100000
```

### 9.1 RDMA 환경에서

```
max locked memory: unlimited
```

이 설정이 없으면 RDMA가 메모리 pin을 못 해서 실패. 우리 회사도 DGX 노드에 적용되어 있을 것.

### 9.2 K8s에서

```yaml
# Pod에서 ulimit 설정 어려움 (containerd 설정 필요)
# 보통 노드 OS 레벨에서 설정
```

---

## 10. 네트워크 성능 튜닝

### 10.1 sysctl 파라미터

```bash
# 현재 값
sysctl net.core.rmem_max
# net.core.rmem_max = 212992

# 변경
sysctl -w net.core.rmem_max=134217728

# 영구
echo "net.core.rmem_max = 134217728" >> /etc/sysctl.conf
```

자주 만지는 것:
```
net.core.rmem_max          # 소켓 read 버퍼 최대
net.core.wmem_max          # 소켓 write 버퍼 최대
net.core.netdev_max_backlog  # NIC 큐 크기
net.ipv4.tcp_rmem          # TCP read 버퍼
net.ipv4.tcp_wmem          # TCP write 버퍼
net.ipv4.tcp_congestion_control  # 혼잡 제어 (cubic, bbr 등)
```

### 10.2 점보 프레임

```bash
ip link set eth0 mtu 9000
```

### 10.3 GSO/TSO/GRO

NIC offload 기능. 큰 패킷 처리를 NIC이 분할/조립.

```bash
ethtool -k eth0   # offload 설정 보기
```

대부분 켜져 있어야 성능 좋음.

---

## 11. 디스크 / IO 성능

### 11.1 IO 스케줄러

```bash
cat /sys/block/sda/queue/scheduler
# noop deadline [cfq]   ← 대괄호가 현재

# 변경
echo "deadline" > /sys/block/sda/queue/scheduler
```

NVMe SSD는 보통 `none` (다중 큐 자체가 분산).

### 11.2 iostat / iotop

```bash
iostat -x 1
# Device  rrqm/s wrqm/s r/s w/s rkB/s wkB/s ... %util
# sda     ...

iotop
# 어떤 프로세스가 디스크 IO 일으키나
```

### 11.3 fio - 성능 측정

```bash
fio --name=test --filename=/data/test --size=1G \
    --rw=randread --bs=4k --iodepth=32 --numjobs=4 --time_based --runtime=30
```

디스크 진단의 표준.

---

## 12. 자주 쓰는 디버깅 명령 모음

```bash
# 시스템 부하
top, htop, atop
uptime           # load average

# 메모리
free -h
vmstat 1

# CPU
mpstat 1
sar -u 1 10

# 디스크
df -h            # 사용량
du -sh /var/log/* | sort -rh
iostat -x 1
iotop

# 네트워크
ss -tunap
ip a / ip route
tcpdump
mtr / traceroute

# 프로세스/시스템 콜
ps auxf
strace -p <pid>
lsof -p <pid>

# 커널 로그
dmesg -T | tail
journalctl -k

# 시스템 정보
uname -a
lsb_release -a
lscpu, lsmem, lsblk, lsusb, lspci

# 사용자/권한
who, w
last           # 로그인 기록
sudo -l        # 내 sudo 권한

# 자원 한계
ulimit -a
cat /proc/sys/fs/file-max
```

---

## 13. 우리 회사 환경에서 알아야 할 것

### 13.1 DGX 노드 튜닝 포인트

```
- max locked memory: unlimited (RDMA용)
- HugePages 활성화 (성능)
- IRQ affinity (NIC을 NUMA-local CPU에)
- swap 비활성화
- MTU 점보 프레임 (9000)
- net.core.rmem/wmem_max 큼
```

### 13.2 트러블슈팅 흐름

```
"Pod 성능 이상해요"
   ↓
1. kubectl describe pod (events)
2. kubectl logs
3. 노드에 들어가서:
   - top: CPU/메모리
   - dmesg: OOM, 하드웨어 에러
   - nvidia-smi: GPU 상태
   - ibstatus: IB 상태
4. /proc, /sys 직접 확인
5. strace로 프로세스 실시간
6. tcpdump/perf 등 깊이 있는 도구
```

### 13.3 알아두면 좋은 것

- DGX OS는 보통 이미 튜닝됨 (MDS가 셋업)
- 그래도 이슈 발생 시 어디를 봐야 하는지 알아야 함
- 노드 직접 접근 권한 있다면 위 명령들로 진단

---

## 14. 면접 예상 질문

### Q1. "프로세스가 갑자기 죽었어요. 어떻게 진단하나요?"
> "먼저 dmesg와 journalctl로 OOM Killer 활동 확인. 그 다음 /proc/<pid>/status (이미 죽었다면 어렵지만), 코어 덤프 활성화, strace로 syscall 추적. K8s 환경이면 Pod 이벤트와 Last State, 종료 코드(137=OOM, 139=SEGV 등)로 일차 진단합니다."

### Q2. "swap을 K8s 노드에서 끄는 이유는?"
> "컨테이너 메모리 limit이 cgroup의 memory.max로 적용되는데, swap이 있으면 limit 초과해도 디스크로 swap 되어 OOM 트리거가 모호해집니다. 또 swap된 페이지 접근은 매우 느려 학습/서빙 latency가 예측 불가능해집니다. K8s가 노드 검증 시 swap 활성화면 경고하는 이유입니다."

### Q3. "/proc/cpuinfo와 /sys/devices/system/cpu/의 차이는?"
> "/proc/cpuinfo는 사람이 읽기 좋은 텍스트 형식의 종합 정보, /sys/devices/system/cpu/는 디바이스별 세분화된 sysfs 인터페이스입니다. CPU governor 변경, online/offline 토글, NUMA 정보 등 세밀한 제어는 /sys로 합니다."

### Q4. "NUMA가 ML 워크로드에 왜 중요한가요?"
> "DGX는 멀티 소켓 시스템이고 GPU와 IB NIC이 특정 NUMA 노드에 붙어 있습니다. 학습 프로세스가 GPU와 다른 NUMA의 CPU에서 돌면 GPUDirect/RDMA 성능이 떨어지고 socket 간 메모리 접근 latency가 추가됩니다. K8s Topology Manager로 GPU/CPU/메모리를 같은 NUMA 노드에 묶어 할당하는 정책이 권장됩니다."

### Q5. "ulimit max locked memory가 RDMA에서 왜 중요한가요?"
> "RDMA는 NIC이 유저 메모리에 직접 DMA하기 때문에 그 메모리가 swap 되면 안 됩니다. 그래서 메모리를 pin(lock)해야 하는데 이게 max locked memory(memlock) limit에 걸립니다. 기본값(64KB 정도)으로는 RDMA가 즉시 실패합니다. unlimited 또는 충분히 크게 설정해야 하고, 컨테이너에서는 securityContext나 containerd 설정으로 전달해야 합니다."

### Q6. "strace로 디버깅한 경험을 말해보세요."
> "예: 'API 응답이 가끔 5초 걸린다' 이슈에서, strace로 해당 프로세스를 attach 했더니 connect() syscall이 3초간 block되는 게 보였습니다. DNS 해석 실패로 fallback 서버를 시도하면서 timeout이 누적된 것이 원인이었습니다. 코드 변경 없이 시스템 콜 레벨에서 봤기 때문에 빠르게 잡았습니다." (자기 경험으로 대체 가능)

---

## 14.5 연계 문서

- 다음 단계: [cgroup-deep-dive.md](cgroup-deep-dive.md) — 여기서 본 `systemd-cgls`, `MemoryMax`가 어떻게 리소스 격리로 이어지는지.
- 네트워크 튜닝 상세: [network-deep-dive.md](network-deep-dive.md) — `sysctl net.core.*`, GSO/TSO, RPS/RSS 커널 경로.
- RDMA memlock·HugePages가 실제 NCCL에 어떻게 쓰이는지: [../hw/gpu-gpudirect-deep-dive.md](../hw/gpu-gpudirect-deep-dive.md), [../hw/nccl-collective-deep-dive.md](../hw/nccl-collective-deep-dive.md).
- 회사 통합 시나리오: [../integration/dgx-ib-multinode-training-guide.md](../integration/dgx-ib-multinode-training-guide.md).

---

## 15. 한 줄 요약

> **"리눅스를 안다는 건 syscall, /proc, /sys, cgroup, namespace, systemd, IRQ affinity, NUMA 같은 기본 메커니즘이 손에 익었다는 것. K8s/Docker/Ceph는 이 위의 추상화일 뿐, 진짜 트러블슈팅과 성능 튜닝은 항상 리눅스 기초로 돌아온다. DGX 같은 고성능 환경에서는 max locked memory, hugepages, NUMA, IRQ affinity 같은 OS 튜닝이 NCCL/RDMA 성능에 직결된다."**
