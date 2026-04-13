# Ceph & 스토리지 완전 정복 - 분산 스토리지의 모든 것

> **목적**: 리눅스 파일시스템 기초부터 Ceph 분산 스토리지의 내부 구조까지 단계별로 쌓아 올린다. 우리 회사가 Ceph + IB로 ML 데이터를 어떻게 다루는지 끝까지 이해.

---

## 0. 큰 그림

```
[애플리케이션]
   ↓ open(), read(), write()
[리눅스 VFS] (Virtual File System)
   ↓
[구체적 파일시스템] (ext4, xfs, cephfs, ...)
   ↓
[블록 디바이스] (/dev/sda, RBD, ...)
   ↓
[물리/네트워크 스토리지]
```

이 스택을 위에서 아래로, 그리고 분산 스토리지(Ceph)로 확장.

---

## 1. 리눅스 파일시스템 기초

### 1.1 VFS (Virtual File System)

리눅스 커널은 다양한 파일시스템(ext4, xfs, nfs, cephfs 등)을 **공통 인터페이스**로 추상화합니다. 이게 VFS.

```
[App]
  ↓ open("/data/file.txt")
[VFS] ── 추상 인터페이스 (inode, dentry, file)
  ↓
[ext4 driver]   [cephfs driver]   [nfs driver]
  ↓                ↓                  ↓
[블록 디바이스]   [네트워크]         [네트워크]
```

핵심 자료구조:
- **inode**: 파일 메타데이터 (소유자, 권한, 크기, 블록 위치)
- **dentry**: 디렉터리 엔트리 (이름 ↔ inode 매핑)
- **file**: 열린 파일 (offset, 모드)
- **superblock**: 파일시스템 전체 메타데이터

### 1.2 inode

**파일 = inode + 데이터 블록**.

```bash
ls -i myfile.txt
# 12345678 myfile.txt   ← inode 번호

stat myfile.txt
# Size: 1024
# Blocks: 8
# Inode: 12345678
# Links: 1
# Access/Modify/Change times
```

inode가 데이터 블록 위치(직접/간접 포인터)를 가집니다.

### 1.3 Page Cache

리눅스는 **모든 파일 IO를 메모리에 캐싱**합니다. 이게 page cache.

```
read()
  ↓
캐시 hit? → 메모리에서 바로 반환
캐시 miss → 디스크에서 읽어 캐시에 저장 → 반환

write()
  ↓
캐시에만 쓰고 즉시 반환 (dirty page)
나중에 백그라운드에서 디스크로 flush
```

이래서 같은 파일 두 번째 read는 폭발적으로 빠릅니다. 그리고 write는 비동기라 빠르지만 **전원 끊기면 데이터 유실** 가능.

### 1.4 fsync

데이터를 디스크에 강제로 flush하는 시스템 콜.

```c
write(fd, data, len);  // page cache에만 들어감 (빠름, 위험)
fsync(fd);             // 디스크에 진짜 쓰임 (느림, 안전)
```

DB나 트랜잭션 시스템은 매 commit마다 fsync를 호출합니다. 그래서 디스크의 fsync 성능이 곧 DB 성능.

### 1.5 mount

파일시스템을 디렉터리에 붙이는 작업.

```bash
mount /dev/sda1 /mnt/data
# 이제 /mnt/data 아래 접근 = /dev/sda1의 ext4를 통해
```

마운트 포인트 = 트리에 다른 파일시스템을 갖다 붙이는 지점. 컨테이너의 mount namespace는 이걸 컨테이너별로 격리합니다.

---

## 2. 스토리지의 3가지 종류

| 종류 | 단위 | 예시 | 용도 |
|------|------|------|------|
| **블록 스토리지** | 고정 크기 블록 | iSCSI, RBD, EBS | OS 디스크, DB |
| **파일 스토리지** | 파일/디렉터리 | NFS, CephFS, SMB | 공유 파일, 홈 디렉터리 |
| **오브젝트 스토리지** | 객체 (메타+데이터) | S3, RGW, GCS | 백업, 미디어, ML 데이터 |

### 2.1 블록 스토리지

- **로우 디스크**처럼 보임. 위에 파일시스템을 만들어 사용
- 예: `mkfs.ext4 /dev/rbd0` 후 마운트
- 빠르고 유연하지만 한 클라이언트가 독점

### 2.2 파일 스토리지

- **파일시스템 자체**를 네트워크로 공유
- POSIX 호환 (open/read/write)
- 여러 클라이언트가 동시 접근 가능

### 2.3 오브젝트 스토리지

- **HTTP API**로 객체 단위 접근 (PUT/GET/DELETE)
- 디렉터리 개념 없음 (key는 그냥 문자열)
- 무한 확장, 메타데이터 풍부, 그러나 random write 불가
- ML 데이터셋 저장에 이상적

---

## 3. Ceph란

> **하나의 클러스터에서 블록/파일/오브젝트 스토리지를 모두 제공하는 분산 스토리지**

### 3.1 핵심 컨셉

- **소프트웨어 기반**: 일반 x86 서버에 디스크 꽂아서 클러스터 구성
- **데이터 자동 분산 + 복제** (replication 또는 erasure coding)
- **단일 장애점 없음** (master 노드가 따로 없음)
- **자가 치유** (OSD 죽으면 다른 곳에 복제본 자동 생성)

### 3.2 우리 회사의 위치

- DGX 클러스터 옆에 Ceph 클러스터 운영
- ML 학습 데이터, 체크포인트, 모델 파일 저장
- IB 전용 포트(`ibp24s0`, `ibp220s0`)로 Ceph 트래픽도 RDMA

---

## 4. Ceph 구성 요소

### 4.1 RADOS (Reliable Autonomic Distributed Object Store)

Ceph의 **핵심 엔진**. 모든 데이터(파일이든 블록이든)는 결국 RADOS 위의 객체로 저장됩니다.

```
[블록 인터페이스 (RBD)]   [파일 인터페이스 (CephFS)]   [객체 인터페이스 (RGW)]
                       ↓
                    [RADOS]   ← 모든 데이터의 진짜 저장소
                       ↓
                  [수많은 OSD]
```

### 4.2 OSD (Object Storage Daemon)

- 실제 디스크에 데이터를 쓰고 읽는 데몬
- 디스크 1개당 OSD 1개가 일반적
- 50대 서버 × 디스크 12개 = OSD 600개 같은 스케일도 가능

### 4.3 MON (Monitor)

- 클러스터의 **상태 정보(map)** 관리
- Paxos 기반 합의로 일관성 유지
- 보통 3개 또는 5개 (홀수, quorum 위해)

관리하는 map들:
- **OSD map**: OSD 목록, 상태(up/down)
- **MON map**: 모니터 자체 정보
- **PG map**: placement group 위치
- **CRUSH map**: 데이터 배치 알고리즘 정보

### 4.4 MGR (Manager)

- 모니터링, 대시보드, 외부 시스템 연동
- Prometheus exporter도 여기서 제공
- 보통 2개 (active/standby)

### 4.5 MDS (Metadata Server)

- **CephFS 사용 시에만** 필요
- 디렉터리 구조, 파일 메타데이터 관리
- 데이터 자체는 OSD에, 메타데이터만 MDS가

### 4.6 RGW (RADOS Gateway)

- **S3 호환 API** 제공 (오브젝트 스토리지 인터페이스)
- 내부적으로는 RADOS에 객체 저장

---

## 5. CRUSH 알고리즘 (Ceph의 핵심)

### 5.1 문제

수백 대의 OSD가 있을 때, 데이터를 **어디에** 저장할지 어떻게 결정하나?

전통적 방법: 메타 서버가 위치 테이블 관리 → 병목, 단일 장애점.

**CRUSH의 답**: **계산으로** 위치를 결정. 메타 서버 불필요.

### 5.2 CRUSH (Controlled Replication Under Scalable Hashing)

```
객체 ID → 해시 → PG (Placement Group) → CRUSH 규칙 → OSD 리스트
```

- 어느 클라이언트든 같은 객체 ID를 넣으면 같은 OSD가 나옴
- 메타 서버에 물어볼 필요 없음
- OSD 추가/제거 시 **데이터 일부만** 재배치 (전체 재해시 X)

### 5.3 CRUSH map과 failure domain

CRUSH는 토폴로지를 인식합니다.

```
root: default
  ├── rack1
  │   ├── host1
  │   │   ├── osd.1
  │   │   └── osd.2
  │   └── host2
  ├── rack2
  └── rack3
```

규칙: "복제본 3개를 **다른 rack**에 둬라" → CRUSH가 자동으로 rack-aware 배치.
- 1개 rack 통째로 나가도 데이터 보존
- 우리 회사도 IDC 내 rack 단위 failure domain 설정 가능

### 5.4 PG (Placement Group)

객체와 OSD 사이의 **간접 레이어**.

```
객체 100만개 → PG 1024개 → OSD 100개
```

왜 필요한가:
- OSD 직접 매핑하면 OSD 추가/제거 시 모든 객체 재계산
- PG 단위로 매핑하면 PG → OSD 매핑만 바뀜 → 효율적

PG 개수는 운영자가 정함 (보통 OSD당 100개 정도 권장).

---

## 6. 데이터 복제 (Replication vs Erasure Coding)

### 6.1 Replication

```
객체 → 3개 OSD에 동일 복사본 저장
    저장 효율: 33% (3배 공간 사용)
    안정성: OSD 2개 동시 손실까지 견딤
    성능: 빠름 (단순 복사)
```

### 6.2 Erasure Coding (EC)

RAID 6 같은 패리티 기반.

```
객체 → 4 데이터 + 2 패리티 = 6 OSD에 분산
    저장 효율: 67% (1.5배 공간)
    안정성: OSD 2개 손실까지
    성능: 느림 (인코딩/디코딩 오버헤드)
```

ML 학습 데이터처럼 큰 콜드 데이터에 적합.

---

## 7. RBD (RADOS Block Device)

### 7.1 동작

- Ceph 위에 **가상 블록 디바이스** 제공
- 클라이언트 입장: `/dev/rbd0` 같은 디스크처럼 보임
- 위에 ext4/xfs 만들어 사용

```bash
# RBD 이미지 생성
rbd create mypool/myimage --size 100G

# 클라이언트에서 매핑
rbd map mypool/myimage
# /dev/rbd0

# 마운트
mkfs.ext4 /dev/rbd0
mount /dev/rbd0 /mnt/data
```

### 7.2 동작 원리

- 100GB 이미지 = 4MB 객체 25,000개로 쪼개져 RADOS에 저장
- RBD 클라이언트가 read/write를 객체 IO로 변환

### 7.3 K8s 연동

CSI 드라이버(`ceph-csi`)를 통해:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: <cluster-id>
  pool: rbd
```

PVC 만들면 자동으로 RBD 이미지 생성 후 Pod에 마운트.

**용도**: DB, 단일 Pod이 독점 사용하는 데이터.

---

## 8. CephFS (분산 파일시스템)

### 8.1 동작

- POSIX 호환 파일시스템
- 여러 클라이언트가 동시 마운트 가능
- 메타데이터: MDS, 데이터: OSD

```bash
# 직접 마운트 (커널 모듈)
mount -t ceph mon1:6789:/ /mnt/cephfs -o name=admin,secret=...
```

### 8.2 K8s 연동

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
provisioner: cephfs.csi.ceph.com
```

ReadWriteMany (RWX) 가능 → 여러 Pod이 같은 PVC 공유.

**용도**: 학습 데이터셋 공유, 여러 학습 잡이 같은 데이터 읽기.

---

## 9. RGW (S3 호환 오브젝트 스토리지)

### 9.1 동작

- HTTP/S 엔드포인트 제공
- AWS S3 API 100% 호환 (대부분)
- 클라이언트는 boto3, awscli 그대로 사용

```python
import boto3
s3 = boto3.client('s3', endpoint_url='https://ceph-rgw.company.com')
s3.upload_file('train.parquet', 'mybucket', 'data/train.parquet')
```

### 9.2 용도

- ML 데이터셋 (모델 학습 시 stream 읽기)
- MLflow 아티팩트 스토어
- 모델 레지스트리 백엔드
- 백업

---

## 10. Ceph + RDMA (★ 우리 회사 핵심)

### 10.1 왜 RDMA?

대규모 ML 학습 시:
- 학습 데이터 prefetch가 IO 병목
- 체크포인트 저장 시 GPU memory → 디스크가 분당 수 TB
- 일반 TCP/IP로는 네트워크 스택이 병목

**Ceph는 메시징 백엔드로 RDMA 지원** (`ms_type = async+rdma`).

### 10.2 우리 회사 구성

```
DGX 노드          Ceph 노드
  │                  │
  ├ ibp24s0  ←─IB→  ibp24s0  (Ceph 전용 채널)
  ├ ibp220s0 ←─IB→  ibp220s0
  ├ ibp64s0  ...    (다른 namespace 학습용)
```

- DGX의 8개 HCA 중 2개를 Ceph 전용으로 할당
- 학습 트래픽과 스토리지 트래픽이 같은 IB fabric을 쓰지만 포트 분리
- 학습 데이터 IO도 RDMA로 → kernel 우회 → 고대역폭

---

## 11. K8s에서 Ceph 사용

### 11.1 CSI (Container Storage Interface)

K8s가 외부 스토리지를 다루는 표준 인터페이스.

```
[Pod] PVC 요청
   ↓
[K8s] StorageClass 보고 CSI 드라이버 호출
   ↓
[ceph-csi] RBD 이미지 생성 / CephFS subvolume 생성
   ↓
[Pod에 마운트]
```

### 11.2 PV / PVC

```yaml
# PVC: "100GB ReadWriteOnce 스토리지 줘"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-data
spec:
  accessModes: [ReadWriteOnce]
  resources: { requests: { storage: 100Gi } }
  storageClassName: ceph-rbd
```

자동으로:
1. ceph-csi가 RBD 이미지 100G 생성
2. PV 자동 생성
3. PVC와 PV 바인딩
4. Pod에 마운트

### 11.3 Access Modes

| 모드 | 약자 | 의미 | 대표 |
|------|------|------|------|
| ReadWriteOnce | RWO | 한 노드만 R/W | RBD |
| ReadOnlyMany | ROX | 여러 노드 읽기만 | - |
| ReadWriteMany | RWX | 여러 노드 R/W | CephFS, NFS |
| ReadWriteOncePod | RWOP | 한 Pod만 R/W | 새 옵션 |

---

## 12. 운영 시나리오

### 12.1 OSD가 죽었다

```bash
ceph status
# HEALTH_WARN 1 osds down
# Degraded data redundancy: 1234/9999 objects degraded

ceph osd tree
# osd.5  down
```

자동 동작:
- MON이 OSD down 감지
- CRUSH가 해당 OSD가 책임지던 PG를 다른 OSD로 재할당
- 자동으로 복제본 재생성 (rebalance)

운영자 액션:
- 디스크 교체 → 새 OSD 추가 → 자동으로 데이터 재배치

### 12.2 클러스터가 SLOW_OPS

```bash
ceph health detail
# 100 slow ops, oldest one blocked for 30 sec
```

원인 후보:
- 특정 OSD의 디스크 느림
- 네트워크 이슈 (IB 링크 다운?)
- PG 개수 과다/부족

### 12.3 K8s Pod에서 마운트 안 됨

- ceph-csi 드라이버 로그 확인
- Ceph 클러스터 자체는 정상인지
- Pod이 있는 노드에서 Ceph 클러스터로 네트워크 도달 가능한지

---

## 13. 면접 예상 질문

### Q1. "Ceph가 단일 장애점이 없다는 게 어떤 의미인가요?"
> "전통적인 분산 스토리지는 메타데이터 서버가 데이터 위치를 관리해서 그게 죽으면 클러스터 전체가 멈춥니다. Ceph는 CRUSH 알고리즘으로 클라이언트가 직접 데이터 위치를 계산할 수 있어 메타 서버 자체가 필요 없습니다. MON은 클러스터 상태 맵만 관리하고 그것도 Paxos 기반으로 다중화되어 있어 단일 장애점이 없습니다."

### Q2. "Replication과 Erasure Coding 중 뭘 쓰나요?"
> "Hot data, latency 민감한 워크로드는 3-replica로 빠르게. 콜드 데이터, 백업, 큰 ML 데이터셋은 EC로 공간 효율 확보. 우리 환경에서는 학습 데이터셋은 EC로, 자주 쓰는 캐시/체크포인트는 replication으로 분리할 수 있습니다."

### Q3. "RBD vs CephFS vs RGW 언제 뭘 쓰나요?"
> "단일 Pod이 독점하는 영구 스토리지(DB 등)는 RBD, 여러 Pod이 공유하는 학습 데이터는 CephFS, ML 데이터셋이나 모델 아티팩트처럼 이뮤터블/대용량은 RGW(S3 API)를 씁니다."

### Q4. "Ceph 트래픽을 IB로 보내는 이유는?"
> "ML 학습은 prefetch와 체크포인트로 분당 수 TB 단위의 IO를 발생시킵니다. 일반 이더넷에서는 TCP/IP 스택 오버헤드와 대역폭이 병목이 됩니다. Ceph는 ms_type=async+rdma 설정으로 OSD 간/클라이언트 통신을 RDMA로 전환할 수 있어, 우리는 DGX의 IB 포트 일부를 Ceph 전용으로 할당해 학습 IO도 RDMA화했습니다."

### Q5. "CSI 드라이버는 정확히 뭘 하나요?"
> "K8s가 외부 스토리지 시스템과 통신하는 표준 인터페이스입니다. PVC가 생성되면 ceph-csi가 ProvisionVolume을 호출해 Ceph에 RBD 이미지를 만들고, Pod이 스케줄되면 NodeStageVolume/NodePublishVolume으로 노드에 RBD를 매핑하고 Pod 안에 마운트합니다."

---

## 14. 한 줄 요약

> **"Ceph는 RADOS라는 분산 객체 저장소 위에 RBD(블록), CephFS(파일), RGW(S3) 인터페이스를 얹은 통합 스토리지다. CRUSH 알고리즘으로 메타 서버 없이 데이터 위치를 계산하고, OSD 단위로 자동 분산/복제/치유된다. K8s는 ceph-csi로 PVC ↔ Ceph 리소스를 연결하고, 우리 환경은 DGX의 IB 포트 일부를 Ceph 전용으로 할당해 학습 IO를 RDMA화했다."**
