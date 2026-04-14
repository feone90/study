# 병렬 파일시스템 딥다이브 — Lustre / Weka / GPFS vs CephFS

> **목적**: 대규모 학습에서 **스토리지가 왜 병목이 되는가**, 현대차 규모에서 선택되는 **Lustre/Weka/GPFS**와 회사의 **CephFS**의 구조·성능·운영 차이를 비교.
> **선행**: [ceph-storage-deep-dive.md](ceph-storage-deep-dive.md)
> **연결**: [../integration/llm-training-workload-deep-dive.md](../integration/llm-training-workload-deep-dive.md)

---

## 0. 문제 — 학습 스토리지의 특성

### 0.1 워크로드 요구

| 단계 | 패턴 | 요구 |
|------|------|------|
| Dataset read | random 작은 파일 수백만 | 높은 **IOPS**, 작은 file open 지연 |
| Checkpoint write | 거대 파일 (수십~수백 GB), 동시 N-way | **aggregated bandwidth**, consistency |
| Model weights 배포 | 순차 대용량 read, 1 → N 노드 | cache hit, read-only 최적화 |
| Log / metric | 소용량 append | 순간 burst |

### 0.2 CephFS가 한계로 가는 지점

- MDS가 **메타데이터 단일 경로**. 수백만 파일 오픈은 `cap` 부하.
- HDD pool 위 CephFS는 **tail latency**가 비결정적.
- 멀티 노드 동시 쓰기 checkpoint는 **RADOS pool** 전체가 flush를 기다림.

→ 수백 GPU 스케일 LLM 학습에서 **Lustre/Weka/GPFS**가 기본 선택인 이유.

---

## 1. Lustre

### 1.1 아키텍처

```
Clients                       ┌─▶ MDS/MDT (메타데이터)
  │ LNet (RDMA/TCP)           │
  ▼                            ├─▶ OSS 1 ─ OST 1 ─ OST 2
Lustre Network  ───────────────┤
                               ├─▶ OSS 2 ─ OST 3 ─ OST 4
                               └─▶ ...
```

- **MGS** (Management): 설정 서버.
- **MDS/MDT** (Metadata Server / Target): 디렉토리·권한·inode.
- **OSS/OST** (Object Storage Server / Target): 실제 파일 블록(object).
- **LNet**: 고성능 Lustre network layer — InfiniBand, Omni-Path, TCP.
- **Clients**: 커널 모듈 `lustre.ko` + `lnet.ko`.

### 1.2 파일이 저장되는 방식

- 파일을 **stripe**로 쪼개 여러 OST에 분산 (`lfs setstripe -c 4`).
- 클라이언트는 MDS에 메타데이터만 묻고, **OSS에 병렬 RDMA**.
- 결과: 총 대역폭 = stripe × OSS 대역폭.

### 1.3 장점

- **대역폭 선형 확장** — OSS 노드 추가하면 비례 증가.
- RDMA native — H100 클러스터에 최적.
- 사실상 HPC 표준.

### 1.4 단점

- 운영 난이도: **커널 모듈 고정**, 버전 업그레이드 = 재부팅.
- MDS는 단일 장애점 기본(HA 설정 가능하지만 복잡).
- 작은 파일 수백만은 MDS가 병목.

### 1.5 튜닝 레버

- `lfs setstripe -c <N>` — stripe 수.
- `lfs migrate` — OST 간 재배치.
- `osc.*.max_rpcs_in_flight` — client RPC 병렬.
- `lctl set_param osc.*.checksums=0` — RDMA만 쓰는 환경에서 CRC off.

---

## 2. Weka

### 2.1 차별점

**NVMe-oF + 사용자 공간 스택** — Lustre가 커널 모듈이라면 Weka는 대부분 **userspace**.

- 모든 서버 노드가 **storage + client** 하이브리드.
- NVMe를 **shared-nothing cluster**로 묶음.
- 데이터 분산: Distributed Erasure Coding (4+2, 8+2 등).
- Metadata도 분산 (Lustre MDS 단일점 문제 없음).

### 2.2 성능

- **수 TB/s aggregated bandwidth**, **수억 IOPS**.
- 작은 파일 수백만도 거뜬 (메타 분산).
- 지연 ~100μs 수준.

### 2.3 장점

- 설치/운영 간단 (Lustre 대비).
- Metadata scale-out.
- Tiering (NVMe hot → S3 cold).

### 2.4 단점

- **상용 (유료)**. 라이선스 모델.
- POSIX 준수 수준은 tuning에 의존.

### 2.5 현대차급에서 자주 보이는 이유

- 벤더 지원 + SLA.
- GPU 클러스터 통합 레퍼런스 많음.
- 운영 인력 HPC 전문성 낮아도 가능.

---

## 3. GPFS (IBM Spectrum Scale)

### 3.1 특징

- IBM HPC 전통. AIX/Linux 지원.
- **shared-disk** 모델 — 블록 단위 공유.
- 분산 락 매니저.
- POSIX 준수 높음.
- AFM (Active File Management)로 사이트 간 cache.

### 3.2 단점

- 라이선스 비용 높음.
- NVIDIA/AI 스택과의 레퍼런스는 Lustre/Weka 대비 적음.

---

## 4. CephFS (회사 현황)

### 4.1 구조 (간략)

```
Client kernel (ceph.ko) 또는 ceph-fuse
  │
  ▼
MON (monitor) — cluster map
MDS — metadata
OSD × N — RADOS object store
  │
 Pool (CephFS data + metadata)
```

### 4.2 학습 워크로드 적합성

- **메타 스케일**: MDS single active가 기본. 다중 active MDS 지원되지만 운영 난이도↑.
- **대역폭**: OSD 수에 비례하되 HDD pool은 한계 명확.
- **latency**: SSD RGW 풀이나 RBD로 회피 가능.
- **체크포인트 write**: 대역폭은 나오지만 **tail latency** 비결정적.

### 4.3 CAP 문제 (실제 ML-25)

- Client가 파일을 열면 MDS에서 **capability (cap)** 획득 — 쓰기/읽기 권한 토큰.
- MDS cache 압박 시 client에게 cap 반납 요청.
- **D-state** 프로세스가 cap 쥐고 반납 못함 → MDS cache full → 다른 요청 block.
- 해결: D-state 유발 노드 재부팅 (ML-25 dgx04 사례).

### 4.4 회사 구성

- **SSD pool → RGW (S3)** — MLflow artifact, Harbor.
- **HDD pool → CephFS** — bulk dataset, 백업.
- 192.168.100.20 Ceph 클러스터 (양대곤 팀 구축).

---

## 5. 비교 매트릭스

| 축 | Lustre | Weka | GPFS | CephFS |
|----|--------|------|------|--------|
| 성능 (H100 클러스터) | ★★★★ | ★★★★★ | ★★★★ | ★★ |
| 작은 파일 | ★★ | ★★★★★ | ★★★★ | ★★ |
| 운영 난이도 | 높음 | 중간 | 높음 | 중간 |
| 비용 | OSS 주류 | 상용 | 상용 | OSS |
| NVIDIA 레퍼런스 | 많음 | 많음 | 적음 | 적음 |
| HA/DR | 복잡 | 내장 | 내장 | 내장 |
| POSIX 준수 | 높음 | 튜닝 의존 | 매우 높음 | 높음 |
| 체크포인트 N-way | 우수 | 매우 우수 | 우수 | 보통 |

---

## 6. 아키텍처 패턴

### 6.1 전용 스토리지 노드

Lustre/Weka/GPFS는 **GPU 노드와 분리된 스토리지 노드**에 구성:

```
[GPU 노드들 (DGX)] ────IB NDR──── [Storage 노드들 (NVMe/HDD)]
                                  │
                                  └─ Lustre OSS 또는 Weka
```

### 6.2 계층화 (Tiering)

```
  Hot   : NVMe   (학습 중 활성 데이터)
  Warm  : HDD    (과거 실험, 중간 데이터)
  Cold  : S3/Tape (장기 보관)
```

Weka/GPFS는 auto-tiering 내장. Lustre는 HSM (Hierarchical Storage Management) 연동.

### 6.3 체크포인트 IO 최적화

- 각 rank가 **별도 파일**로 쓰기 (contention 최소화).
- `torch.save` 대신 `torch.distributed.checkpoint` — 샤딩 체크포인트.
- **bounded parallelism** — 모든 rank가 동시에 쓰면 스토리지 knee point. 그룹 나눠 sequential.
- async checkpoint — compute과 I/O 오버랩.

---

## 7. K8s 통합

### 7.1 Lustre CSI

- Open-source CSI 드라이버 (Azure, DataDirect 제공).
- StaticProvisioning 일반적 — 미리 생성된 Lustre FS를 PV로.
- Mount: `fsType: lustre`, mountOptions로 stripe 지정.

### 7.2 Weka CSI

- Weka 공식 CSI. 동적 프로비저닝 지원.
- Pod에 Weka client 설치 필요 (DaemonSet).
- IB RDMA mount 지원.

### 7.3 CephFS CSI (회사 사용)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cephfs-hdd
provisioner: cephfs.csi.ceph.com
parameters:
  clusterID: <ceph-id>
  fsName: dn-cephfs
  pool: cephfs_data_hdd
  csi.storage.k8s.io/provisioner-secret-name: ceph-secret
```

---

## 8. 성능 벤치마크

### 8.1 도구

- `fio` — 범용 IO 벤치. bandwidth, IOPS, latency.
- `ior` — HPC 병렬 벤치. N-to-N, N-to-1 쓰기.
- `mdtest` — 메타데이터 스트레스 (file create/stat/delete).
- `dlio-benchmark` — AI 워크로드 재현 (imagenet, 체크포인트).

### 8.2 기준선 (참고)

- H100 노드 4대 + Weka : AllReduce 시 checkpoint write 200GB를 ~5초.
- Lustre 유사 규모 : ~7초.
- CephFS HDD : ~분 단위 (비교 불가, 프로덕션 대형 LLM엔 부적합).

---

## 9. 장애/시나리오

### 9.1 "Lustre MDS CPU 100%"

- 메타 연산 (stat, open) 폭주. `lctl get_param mdt.*.md_stats`.
- 원인: dataloader가 작은 파일 수만개 병렬 open. 대책: tar/wds 포맷, dataset cache.

### 9.2 "Weka rebuild 중 성능 저하"

- 노드 장애 후 erasure rebuild. 사용자 IO와 경합. **QoS 정책**으로 rebuild priority 조정.

### 9.3 "CephFS CAP release 실패" (실제 ML-25)

- Client가 cap 못 반납 → MDS cache full → 전체 hang.
- 해결: 해당 client 노드 재부팅, MDS 재시작은 최후.

### 9.4 "체크포인트 쓰는 중 학습 느려짐"

- I/O가 compute SM을 간접적으로 차지 (H2D copy).
- 대책: separate checkpoint rank, async offload.

---

## 10. 현대차 규모에서의 선택

**가정 스케일**: 수백~수천 H100, LLM 학습.

**일반적 선택**:
1. **Weka** 또는 **Lustre**가 메인 스토리지.
2. CephFS는 **보조/백업** 용도 (로그, 아카이브).
3. S3 (RGW 또는 오브젝트 스토리지) for artifact.

**본인(cwkim) 관점에서의 답변**:
> "저희 회사는 중규모 클러스터(4 DGX)에서 CephFS + RGW로 **운영 단순성** 우선 설계했습니다. 수백 노드 스케일에서는 CephFS의 MDS가 병목이므로 **Lustre나 Weka**로 이전하는 게 정석입니다. Lustre는 OSS 기반이라 OpEx 낮지만 운영 전문성 필요, Weka는 상용 지원 + userspace라 운영 부담 적다는 트레이드오프입니다. 현재 제가 운영에서 만난 CAP 반납 실패(ML-25) 같은 CephFS 한계가 스케일 커지면 더 빈번해질 것이라 **Weka를 유력 후보로 검토**할 것 같습니다."

---

## 11. 면접 Q&A

### Q1 (기초). Lustre의 file striping이 뭐고 왜 중요?
A. 파일을 N 조각으로 쪼개 N개 OST에 분산. Client가 OSS에 **병렬 RDMA** → 총 대역폭 = stripe 수 × per-OSS 대역폭. 대용량 파일의 aggregated throughput을 만든다.

### Q2. Weka가 "userspace"라는 말의 의미?
A. 데이터 경로(IO)를 커널 모듈 대신 **DPDK + SPDK userspace 스택**으로 처리. 장점: 커널 버전 의존성 낮음, 업그레이드 쉬움, CPU 캐시 효율. 단점: 기존 OS tool로 디버깅 어려움.

### Q3. CephFS MDS를 여러 active로 늘리면 되지 않나?
A. multi-active MDS는 **디렉토리 sub-tree partitioning**으로 동작. 잘 나뉘면 스케일 되지만, **hot subtree**가 있으면 partitioning 자주 변경 → 오버헤드. 파일 이동 locking 복잡. 회사 규모에서는 운영 리스크 대비 이득 불충분해 single active + SSD 풀 분리로 대응.

### Q4. LLM 체크포인트 200GB 빠르게 쓰는 전략?
A. (1) **torch.distributed.checkpoint** 샤딩 — rank별 별도 파일. (2) **async write** — GPU가 다음 step 돌리는 중 IO. (3) 병렬 파일시스템 (Weka/Lustre)의 N-way write. (4) compression (zstd).

### Q5. CephFS와 RBD 언제 뭘?
A. **CephFS** = POSIX 공유 파일시스템. 여러 Pod이 동시 read/write. **RBD** = 블록 디바이스. 단일 Pod 독점 (ReadWriteOnce). DB Persistent Volume, checkpoint scratch엔 RBD, 팀 공유 dataset엔 CephFS.

### Q6. Lustre에 OSS 노드 추가하면 즉시 성능 2배?
A. 신규 OSS의 OST는 **새 파일**에만 stripe 포함. 기존 파일은 `lfs migrate`로 수동 재분산 필요. 성능 2배는 **write-heavy 신규 워크로드** 기준. read-heavy 기존 데이터셋은 migrate 없이는 그대로.

### Q7. D-state 프로세스가 왜 CAP 반납 못하나?
A. D-state는 "uninterruptible sleep" — 커널이 IO 대기 중이라 시그널 무시. CephFS client의 cap release는 커널 코드 경로에서 수행되는데, 해당 프로세스가 이 경로에서 멈춰있으면 반납 불가. 결국 프로세스가 IO 깨어나거나 호스트 재부팅밖에.

### Q8. HPC에서 NFS를 왜 안 쓰나?
A. (1) NFSv3는 **single server**, 대역폭/IOPS 한계. (2) NFSv4도 **pNFS** 없이는 분산 안 됨. (3) POSIX locking 오버헤드. (4) 100Gbps급 bandwidth는 NFS 튜닝으로 어려움. 소규모 개발엔 적합, 학습 클러스터엔 부적합.

### Q9. Weka의 erasure coding이 replication보다 뭐가 좋은가?
A. 4+2 EC = 6 노드에 데이터 + 패리티 → 저장 효율 66% (replication 3x는 33%). 2노드 동시 실패까지 복구. 단점은 write 시 encode overhead — Weka는 userspace 최적화로 거의 없는 수준.

### Q10. 현대차 규모(수백~수천 GPU)로 가면 회사 CephFS를 그대로?
A. 안 그렇다. MDS 단일 active, HDD tail latency, 체크포인트 bandwidth 모두 knee point 진입. **Weka 또는 Lustre 전환**이 표준 경로. 마이그레이션 기간 동안 **CephFS를 백업/아카이브 계층**으로 격하, 핫 데이터는 신규 FS로. 운영 관점에선 Weka의 벤더 지원이 전환 부담 낮음.

---

## 12. 체크리스트

- [ ] Lustre MGS/MDS/OSS/OST 역할
- [ ] File striping 의미, `lfs setstripe`
- [ ] Weka userspace + EC 구조
- [ ] CephFS CAP 메커니즘, D-state 실패 원인
- [ ] 4개 FS 비교 매트릭스 (성능/비용/운영)
- [ ] Multi-active MDS 트레이드오프
- [ ] 체크포인트 전략 4가지
- [ ] CephFS vs RBD 용도
- [ ] 회사 규모→현대차 규모 전환 시나리오 답변

---

## 12.5 2025-2026 최신 키워드 (면접 심화)

- **DAOS (Distributed Asynchronous Object Storage)**: Intel/HPE 주도 NVMe-class PMem 타깃. Aurora/Leonardo HPC 채택.
- **NVIDIA GPUDirect Storage (GDS / cuFile)**: NVMe/IB에서 GPU 메모리로 **CPU bounce 없이** 직접 로드. PyTorch Data Loader + DALI 경로.
- **Lustre 2.15 LTS**: PFL(Progressive File Layout), DoM(Data-on-MDT) — 작은 파일 MDS에 상주시켜 CephFS 고민 일부 해소.
- **VAST Data / DDN EXAScaler AI400X2**: H100 레퍼런스 GPU 스토리지. Weka 경쟁사.
- **S3 over RDMA (WebObject)**: MinIO/Ceph RGW에서 RDMA transport 실험. Artifact I/O 가속.

## 13. 연계 문서

- HW 기반: [./gpu-gpudirect-deep-dive.md](./gpu-gpudirect-deep-dive.md), [./ceph-storage-deep-dive.md](./ceph-storage-deep-dive.md)
- 커널 계층: [../kernel/rdma-ib-deep-dive.md](../kernel/rdma-ib-deep-dive.md)
- K8s CSI/CNI: [../k8s/cni-multus-deep-dive.md](../k8s/cni-multus-deep-dive.md)
- 학습 I/O 서사: [../integration/llm-training-workload-deep-dive.md](../integration/llm-training-workload-deep-dive.md), [../k8s/mlops-stack-deep-dive.md](../k8s/mlops-stack-deep-dive.md)
- 장애 서사(ML-25 CAP): [../../interview/ml-platform-ownership-guide.md](../../interview/ml-platform-ownership-guide.md)
- 허브: [../integration/dgx-ib-multinode-training-guide.md](../integration/dgx-ib-multinode-training-guide.md)

## 14. 참고

- Lustre manual: https://doc.lustre.org/
- Weka docs: https://docs.weka.io/
- Ceph: https://docs.ceph.com/
- DLIO-benchmark: https://github.com/argonne-lcf/dlio_benchmark
- 관련: [ceph-storage-deep-dive.md](ceph-storage-deep-dive.md), [../integration/llm-training-workload-deep-dive.md](../integration/llm-training-workload-deep-dive.md)
