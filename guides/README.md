# guides/

현대차 DevOps/MLOps 면접 대비 딥다이브. 공통 구조:
**큰 그림 → 계층 분해 → 실습 → 회사 특화 → 면접 Q&A → 한 줄 요약**.

## 서브 디렉터리

| 폴더 | 범위 |
| --- | --- |
| [integration/](integration/) | 여러 계층을 통합하는 허브 문서 (DGX/IB/K8s 통합) |
| [k8s/](k8s/) | K8s 컨트롤 플레인·런타임·Operator·관측·보안 |
| [kernel/](kernel/) | 리눅스 커널·cgroup·eBPF·네트워크·CNI 내부 |
| [hw/](hw/) | GPU/NVLink/GPUDirect, Ceph 스토리지 |

## 읽는 순서 추천

1. `kernel/linux-fundamentals-deep-dive.md` → `kernel/cgroup-deep-dive.md` → `k8s/container-runtime-deep-dive.md`
2. `k8s/k8s-control-plane-deep-dive.md` → `kernel/network-deep-dive.md` → `kernel/cni-kernel-deep-dive.md`
3. `hw/gpu-gpudirect-deep-dive.md` → `hw/ceph-storage-deep-dive.md` → `k8s/nvidia-network-operator-deep-dive.md`
4. `kernel/ebpf-deep-dive.md` → `k8s/observability-deep-dive.md` → `k8s/security-deep-dive.md` → **`integration/dgx-ib-multinode-training-guide.md`**

## 운영 원칙

- 같은 예시(`mlx5_3`/`ibp64s0`/`0000:40:00.0`)를 여러 문서가 관통하도록 유지.
- 문서 간 상호참조는 상대경로(`../kernel/xxx.md`) 로.
- 외부 참고 자료(논문·블로그·치트시트)는 `guides/`가 아닌 [../references/](../references/) 로.
