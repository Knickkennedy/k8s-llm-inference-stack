# k8s-llm-inference-stack

A production-grade, GitOps-managed LLM inference stack running on a bare-metal k3s cluster.

## Stack
- **Inference**: Ollama (Mistral 7B Q4) on dedicated inference node
- **Orchestration**: Kubernetes (k3s) with node-pinned workloads
- **GitOps**: ArgoCD for declarative, git-driven deployments
- **Observability**: Prometheus + Grafana with inference-specific dashboards
- **Gateway**: Nginx API gateway with rate limiting

## Cluster Architecture
- `cp1` — Control plane (Raspberry Pi 4, ARM64)
- `cp2` — Inference node (Mini PC, AMD64, 8GB RAM)
- `cp3` — Observability node (Mini PC, AMD64, 8GB RAM)

## Getting Started
Documentation in progress. See `bootstrap/` for cluster setup.
