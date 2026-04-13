# Container Runtime & Image 완전 정복

> **목적**: "컨테이너가 시작된다"는 말 뒤에 실제로 무슨 일이 일어나는지 OCI 표준, containerd, runc, OverlayFS까지 끝까지 본다. Docker는 사실 이 위에 얹힌 얇은 껍데기.

---

## 0. 큰 그림

```
[사용자: docker run / kubectl apply]
   ↓
[고수준 런타임: Docker / containerd / CRI-O]
   ↓ OCI Runtime Spec (JSON)
[저수준 런타임: runc / crun / kata]
   ↓ 시스템 콜
[리눅스 커널: namespace + cgroup + capabilities]
   ↓
[컨테이너 프로세스]
```

K8s 1.20+ 부터는 **containerd**가 표준이고 Docker는 더 이상 K8s의 직접 런타임이 아닙니다.

---

## 1. OCI 표준

### 1.1 왜 표준이 필요했나

초기에는 도커가 독점. "도커 이미지", "도커 런타임"이 사실상 표준.

→ Linux Foundation 산하 **Open Container Initiative (OCI)** 가 표준 정의:
- **Image Spec**: 이미지 포맷 (어떻게 패키징할지)
- **Runtime Spec**: 런타임 동작 (어떻게 실행할지)
- **Distribution Spec**: 레지스트리 API

### 1.2 결과

이제 도커, containerd, podman 등이 **같은 이미지를 같은 방식으로 실행**.

---

## 2. 컨테이너 이미지의 정체

### 2.1 그냥 tar 파일 모음

이미지는 마법이 아니라 **파일시스템 스냅샷의 레이어들 + 메타데이터 JSON**.

```bash
# 이미지를 tar로 추출
docker save nginx:latest -o nginx.tar
tar -tf nginx.tar | head
# manifest.json
# 5dbe8...layer.tar
# a8b9...layer.tar
# config.json
```

내용:
- `manifest.json`: 어떤 레이어들이 있는지 목록
- `config.json`: ENTRYPOINT, ENV, 환경 변수 등
- 각 `layer.tar`: 그 레이어의 파일들

### 2.2 레이어란

Dockerfile의 각 명령이 **하나의 레이어**를 만듭니다.

```dockerfile
FROM ubuntu:22.04          # 레이어 1: 우분투 베이스
RUN apt-get update         # 레이어 2: apt 캐시
RUN apt-get install nginx  # 레이어 3: nginx 설치
COPY config /etc/nginx/    # 레이어 4: 설정 파일
```

각 레이어는 **이전 레이어와의 차이(diff)** 만 저장.

### 2.3 왜 이렇게?

- **재사용**: 여러 이미지가 같은 베이스 레이어 공유 → 디스크 절약
- **캐싱**: 변경되지 않은 레이어는 다시 안 받음 → 빌드 빠름
- **배포 효율**: 변경된 레이어만 push/pull

### 2.4 이미지 = "조립식 가구 설명서"

- 레이어들 = 부품
- manifest = 조립 순서
- config = 사용 설명서 (어떻게 실행할지)

---

## 3. OverlayFS - 레이어를 합치는 마법

### 3.1 문제

10개 레이어가 있는 이미지를 어떻게 하나의 파일시스템처럼 보이게?

→ **OverlayFS** (또는 overlay2): 리눅스 커널의 union filesystem.

### 3.2 동작 원리

```
[upper] (쓰기 가능, 컨테이너 변경사항)
   ↓
[lower3] ── 레이어 3
   ↓
[lower2] ── 레이어 2
   ↓
[lower1] ── 레이어 1 (베이스, 읽기 전용)
   ↓
[merged] ← 이게 컨테이너가 보는 최종 파일시스템
```

- 읽기: 위에서부터 검색, 처음 발견된 파일 반환
- 쓰기: 항상 upper에 작성 (lower는 절대 안 건드림)
- 삭제: upper에 "whiteout" 마커 생성

### 3.3 직접 해보기

```bash
mkdir lower1 lower2 upper work merged
echo "from lower1" > lower1/a.txt
echo "from lower2" > lower2/a.txt
echo "from lower2 only" > lower2/b.txt

sudo mount -t overlay overlay \
  -o lowerdir=lower2:lower1,upperdir=upper,workdir=work \
  merged

ls merged/
# a.txt b.txt
cat merged/a.txt
# from lower2  (위 레이어 우선)

# 컨테이너에서 쓰기 시뮬레이션
echo "modified" > merged/a.txt
ls upper/
# a.txt   ← upper에만 쓰임 (Copy-on-Write)
```

이게 컨테이너 이미지가 동작하는 방식의 핵심.

### 3.4 컨테이너 시작 시

```
[베이스 이미지 레이어들] = lower (read-only)
[새로 만든 빈 디렉터리] = upper
   ↓
[OverlayFS mount]
   ↓
[/var/lib/containerd/.../merged] ← 컨테이너의 / 가 됨
```

컨테이너가 만든 모든 변경사항은 upper에만. 컨테이너 삭제하면 upper만 삭제, 원본 이미지는 그대로.

---

## 4. containerd - K8s의 표준 런타임

### 4.1 위치

```
[K8s kubelet]
   ↓ CRI (Container Runtime Interface, gRPC)
[containerd]
   ↓ OCI Runtime Spec
[runc]
   ↓
[리눅스 커널]
```

### 4.2 책임

- 이미지 pull / push / 관리
- 컨테이너 생성 / 시작 / 정지 / 삭제
- snapshot 관리 (OverlayFS 마운트 등)
- runc를 호출해 실제 컨테이너 실행

### 4.3 왜 Docker 대신 containerd?

- Docker = containerd + Docker CLI + 빌드 + 그 외 잡다
- K8s는 컨테이너 실행만 필요 → containerd가 더 슬림하고 적합
- K8s 1.24부터 dockershim 제거

### 4.4 도구

```bash
# containerd 직접 다루기
ctr image list
ctr container list
ctr task list

# K8s 관리 컨테이너 보려면
crictl ps        # docker ps와 비슷
crictl images
crictl logs <container-id>
```

---

## 5. runc - 진짜 컨테이너를 만드는 놈

### 5.1 책임

- OCI Runtime Spec(JSON) 받아서
- namespace 생성, cgroup 설정, capabilities 적용
- 컨테이너 프로세스 실행

### 5.2 OCI Runtime Spec 예시

```json
{
  "process": {
    "args": ["/bin/bash"],
    "env": ["PATH=/usr/bin"]
  },
  "root": { "path": "rootfs" },
  "linux": {
    "namespaces": [
      {"type": "pid"}, {"type": "net"},
      {"type": "ipc"}, {"type": "mnt"}, {"type": "uts"}
    ],
    "resources": {
      "memory": {"limit": 1073741824},
      "cpu": {"quota": 100000, "period": 100000}
    },
    "devices": [...]
  }
}
```

이 JSON 하나가 컨테이너의 모든 것을 정의.

### 5.3 runc 실행

```bash
mkdir mycontainer && cd mycontainer
mkdir rootfs
# 베이스 파일시스템 추출 (예: alpine)
docker export $(docker create alpine) | tar -C rootfs -xf -

# OCI spec 생성
runc spec  # config.json 자동 생성

# 컨테이너 실행
sudo runc run mycontainer
# / # _   ← 컨테이너 안!
```

이 과정이 docker run / kubectl run 안에서 일어나는 일.

---

## 6. Pod 생성 전체 흐름 (★ 면접용 정리)

```
[1] kubectl apply -f pod.yaml
       ↓
[2] API Server가 etcd에 Pod object 저장
       ↓
[3] Scheduler가 노드 선택
       ↓
[4] kubelet이 자기 노드의 Pod 변경 감지
       ↓
[5] kubelet이 containerd에 CRI gRPC 요청
       - RunPodSandbox: pause 컨테이너 생성 (namespace 보유)
       - PullImage: 이미지 다운로드
       - CreateContainer: 컨테이너 생성
       ↓
[6] containerd가:
       - 이미지 레이어 OverlayFS 마운트 → rootfs 준비
       - OCI Runtime Spec(config.json) 생성
       - runc 호출
       ↓
[7] runc가:
       - clone() 시스템 콜로 새 namespace 생성
       - cgroupfs에 자원 제한 설정
       - chroot/pivot_root로 rootfs 진입
       - capabilities 드롭
       - exec()로 실제 프로세스 시작
       ↓
[8] CNI 호출 (kubelet → containerd → CNI 플러그인)
       - veth pair 생성, IP 할당, 라우팅
       ↓
[9] 컨테이너 실행 중
```

---

## 7. pause 컨테이너 (Pod의 비밀)

### 7.1 무엇

K8s Pod 시작하면 항상 보이는 신비한 컨테이너.

```bash
crictl ps -a
# CONTAINER ... IMAGE             NAME
# abc123  ... my-app:1.0         mycontainer
# def456  ... pause:3.9          POD
```

### 7.2 역할

- Pod의 namespace를 **소유**하는 컨테이너
- Pod 안의 모든 컨테이너가 이 pause의 namespace를 공유
- pause가 죽으면 Pod 자체가 죽음

### 7.3 왜 필요?

- Pod = 여러 컨테이너 묶음 (network, IPC namespace 공유)
- 누군가는 그 namespace의 "owner"가 되어야 함
- 메인 앱 컨테이너가 죽었다 살아도 namespace는 유지되어야 함
- → pause가 영원히 살아 있으면서 namespace 유지

`/pause` 바이너리는 sleep만 무한히 함. 매우 작음(700KB).

---

## 8. 이미지 빌드

### 8.1 Dockerfile 예시 분석

```dockerfile
FROM python:3.11-slim                    # 베이스 이미지
WORKDIR /app                             # 작업 디렉터리
COPY requirements.txt .                  # 파일 복사
RUN pip install -r requirements.txt      # 패키지 설치 (레이어!)
COPY . .                                 # 소스 복사 (마지막)
CMD ["python", "app.py"]                 # 실행 명령
```

### 8.2 캐싱 전략

```dockerfile
# 나쁜 예: 소스 변경할 때마다 pip install 재실행
COPY . .
RUN pip install -r requirements.txt

# 좋은 예: requirements 변경 시에만 pip install 재실행
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
```

레이어 캐시는 **위에서 아래로** 무효화. 자주 안 바뀌는 것을 위로.

### 8.3 multi-stage 빌드

```dockerfile
# 빌드 스테이지
FROM golang:1.21 AS builder
WORKDIR /src
COPY . .
RUN go build -o app

# 런타임 스테이지 (가벼움)
FROM alpine:3.18
COPY --from=builder /src/app /app
CMD ["/app"]
```

빌드 도구 없이 결과물만 가져와서 이미지 크기 감소.

### 8.4 Docker 없이 빌드: BuildKit, kaniko, buildah

```bash
# K8s 안에서 이미지 빌드 (Docker daemon 없이)
kaniko --dockerfile=Dockerfile --destination=registry/myimage:v1
```

CI/CD 환경에서 권장.

---

## 9. 이미지 레지스트리

### 9.1 구조

```
registry.company.com/
├── library/
│   ├── nginx:latest
│   └── ubuntu:22.04
└── teams/
    ├── ml/training:v3
    └── platform/api:v1
```

### 9.2 동작

```bash
docker pull nginx:latest
# 1. registry-1.docker.io에 manifest 요청
# 2. manifest에서 레이어 목록 받음
# 3. 각 레이어 blob 다운로드
# 4. 로컬에서 OverlayFS 마운트 가능한 형태로 저장
```

### 9.3 종류

- Docker Hub (퍼블릭)
- AWS ECR, GCP GCR, Azure ACR (클라우드)
- **Harbor** (셀프 호스팅, 많이 씀)
- Quay, Nexus

### 9.4 인증

```yaml
# K8s에서 private registry 사용
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64>
---
spec:
  imagePullSecrets:
  - name: regcred
```

---

## 10. rootless / user namespace

### 10.1 문제

전통적으로 컨테이너 안 root = 호스트 root (UID 0). 보안 위험.

### 10.2 해결: user namespace

```
호스트 UID 100000 ↔ 컨테이너 안 UID 0

→ 컨테이너 안에서는 root처럼 보이지만
   호스트에서는 평범한 일반 유저
```

권한 탈출해도 호스트에서는 일반 유저 권한만.

### 10.3 podman, rootless docker

이런 것들이 user namespace 기반으로 권한 없이 컨테이너 실행 가능.

K8s에서도 점진적으로 도입 중 (`spec.hostUsers: false`).

---

## 11. 우리 회사 환경 적용

### 11.1 컨테이너 런타임

DGX 노드들에서 containerd가 실행 중. NVIDIA Container Toolkit이 설치되어 GPU 컨테이너가 동작.

```bash
# 노드에서 확인
crictl info
# runtimeName: containerd

ls /etc/containerd/config.toml
# 안에 nvidia runtime이 default로 설정되어 있을 것
```

### 11.2 GPU 컨테이너 실행 시

```
[1] kubelet → containerd: CreateContainer
[2] containerd가 OCI spec 생성하면서 nvidia hook 추가
[3] runc가 컨테이너 시작
[4] nvidia-container-runtime hook이 실행되어:
    - /dev/nvidia* 디바이스 파일 마운트
    - libnvidia-*, libcuda 라이브러리 마운트
[5] 컨테이너 안에서 GPU 사용 가능
```

### 11.3 ML 이미지 특징

- 대용량 (CUDA + cuDNN + PyTorch = 5~15GB)
- 베이스: `nvcr.io/nvidia/pytorch:xx.xx-py3` (NVIDIA 공식)
- 캐싱 잘 활용해야 빌드 시간 단축

---

## 12. 면접 예상 질문

### Q1. "Docker와 containerd의 관계는?"
> "containerd는 Docker가 분리한 핵심 컨테이너 런타임입니다. Docker = containerd + Docker CLI + 이미지 빌드 + 네트워킹 등 부가 기능. K8s는 컨테이너 실행만 필요해서 1.24부터 Docker 대신 containerd를 직접 사용하고 있고, 우리 환경도 containerd 기반입니다."

### Q2. "컨테이너 이미지가 가벼운 이유는?"
> "이미지가 레이어로 구성되어 있어 베이스 이미지를 여러 컨테이너가 공유합니다. 또 OverlayFS로 마운트되기 때문에 컨테이너마다 파일시스템을 복사하지 않고, 변경사항만 별도 upper 디렉터리에 Copy-on-Write로 저장합니다. 1GB 이미지로 100개 컨테이너를 띄워도 디스크는 1GB 약간 + 변경분만 사용합니다."

### Q3. "Pod 안의 pause 컨테이너는 왜 있나요?"
> "Pod의 namespace(network, IPC 등)를 소유하는 컨테이너입니다. Pod 안의 다른 컨테이너들이 이 pause의 namespace를 공유하기 때문에 메인 컨테이너가 재시작되어도 namespace가 유지됩니다. 그래서 IP가 변하지 않고 sidecar 컨테이너끼리 localhost로 통신할 수 있습니다."

### Q4. "Pod 생성 흐름을 끝까지 설명해보세요."
> (위 6장의 9단계 그대로 답하면 됨)

### Q5. "이미지 빌드 캐싱은 어떻게 효율화하나요?"
> "Dockerfile은 위에서 아래로 레이어가 캐시되고, 변경되면 그 아래 모두 무효화됩니다. 자주 바뀌지 않는 것(시스템 패키지 설치, 의존성 install)을 위에 두고, 자주 바뀌는 소스 코드 COPY를 마지막에 둬야 합니다. multi-stage build로 빌드 도구를 최종 이미지에서 제거하고, .dockerignore로 불필요한 파일 제외도 중요합니다."

---

## 13. 한 줄 요약

> **"컨테이너 = OCI 표준 이미지를 OverlayFS로 마운트해 namespace + cgroup으로 격리한 프로세스. K8s는 kubelet → containerd(CRI) → runc(OCI Runtime) 흐름으로 컨테이너를 만들고, GPU 같은 특수 자원은 nvidia-container-runtime hook이 끼어들어 디바이스/라이브러리를 추가 마운트한다."**
