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

전체 커리큘럼은 [../CURRICULUM.md](../CURRICULUM.md) 참고. 요약:

1. **기초** — `kernel/linux-fundamentals` → `kernel/cgroup` → `k8s/container-runtime` → `kernel/network`
2. **클러스터** — `k8s/k8s-control-plane` → `kernel/cni-kernel`
3. **HW** — `hw/gpu-gpudirect` → `k8s/nvidia-network-operator` → `hw/nccl-collective` → `hw/cuda-stack` → `hw/ceph-storage` → `hw/parallel-filesystem`
4. **MLOps 플랫폼** — `k8s/mlops-stack` → `k8s/inference-serving` → `k8s/multi-tenancy-scheduler` → `k8s/authn-authz` → `k8s/security`
5. **관측/통합** — `kernel/ebpf` → `k8s/observability` → `k8s/logging-pipeline` → `integration/llm-training-workload` → **`integration/dgx-ib-multinode-training-guide.md`** (허브)

## 운영 원칙

- 같은 예시(`mlx5_3`/`ibp64s0`/`0000:40:00.0`)를 여러 문서가 관통하도록 유지.
- 문서 간 상호참조는 상대경로(`../kernel/xxx.md`) 로.
- 외부 참고 자료(논문·블로그·치트시트)는 `guides/`가 아닌 [../references/](../references/) 로.
