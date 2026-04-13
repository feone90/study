# TODO & 작업 히스토리

## 다음에 할 것 (우선순위 순)

- [ ] **1. 커밋 정리** — 아래 3개 논리 단위로 분할 커밋
  - 리뷰 패치: `guides/integration/dgx-ib-multinode-training-guide.md`, `guides/k8s/nvidia-network-operator-deep-dive.md`
  - 폴더 재구조화 + 각 서브폴더 README.md, 상호참조 경로 수정
  - CLAUDE.md 정비 (메타 규칙 추가)
- [ ] **2. 나머지 12개 딥다이브 리뷰** — [CLAUDE.md](CLAUDE.md) 의 8축 체크리스트 적용
  - `guides/kernel/`: linux-fundamentals, cgroup, network, cni-kernel, ebpf
  - `guides/k8s/`: k8s-control-plane, container-runtime, nvidia-network-operator, observability, security
  - `guides/hw/`: gpu-gpudirect, ceph-storage
- [ ] **3. `references/` 채우기** — IB 스펙, NCCL 튜닝 가이드, KEP 등 외부 자료 정리
- [ ] **4. 문서 간 상호참조 감사** — 허브 외 문서들끼리도 연결 보강 (인바운드 링크)
- [ ] **5. (선택) 마크다운 린트 통일** — MD022/MD032 경고 전반 정리
- [ ] **6. (선택) `.gitignore` 추가** — `.DS_Store`, `.claude/` 제외

---

## 작업 히스토리

### 2026-04-13

#### 리뷰 패치 — `guides/integration/dgx-ib-multinode-training-guide.md`
8축 체크리스트 리뷰 후 P0/P1/P2 반영.

- **P0**: NCCL_IB_TIMEOUT 공식 수정 (`4.096μs × 2^value` 기준, 값 22 ≈ 17초)
- **P1**:
  - PCIe 토폴로지 정정 (GPU:HCA 2:1 → 1:1 페어링)
  - §2.4 신설 — AllReduce 알고리즘 (Ring vs Tree vs NVLS/SHARP) + 튜닝 레버 (NCCL_ALGO/PROTO/COLLNET_ENABLE/NVLS_ENABLE/TOPO_FILE)
  - §2.5 신설 — GPUDirect RDMA 원리 (bounce-buffer vs P2P, 활성 조건, `GDRDMA` 로그 검증)
  - `rdma link` netns 인식 커널 5.3+ caveat 추가
  - 교차 문서 링크(`cni-kernel`, `nvidia-network-operator`) 추가
- **P2**:
  - §1.5 난이도 램프 이스케이프 해치 (주니어는 건너뛰어도 된다는 안내)
  - Q2/Q5/Q8 꼬리질문 추가 (SR-IOV vs MIG, Ring vs Tree 선택, SM 장애)
  - Q8 트러블슈팅 순서 물리→상위로 재정렬

#### 폴더 재구조화
기존 `guides/` 평면 13개 → 4개 서브폴더로 계층화 + `references/` 신설.

- `guides/kernel/` (5): linux-fundamentals, cgroup, network, cni-kernel, ebpf
- `guides/k8s/` (5): k8s-control-plane, container-runtime, nvidia-network-operator, observability, security
- `guides/hw/` (2): gpu-gpudirect, ceph-storage
- `guides/integration/` (1): dgx-ib-multinode-training-guide (허브)
- `references/`: 외부 참고 자료 플레이스홀더
- 각 서브폴더 `README.md` 작성, 최상위 `guides/README.md` 가 포인터 역할
- 허브 문서 내부 상호참조 3곳을 상대경로(`../kernel/`, `../k8s/`) 로 수정

#### CLAUDE.md 정비
- 66줄 → 64줄로 슬림화
- 문서 집합 섹션을 `guides/README.md` 포인터로 축소
- "메타 규칙" 섹션 추가 (60줄 유지 + 파일 구조 상시 점검 + 네이밍 통일)

#### 기타
- `guides/INDEX.md` (임시 인덱스) 는 `guides/README.md` 로 대체 후 삭제
- 포트폴리오/피드백 문서(`portfolio_*.md`, `v8/v12/v13_feedback.md`) 는 외부 레포 (`workspace-cwkim/cwkim/guides/`) 로 이미 분리돼 있어 본 레포에서는 제거된 상태
