# AuthN/AuthZ 딥다이브 — Keycloak + Dex + Istio + OIDC

> **목적**: "Kubeflow에 SSO 붙였다"라는 한 줄 뒤의 **OIDC code flow, JWT 검증, Istio RequestAuthentication, Keycloak realm/client, Dex connector** 구조를 계층별로 재현.
> **선행**: [security-deep-dive.md](security-deep-dive.md), [mlops-stack-deep-dive.md](mlops-stack-deep-dive.md)

---

## 0. 큰 그림

```
Browser
  │  (1) GET /kubeflow
  ▼
Istio Gateway (envoy)
  │  (2) ext_authz → oauth2-proxy/AuthService
  ▼  (3) redirect to Dex
Dex (OIDC provider)
  │  (4) upstream IdP = Keycloak
  ▼
Keycloak (realm: dnotitia)
  │  (5) 사용자 login → authorization code
  ▼  (6) redirect back with code
Dex → token endpoint → Keycloak
  │  (7) ID token (JWT) 받음
  ▼
oauth2-proxy (AuthService)
  │  (8) 쿠키 설정, JWT를 요청 헤더에 attach
  ▼
Istio Gateway → Kubeflow / KServe
  │  (9) RequestAuthentication: JWT 검증
  │ (10) AuthorizationPolicy: RBAC
  ▼
Pod
```

**면접 프레임**: 인증 계층은 **"identity 발급"(Keycloak) → "OIDC broker"(Dex) → "메쉬 gateway 검증"(Istio) → "namespace RBAC"(K8s)** 4단계.

---

## 1. OIDC 기초

### 1.1 OAuth2 vs OIDC

- **OAuth2** = 인가(authorization) 프로토콜. 제3자 리소스 접근 delegation.
- **OIDC** = OAuth2 위에 **인증(authentication)** 을 얹은 확장. **ID Token(JWT)** 발급 추가.

### 1.2 Authorization Code Flow (with PKCE)

```
[User] ──(1) GET /app────▶ [RP (Relying Party)]
[User] ◀─(2) 302 → /authorize?client_id=...&code_challenge=... [IdP]
[User] ──(3) login──────▶ [IdP]
[User] ◀─(4) 302 → callback?code=xxx
[RP]   ──(5) POST /token (code + code_verifier)──▶ [IdP]
[RP]   ◀─(6) { id_token, access_token, refresh_token }
[RP]   ──(7) 세션 쿠키 set
```

**ID Token** = JWT. 다음 3부분 `header.payload.signature`.

### 1.3 JWT 구조

```json
// Header
{ "alg": "RS256", "kid": "abc123", "typ": "JWT" }

// Payload (claims)
{
  "iss": "https://keycloak.dnotitia.com/realms/dnotitia",
  "sub": "a1b2c3d4-...",                  // user UUID
  "aud": "kubeflow",                      // client ID
  "exp": 1712345678,
  "iat": 1712342078,
  "email": "cwkim@dnotitia.com",
  "groups": ["platform-admins"]
}

// Signature — IdP 개인키로 서명
```

### 1.4 JWK / JWKS

IdP가 공개키를 `/.well-known/jwks.json`에 노출. RP(Istio 등)는 **kid로 조회해 signature 검증**. 키 로테이션 시 `jwks_uri`를 주기 refetch.

---

## 2. Keycloak

### 2.1 Realm

**트러스트 도메인 하나**. 회사에선 `dnotitia` realm.

- **Users**: 사용자 엔트리 (email, groups, attributes)
- **Clients**: OAuth2 client (kubeflow, grafana, harbor, airflow 등)
- **Roles / Groups**: 역할 계층
- **Identity Providers**: 외부 IdP 연동 (Google, SAML 등) — 우리는 사용 안 함
- **User Federation**: LDAP/AD 동기화 (있다면)

### 2.2 Client 종류

- **public** — 브라우저 SPA (client secret 없음, PKCE 필수)
- **confidential** — 서버사이드 (client_secret 사용)
- **bearer-only** — token 검증만, login 안 받음

Kubeflow/Dex 조합에선 **Dex가 public client** (PKCE).

### 2.3 Client Scope / Mapper

- **Scope**: 어떤 claim을 token에 포함할지.
- **Mapper**: K8s groups 매핑, custom attribute 주입.

예: "이 사용자가 속한 Keycloak groups를 `groups` claim으로" mapper 설정 → Kubeflow 쪽에서 namespace RBAC 연동.

### 2.4 Keycloak + Harbor 연동 (실제 사례)

Confluence 95912565:
- Harbor는 **OIDC provider로 Keycloak** 등록.
- 사용자는 Harbor UI 접근 시 Keycloak로 리다이렉트.
- CLI 용도로는 Harbor에서 **CLI Secret** 발급 (기본 패스워드 불가).

---

## 3. Dex

### 3.1 역할

OIDC **프록시/broker**. Upstream IdP(LDAP, Keycloak, Google 등)를 표준 OIDC로 abstraction.

**왜 Keycloak 앞에 Dex를 두는가?**
- Kubeflow는 원래 Dex를 기본 IdP로 설계 — Dex를 빼면 통합 설정 복잡.
- Dex는 **staticPasswords**, **LDAP**, **upstream OIDC** 여러 connector를 동시에.
- Keycloak 장애 시 Dex의 staticPassword로 임시 로그인 가능 (운영 옵션).

### 3.2 Dex connector (Keycloak 연동)

```yaml
# dex-config
connectors:
- type: oidc
  id: keycloak
  name: Keycloak
  config:
    issuer: https://keycloak.dnotitia.com/realms/dnotitia
    clientID: dex
    clientSecret: ...
    redirectURI: https://dex.dnotitia.com/callback
    scopes:
    - openid
    - email
    - groups
```

### 3.3 Dex client (Kubeflow 쪽)

```yaml
staticClients:
- id: kubeflow-oidc-authservice
  redirectURIs:
  - https://kubeflow.dnotitia.com/authservice/oidc/callback
  name: Kubeflow
  secret: ...
```

---

## 4. Istio AuthService (oauth2-proxy 패턴)

### 4.1 역할

Istio의 Gateway가 요청 받을 때 **인증 쿠키/세션이 있는지 확인** → 없으면 Dex로 리다이렉트. 있으면 JWT를 헤더에 주입해 upstream 서비스로.

구현 옵션:
- **oauth2-proxy** (Kubeflow 기본 채택)
- **AuthService** (istio-ecnode 공식 예제)
- 커스텀 envoy ext_authz

### 4.2 envoy ext_authz 훅

Istio gateway envoy 설정:
```yaml
http_filters:
- name: envoy.filters.http.ext_authz
  typed_config:
    grpc_service:
      envoy_grpc:
        cluster_name: oauth2-proxy
```

모든 요청 → oauth2-proxy → OK면 통과.

### 4.3 RequestAuthentication (Istio CRD)

envoy 자체의 JWT 검증 기능:

```yaml
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: dex-jwt
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  jwtRules:
  - issuer: "https://dex.dnotitia.com"
    jwksUri: "https://dex.dnotitia.com/keys"
    forwardOriginalToken: true
```

**forwardOriginalToken** — upstream으로 token 원본 전달 (Kubeflow 내부 서비스가 추가 검증).

### 4.4 AuthorizationPolicy

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: require-auth
  namespace: kubeflow
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals: ["*"]    # JWT가 있기만 하면
  - to:
    - operation:
        paths: ["/healthz"]         # healthz는 예외
```

**action 옵션**: `ALLOW` (기본), `DENY`, `AUDIT`, `CUSTOM` (ext_authz).

---

## 5. Kubernetes API 접근 (kubectl)

### 5.1 OIDC 플러그인 방식

`~/.kube/config`:

```yaml
users:
- name: cwkim
  user:
    auth-provider:
      name: oidc
      config:
        idp-issuer-url: https://dex.dnotitia.com
        client-id: kubectl
        id-token: ey...
        refresh-token: ...
```

kubectl 호출 시:
1. ID Token 만료 확인.
2. 만료면 refresh token으로 재발급.
3. `Authorization: Bearer <id_token>` 헤더로 apiserver 호출.

### 5.2 apiserver --oidc-* 플래그

```
--oidc-issuer-url=https://dex.dnotitia.com
--oidc-client-id=kubectl
--oidc-username-claim=email
--oidc-groups-claim=groups
```

apiserver는 JWT 검증 + `User` 객체 구성 (username, groups).

### 5.3 RBAC 매핑

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-admins
subjects:
- kind: Group
  name: "platform-admins"       # JWT groups claim
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

### 5.4 대안: kubelogin (kubectl-oidc_login)

krew plugin. OIDC 플로우를 더 잘 지원하며 refresh 처리 견고. `~/.kube/config`의 `exec` 타입으로 호출.

---

## 6. 서비스 간 인증 (mTLS)

### 6.1 Istio 자동 mTLS

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT           # 또는 PERMISSIVE
```

- **STRICT** — 평문 거부.
- **PERMISSIVE** — mTLS/평문 둘 다 (마이그레이션용).

### 6.2 SPIFFE ID

Istio는 자동으로 `spiffe://cluster.local/ns/<ns>/sa/<sa>` ID 발급. AuthorizationPolicy에서 `source.principals` 로 참조:

```yaml
rules:
- from:
  - source:
      principals: ["cluster.local/ns/kubeflow/sa/notebook-controller"]
```

---

## 7. ServiceAccount Token

### 7.1 Pod → API 호출

Pod은 `/var/run/secrets/kubernetes.io/serviceaccount/token`의 JWT로 apiserver 호출.

K8s 1.24+: **TokenRequest API** 기반 bound token — 만료 시간, audience, 자동 로테이션.

### 7.2 Projected Service Account Token

```yaml
volumes:
- name: api-token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: https://kubernetes.default.svc
```

**장점**: 토큰 수명 제한, audience별 분리 → 탈취 피해 축소.

### 7.3 Workload Identity 패턴

클라우드에서 Pod이 외부 서비스(S3 등) 호출할 때 SA token을 **IAM role로 swap**. EKS IRSA, GKE WIF.

온프레에선 Vault + SA token 검증 으로 대체 가능.

---

## 8. 장애/트러블

### 8.1 "로그인 무한 redirect"

- **원인 후보**: oauth2-proxy 쿠키 도메인 불일치, TLS 인증서 문제, Dex redirect URI 미등록.
- 체크: 브라우저 dev tools로 리다이렉트 체인, `kubectl logs oauth2-proxy`.

### 8.2 "Keycloak login은 성공하는데 Kubeflow는 403"

- JWT groups claim이 없어 AuthorizationPolicy 거부.
- Keycloak → Dex scope에 groups 포함 확인. Keycloak mapper 설정 확인.

### 8.3 "kubectl이 `Unauthorized`"

- ID token 만료 + refresh 실패.
- `kubectl oidc-login setup` 재실행, 또는 `kubelogin get-token` 수동 호출로 진단.

### 8.4 "Harbor OIDC 로그인 후 docker login 실패"

- Harbor는 OIDC 사용자에게 **CLI Secret 발급 후 사용** 필요. 일반 패스워드로 login 불가. Harbor UI > User Profile > CLI Secret 확인.

### 8.5 "JWT 서명 검증 실패 (key rotation)"

- IdP가 키를 돌렸는데 RP가 JWKS 캐시 갱신 안 함.
- Istio envoy는 주기적 refetch. `jwks_uri` 응답 TTL 확인, envoy stat `jwt_authn.jwks_fetch_success`.

---

## 9. 회사 실제 구성 요약

| 계층 | 컴포넌트 | 비고 |
|------|----------|------|
| IdP | **Keycloak** (realm: dnotitia) | LDAP 연동 유무에 따라 |
| OIDC broker | **Dex** | Kubeflow 기본 유지 목적 |
| Gateway | **Istio + oauth2-proxy (AuthService)** | 쿠키 세션 + JWT 주입 |
| JWT 검증 | **Istio RequestAuthentication** | jwks_uri = Dex |
| 인가 | **Istio AuthorizationPolicy** + **K8s RBAC** | namespace별 |
| Harbor | 자체 OIDC = Keycloak | CLI Secret으로 docker login |
| Airflow UI | oauth2-proxy 또는 Airflow native OIDC | auth manager 변경 이력 있음(ML-30) |
| MLflow | 자체 인증 없음 → Istio layer에서 방어 | |

---

## 10. 실습

### 10.1 JWT 디코드

```bash
echo "ey..." | cut -d. -f2 | base64 -d | jq .
```

봐야 할 claim: `iss`, `aud`, `exp`, `email`, `groups`.

### 10.2 Keycloak 토큰 요청 (client_credentials)

```bash
curl -X POST https://keycloak.dnotitia.com/realms/dnotitia/protocol/openid-connect/token \
  -d grant_type=client_credentials \
  -d client_id=mytool \
  -d client_secret=...
```

### 10.3 Istio JWT 디버깅

```bash
# gateway envoy stats
kubectl -n istio-system exec deploy/istio-ingressgateway -- \
  curl -s localhost:15000/stats | grep jwt_authn
```

### 10.4 oauth2-proxy 로그

```bash
kubectl -n istio-system logs deploy/oauth2-proxy --tail=200
```

---

## 11. 면접 Q&A

### Q1 (기초). OAuth2와 OIDC 차이 한 줄?
A. OAuth2는 인가, OIDC는 OAuth2 + 인증(ID Token). OIDC의 ID Token은 JWT로 사용자 정보(sub, email, groups) 포함.

### Q2. Dex를 왜 Keycloak 앞에 두는가?
A. (1) Kubeflow가 Dex 전제로 설계. (2) Dex는 여러 upstream connector를 추상화. (3) Keycloak 장애 시 staticPassword로 임시 로그인 가능. 단점은 홉 하나 추가 — 운영 책임 컴포넌트 증가.

### Q3. Istio RequestAuthentication과 AuthorizationPolicy 차이?
A. RequestAuthentication = **JWT 검증** (서명, 만료, iss). 통과하면 envoy의 metadata에 principals 기록. AuthorizationPolicy = **인가 결정** — principals/source/paths 기반 ALLOW/DENY. 순서: Authentication → Authorization.

### Q4. JWT 서명 키 로테이션 시 무엇을 신경써야?
A. IdP가 `jwks_uri`에 새 키 추가 (이전 키 당분간 유지) → RP들이 JWKS refetch → 이전 키 제거. Istio envoy는 `remote_jwks.cacheDuration` 내에서 refetch. **오프라인 jwks 박아두면 안 됨** — 로테이션 시 서비스 중단.

### Q5. PKCE를 public client에 왜 쓰나?
A. Public client(SPA, CLI)는 client_secret 안전 보관 불가. authorization code를 가로채도 `code_verifier` 없으면 token 교환 실패. `code_challenge = SHA256(code_verifier)` 를 authorize 요청에 먼저 심음.

### Q6. K8s apiserver의 `--oidc-groups-claim` 이 중요한 이유?
A. RBAC의 `kind: Group` subject와 매칭되는 claim 이름 지정. claim 이름이 틀리면 사용자는 인증되지만 권한 매핑 안 돼서 `Forbidden`. 회사는 보통 `groups`.

### Q7. oauth2-proxy가 없어도 되나?
A. Istio의 ext_authz + RequestAuthentication만으로 **JWT 검증**은 가능. 하지만 브라우저에 **쿠키 세션** 관리(login 유도, state 관리, logout)는 필요 — oauth2-proxy가 해당 역할. API-only endpoint라면 oauth2-proxy 생략 가능.

### Q8. ServiceAccount projected token이 왜 보안상 권장?
A. (1) **TTL** (기본 1시간). (2) **audience binding** — 탈취해도 다른 service엔 못 씀. (3) 자동 로테이션. 과거 Secret 타입 SA token은 만료 없어서 유출 시 영구 유효.

### Q9. Harbor docker login 실패 해결?
A. OIDC 사용자는 UI는 되지만 docker login은 User Profile에서 CLI Secret 발급 후 그것으로. docker daemon이 OIDC flow 못 돌리므로 basic auth 대체용.

### Q10. Kubeflow Profile이 RBAC/AuthorizationPolicy 자동 만드는 흐름?
A. Profile CR 생성 → Profile Controller가 (1) namespace, (2) RoleBinding (namespaceAdmin → owner email), (3) Istio AuthorizationPolicy (이 namespace는 이 사용자의 JWT principals만 허용) 생성. 사용자 onboarding을 CR 하나로 완결.

---

## 12. 체크리스트

- [ ] Authorization Code Flow 8단계 설명
- [ ] JWT 구조 + kid/jwks
- [ ] Keycloak realm/client 3가지 client 타입
- [ ] Dex connector, staticClient 역할
- [ ] Istio RequestAuthentication vs AuthorizationPolicy
- [ ] PKCE 동작 방식
- [ ] OIDC groups claim → RBAC Group 매핑
- [ ] SA projected token 장점 3가지
- [ ] Harbor CLI Secret 필요 이유
- [ ] 로그인 redirect 루프 트리아지 3단계

---

## 13. 참고

- OIDC spec: https://openid.net/specs/openid-connect-core-1_0.html
- Keycloak: https://www.keycloak.org/documentation
- Dex: https://dexidp.io/
- Istio security: https://istio.io/latest/docs/concepts/security/
- 관련: [security-deep-dive.md](security-deep-dive.md), [mlops-stack-deep-dive.md](mlops-stack-deep-dive.md), [inference-serving-deep-dive.md](inference-serving-deep-dive.md)
