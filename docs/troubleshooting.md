# Troubleshooting

## Prometheus Cannot Scrape Ollama Metrics

### Symptom
Prometheus targets page shows ollama job as `down` with `context deadline exceeded`.

### Cause
The ollama-metrics sidecar exposes metrics on port 8080, but the Ollama Kubernetes 
Service only exposed port 11434 by default. Prometheus on a different node could not 
reach the sidecar port across the cluster.

### Fix
Ensure the Ollama service exposes both ports:
```yaml
ports:
  - name: http
    port: 11434
    targetPort: 11434
  - name: metrics
    port: 8080
    targetPort: 8080
```

---

## Ollama Model Unloads From Memory

### Symptom
ollama-metrics sidecar logs show `Refreshed metrics data: 0 models loaded` and 
inference requests are slow as the model reloads on every request.

### Cause
On CPU-only nodes with limited RAM, Ollama aggressively evicts models from memory 
when idle. Default keep-alive is too short for homelab use.

### Fix
Set `OLLAMA_KEEP_ALIVE` environment variable in the Ollama deployment:
```yaml
env:
  - name: OLLAMA_KEEP_ALIVE
    value: "24h"
```

---

## Model Pull Job Stuck in Readiness Loop

### Symptom
The `ollama-pull-mistral` job pod loops on `Ollama not ready yet, waiting...` 
even though the Ollama pod is healthy and responding.

### Root Causes & Fixes

**1. curl not found**
The Ollama image does not include `curl`. Use `wget` or bash TCP sockets instead.

**2. wget not found / TCP socket fails**  
The `/dev/tcp` bash built-in requires `bash`, not `sh`. The Ollama image has bash 
at `/usr/bin/bash`, not `/bin/sh`. Always specify the full path:
```yaml
command:
  - /usr/bin/bash
  - -c
```

**3. wget returns non-zero on valid response**
Ollama's root endpoint returns a plain text response that wget treats as an error.
Use TCP socket check instead:
```bash
until (echo > /dev/tcp/ollama/11434) 2>/dev/null; do
  sleep 5
done
```

---

## ArgoCD LoadBalancer Stuck Pending

### Symptom
`kubectl get svc argocd-server -n argocd` shows `EXTERNAL-IP` as `<pending>` 
indefinitely.

### Cause
k3s ships with Traefik which already binds ports 80 and 443 across all nodes via 
Klipper LoadBalancer. A second LoadBalancer service cannot claim those ports.

### Fix
Use NodePort instead of LoadBalancer for ArgoCD:
```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'
```

---

## Ollama Pod Stuck Pending on Rolling Update

### Symptom
After updating the Ollama deployment, the new pod stays `Pending` with:
```
0/3 nodes are available: 1 Insufficient memory
```

### Cause
On a single inference node with 8GB RAM, the existing Ollama pod consumes ~6GB. 
There is insufficient memory to schedule a second pod for a rolling update.

### Fix
Manually delete the existing pod to free memory before the new one schedules:
```bash
kubectl delete pod -n ollama $(kubectl get pod -n ollama \
  -o jsonpath='{.items[0].metadata.name}')
```
Kubernetes will immediately reschedule using the updated deployment spec.

---

## Kubernetes Node Metrics Showing 400 Bad Request

### Symptom
Prometheus targets show `kubernetes-nodes` job as `down` with 
`server returned HTTP status 400 Bad Request` on port 10250.

### Cause
The kubelet metrics endpoint on port 10250 requires TLS client certificate 
authentication. Prometheus needs additional RBAC and TLS configuration to scrape it.

### Notes
Node-level metrics are not required for this project's inference observability goals. 
This is expected behavior and can be safely ignored. If node metrics are needed, 
consider deploying the full `kube-prometheus-stack` Helm chart which includes 
pre-configured TLS scraping.

---

## SSH Key Accidentally Committed to Repository

### Symptom
GitGuardian alerts with `OpenSSH Private Key detected in public repository`.

### Cause
Running `ssh-keygen` from inside the repository directory saves keys to the current 
working directory instead of `~/.ssh/`, and a subsequent `git add .` commits them.

### Prevention
Always specify an explicit path when generating SSH keys:
```bash
ssh-keygen -t ed25519 -C "your@email.com" -f ~/.ssh/id_ed25519
```

### Fix
If a key is accidentally committed:
1. Immediately revoke the key on GitHub: Settings → SSH Keys → Delete
2. Generate a new key pair
3. Scrub the key from git history:
```bash
pip install git-filter-repo
export PATH="$PATH:$HOME/.local/bin"
git filter-repo --path id_ed25519 --invert-paths --force
git remote add origin git@github.com:<username>/<repo>.git
git push origin main --force
```
4. Add SSH key patterns to `.gitignore`:
```
id_rsa
id_ed25519
id_ecdsa
*.pem
*.key
*.pub
```

---

## Ollama Metrics Missing Request-Level Data (Tokens, Latency)

### Symptom
Prometheus only shows three metrics for Ollama:
- `ollama_loaded_models`
- `ollama_model_loaded`
- `ollama_model_ram_mb`

Request-level metrics like `ollama_prompt_tokens_total`, 
`ollama_generated_tokens_total`, and `ollama_request_duration_seconds` 
are missing entirely.

### Cause
The ollama-metrics sidecar acts as a **transparent proxy** — it only captures 
request metrics for traffic routed **through it** on port 8080. If the Ollama 
service points directly to Ollama's port 11434, the sidecar never sees the 
requests and cannot record metrics for them.

### Fix
Update the Ollama service so port 11434 targets the sidecar on port 8080, 
which then proxies internally to Ollama:
```yaml
ports:
  - name: http
    port: 11434
    targetPort: 8080
  - name: metrics
    port: 8080
    targetPort: 8080
```

This ensures all inference traffic flows through the sidecar, populating 
the full set of request-level metrics without any changes to client code.
