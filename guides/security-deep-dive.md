# 컨테이너/K8s 보안 완전 정복

> **목적**: 컨테이너와 K8s의 보안 메커니즘을 커널 레벨부터 정책 레벨까지 본다. capabilities, seccomp, SELinux/AppArmor, Pod Security Standards, RBAC, Network Policy, Secret 관리.

---

## 0. 보안의 계층

```
[Application Layer]   - Pod Security Standards, RBAC
        ↓
[Container Runtime]   - capabilities, seccomp, AppArmor/SELinux
        ↓
[Linux Kernel]        - namespace 격리, cgroup, LSM
        ↓
[Hardware]           - TEE, SEV, TDX (선택)
```

각 계층마다 보안 메커니즘이 있고 모두 조합되어 "다층 방어".

---

## 1. 컨테이너 = 격리 vs 가상화

### 1.1 VM과의 차이

```
VM:
  [App] → [Guest OS] → [Hypervisor] → [Hardware]
  격리 강함, 무거움

Container:
  [App] → [공유 호스트 커널] → [Hardware]
  가벼움, 커널 공유 → 보안 약함
```

컨테이너 안 root가 호스트 커널 취약점을 찌르면 → 호스트 탈출 가능.

### 1.2 컨테이너의 격리 메커니즘 정리

| 메커니즘 | 격리하는 것 |
|---------|----------|
| namespace | "보이는 것" |
| cgroup | "쓸 수 있는 양" |
| capabilities | 권한 최소화 |
| seccomp | 허용 syscall 제한 |
| LSM (AppArmor/SELinux) | 파일/네트워크 정책 |
| user namespace | UID 매핑 |

---

## 2. Linux Capabilities

### 2.1 root의 분해

전통적으로 root = 모든 권한. 너무 강함.

→ **capabilities**: root 권한을 ~40개로 쪼갬.

| Capability | 의미 |
|------------|------|
| `CAP_NET_ADMIN` | 네트워크 설정 변경 |
| `CAP_SYS_ADMIN` | 시스템 관리 (가장 강력, 위험) |
| `CAP_SYS_PTRACE` | 다른 프로세스 추적 |
| `CAP_NET_RAW` | raw socket (ping 등) |
| `CAP_CHOWN` | 파일 소유자 변경 |
| `CAP_DAC_OVERRIDE` | 파일 권한 무시 |

### 2.2 컨테이너 기본 capabilities

도커/containerd 기본:
```
allowed: SETUID, SETGID, NET_BIND_SERVICE, AUDIT_WRITE, ...
dropped: SYS_ADMIN, SYS_PTRACE, NET_ADMIN, ...
```

루트라도 호스트 시스템 설정 못 바꿈.

### 2.3 K8s에서 제어

```yaml
spec:
  containers:
  - securityContext:
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]   # 80포트 바인드만
```

원칙: **drop ALL → 필요한 것만 add**. (최소 권한)

---

## 3. seccomp (Secure Computing)

### 3.1 무엇

프로세스가 호출 가능한 syscall을 **화이트리스트**로 제한.

```
[App] → syscall(read, write, open, ...)
         ↓
       [seccomp filter]
         ↓
       allow / deny / kill
```

### 3.2 도커 기본 프로필

도커는 약 60개 syscall을 차단 (전체 ~330개 중).
- `kexec_load`: 커널 교체
- `mount`: 파일시스템 마운트
- `reboot`: 재부팅
- 등등

### 3.3 K8s에서 적용

```yaml
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault   # 컨테이너 런타임 기본 프로필
      # 또는
      type: Localhost
      localhostProfile: profiles/audit.json
```

`RuntimeDefault`만 켜도 큰 보안 향상.

### 3.4 커스텀 프로필

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "exit"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

이 프로필 적용된 컨테이너는 위 5개 syscall만 가능. 매우 빡빡.

---

## 4. LSM (Linux Security Module)

### 4.1 무엇

커널의 보안 정책 hook 프레임워크. 대표 구현:
- **SELinux** (RHEL 계열)
- **AppArmor** (Ubuntu/Debian)
- **Tomoyo**, **Smack** 등

### 4.2 SELinux

라벨 기반 강제 접근 제어 (MAC).

```
파일/프로세스에 라벨 부여:
  /etc/passwd → system_u:object_r:passwd_file_t
  apache 프로세스 → system_u:system_r:httpd_t

정책: httpd_t는 passwd_file_t를 읽을 수만 있음
```

매우 강력하지만 복잡함. 디버깅 어려움.

### 4.3 AppArmor

경로 기반 정책. SELinux보다 단순.

```
profile docker-default {
  # 허용
  /usr/bin/myapp ix,
  /var/log/** w,
  # 거부
  deny /etc/shadow r,
  deny /sys/** w,
}
```

K8s에서:
```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/mycontainer: runtime/default
```

---

## 5. User Namespace - UID 매핑

### 5.1 컨셉

컨테이너 안 root(UID 0)를 호스트의 비특권 UID로 매핑.

```
[컨테이너 안] UID 0 (root) → [호스트] UID 100000 (일반 유저)
```

### 5.2 효과

- 컨테이너 root가 호스트 root 권한 못 가짐
- 컨테이너 탈출 시도해도 일반 유저 권한
- 보안 한 단계 추가

### 5.3 K8s 도입

K8s 1.30+ alpha:
```yaml
spec:
  hostUsers: false   # user namespace 활성화
```

아직 광범위하게 안 쓰임. Rootless Podman 등이 활용.

---

## 6. Pod Security Standards (PSS)

### 6.1 PSP 폐지

K8s 1.25부터 PodSecurityPolicy(PSP) 제거 → PSS로 대체.

### 6.2 3단계 프로필

| 프로필 | 의미 |
|--------|------|
| **privileged** | 제한 없음 |
| **baseline** | 권한 상승 차단 (기본) |
| **restricted** | 매우 빡빡 (최소 권한) |

### 6.3 적용

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

`enforce` 위반 → Pod 생성 거부.

### 6.4 restricted 요구사항

- non-root user 실행
- privileged 컨테이너 금지
- hostPath, hostNetwork, hostPID 금지
- capabilities는 NET_BIND_SERVICE 빼고 모두 drop
- seccomp RuntimeDefault 필수

---

## 7. RBAC (Role-Based Access Control)

(K8s control plane 문서에서 다룸. 핵심만 재정리)

### 7.1 원칙

**최소 권한 (least privilege)**.

나쁜 예:
```yaml
roleRef:
  kind: ClusterRole
  name: cluster-admin   # 모든 권한!
```

좋은 예:
```yaml
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]   # 필요한 것만
```

### 7.2 ServiceAccount Token

Pod 안에 자동 마운트되는 SA token이 강력하면 위험.

방어:
```yaml
spec:
  automountServiceAccountToken: false   # 필요 없으면 끔
```

---

## 8. Network Policy

### 8.1 기본은 "all open"

K8s 기본: 모든 Pod이 모든 Pod와 통신 가능. 위험.

### 8.2 NetworkPolicy로 제한

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-from-app
  namespace: prod
spec:
  podSelector:
    matchLabels: { app: db }
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: { app: api }
    ports:
    - port: 5432
```

`db` Pod에는 같은 네임스페이스의 `api` Pod에서 5432 포트로만 들어올 수 있음.

### 8.3 Default deny

```yaml
# 기본 모든 ingress 차단
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress: []     # 비워두면 모두 거부
```

이걸 깔고 필요한 통신만 열어주는 패턴.

### 8.4 CNI 지원 필요

NetworkPolicy는 K8s 표준이지만 **실제 강제는 CNI가 함**. Calico, Cilium 등은 지원. 일부 CNI는 미지원.

### 8.5 Cilium의 L7 정책

Calico/Cilium은 L4(IP/포트)까지. **Cilium은 eBPF로 L7도 가능**:

```yaml
# HTTP method/path 단위 정책
http:
- method: GET
  path: /api/v1/.*
```

---

## 9. Secret 관리

### 9.1 K8s Secret의 함정

```yaml
kind: Secret
data:
  password: cGFzc3dvcmQ=   # base64만! 암호화 아님!
```

base64는 인코딩이지 암호화가 아님. etcd에 평문에 가까운 상태로 저장됨.

### 9.2 etcd 암호화

```yaml
# kube-apiserver --encryption-provider-config=/etc/encryption.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: [secrets]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-encoded-32-byte-key>
```

이걸 켜야 etcd에서 진짜 암호화됨.

### 9.3 외부 secret 관리

더 안전한 방법: HashiCorp Vault, AWS Secrets Manager, Azure Key Vault 등.

```
[Vault] 마스터 시크릿 보유
   ↓ External Secrets Operator
[K8s Secret] 자동 동기화
   ↓
[Pod]
```

또는:
- **Sealed Secrets** (Bitnami): 암호화된 Secret을 git에 커밋 가능
- **SOPS**: Mozilla, ArgoCD에서 사용

### 9.4 Pod에서 Secret 노출 방식

```yaml
# 환경 변수 (덜 안전, 다른 프로세스가 볼 수 있음)
env:
- name: DB_PASS
  valueFrom:
    secretKeyRef: { name: db, key: password }

# 파일 마운트 (더 안전)
volumeMounts:
- name: secret
  mountPath: /etc/secret
  readOnly: true
volumes:
- name: secret
  secret:
    secretName: db
```

---

## 10. 이미지 보안

### 10.1 이미지 출처

- 신뢰할 수 있는 레지스트리만 (회사 Harbor 등)
- 베이스 이미지 관리 (작고 검증된 것)
- 정기적으로 베이스 업데이트

### 10.2 이미지 스캔

| 도구 | 특징 |
|------|------|
| **Trivy** | 가장 인기, 빠름 |
| **Grype** | Anchore |
| **Clair** | 오픈소스 |
| **Snyk** | 상용 |

```bash
trivy image myimage:v1
# CVE-2023-XXXX  HIGH  openssl  fixed in 1.1.1q
# ...
```

CI 파이프라인에 통합.

### 10.3 이미지 서명

**cosign** (sigstore): 이미지에 디지털 서명.

```bash
cosign sign --key cosign.key registry/image:v1
cosign verify --key cosign.pub registry/image:v1
```

K8s에서 admission controller로 "서명된 이미지만 허용" 정책 가능.

### 10.4 SBOM (Software Bill of Materials)

이미지가 어떤 라이브러리 포함하는지 명시. 취약점 발견 시 영향 추적 용이.

```bash
syft myimage:v1 -o spdx-json > sbom.json
```

---

## 11. 런타임 보안 (Runtime Security)

### 11.1 Falco

eBPF 기반 런타임 위협 탐지.

```yaml
# 규칙 예시
- rule: Shell in Container
  desc: Detect shell exec in containers
  condition: container.id != host and proc.name = bash
  output: "Shell spawned in %container.name"
  priority: WARNING
```

이상한 행동 발생 시 즉시 알람.

### 11.2 정책 엔진

**OPA Gatekeeper**, **Kyverno**: K8s 객체 생성 시 정책 강제.

```yaml
# Kyverno 예시
spec:
  rules:
  - name: require-non-root
    validate:
      message: "Must run as non-root"
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
```

`runAsNonRoot: true` 없는 Pod 생성 거부.

---

## 12. 우리 회사 환경 보안 체크리스트

### 12.1 컨테이너 레벨

- [ ] 모든 Pod `runAsNonRoot: true` (가능한 경우)
- [ ] capabilities drop ALL, add 최소
- [ ] seccompProfile RuntimeDefault
- [ ] hostNetwork/hostPID 금지 (필요 시 예외)
- [ ] readOnlyRootFilesystem 권장

### 12.2 K8s 레벨

- [ ] RBAC 최소 권한 원칙
- [ ] NetworkPolicy로 namespace 간 격리
- [ ] PSS restricted 적용 (가능한 namespace)
- [ ] etcd 암호화 활성화
- [ ] Audit logging 활성화

### 12.3 ML 워크로드 특수성

- [ ] 학습 Pod이 GPU/RDMA 디바이스 직접 접근 → securityContext 일부 완화 필요
- [ ] Host-Device CNI는 hostNetwork 비슷한 권한 → 정책 검토
- [ ] privileged 모드 피하고 capabilities로 세분화

### 12.4 Secret

- [ ] etcd 암호화
- [ ] Vault 또는 External Secrets 도입 검토
- [ ] 환경 변수보다 파일 마운트
- [ ] git에 평문 Secret 절대 커밋 금지

### 12.5 이미지

- [ ] 사내 Harbor 강제
- [ ] CI에서 Trivy 스캔
- [ ] cosign 서명 (장기 목표)

---

## 13. 면접 예상 질문

### Q1. "컨테이너가 VM보다 보안에 약한 이유는?"
> "VM은 hypervisor가 게스트 커널과 호스트 커널을 분리하지만, 컨테이너는 호스트 커널을 모든 컨테이너가 공유합니다. 컨테이너 안 root가 호스트 커널 취약점(예: 권한 상승 버그)을 찌르면 호스트 탈출이 가능합니다. 그래서 capabilities, seccomp, LSM, user namespace로 다층 방어를 합니다."

### Q2. "K8s Secret이 안전한가요?"
> "기본은 base64 인코딩일 뿐 평문에 가깝게 etcd에 저장됩니다. EncryptionConfiguration으로 etcd 암호화를 활성화해야 진짜 암호화되고, 더 안전하게는 Vault 같은 외부 시크릿 매니저를 External Secrets Operator로 동기화하는 패턴을 씁니다."

### Q3. "Pod Security Standards의 restricted 프로필이 강제하는 것은?"
> "non-root 실행, privileged 금지, hostPath/hostNetwork 금지, capabilities 거의 모두 drop, seccomp RuntimeDefault 필수 등입니다. 매우 빡빡해서 일반 앱은 적용 가능하지만 GPU/RDMA가 필요한 ML 워크로드는 일부 권한을 완화해야 할 수 있습니다."

### Q4. "Network Policy가 적용 안 되는 경우?"
> "NetworkPolicy는 K8s 표준 객체일 뿐, 실제 강제는 CNI가 합니다. Calico, Cilium은 지원하지만 일부 CNI는 미지원입니다. 또 Cilium은 eBPF로 L7(HTTP method/path)까지 정책 가능한 반면 다른 CNI들은 L4(IP/포트)까지만 지원합니다."

### Q5. "이미지 공급망 보안은 어떻게 강화하나요?"
> "신뢰할 수 있는 레지스트리(사내 Harbor 등)만 사용, 베이스 이미지 정기 업데이트, CI에서 Trivy/Grype로 CVE 스캔, cosign으로 이미지 서명, admission controller로 '서명되지 않은 이미지 거부' 정책 강제, SBOM 생성으로 라이브러리 추적 가능하게 합니다."

### Q6. "Falco는 어떻게 동작하나요?"
> "eBPF로 커널의 syscall과 컨테이너 이벤트를 관측하고, 정의된 규칙(예: '컨테이너 안에서 bash exec')에 매칭되는 행위를 실시간 탐지해 알람합니다. 침입 탐지/이상 행위 모니터링 용도로, 사후 분석이 아닌 런타임 방어 계층입니다."

---

## 14. 한 줄 요약

> **"컨테이너 보안 = 커널 격리 메커니즘(namespace, capabilities, seccomp, LSM, user ns)을 다층으로 쌓고, K8s 레벨(RBAC, NetworkPolicy, PSS)에서 정책으로 강제하며, 이미지 공급망(스캔, 서명, SBOM)과 런타임 위협 탐지(Falco)로 보완하는 것. 최소 권한 원칙(drop ALL → 필요한 것만 add)이 모든 것의 출발점."**
