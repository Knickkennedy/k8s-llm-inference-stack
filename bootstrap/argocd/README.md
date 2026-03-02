# ArgoCD Bootstrap

Apply in this order:
```bash
# 1. Create namespace
kubectl apply -f namespace.yaml

# 2. Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Apply insecure mode config (required for Traefik HTTP ingress)
kubectl apply -f argocd-cmd-params.yaml

# 4. Restart ArgoCD server to pick up config
kubectl rollout restart deployment argocd-server -n argocd

# 5. Apply app-of-apps
kubectl apply -f ../../argocd/app-of-apps.yaml
```
