# ArgoCD Bootstrap

Apply in this order:
```bash
# 1. Create namespace
kubectl apply -f bootstrap/argocd/namespace.yaml

# 2. Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Fix ApplicationSet CRD (known annotation size issue)
kubectl apply --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/crds/applicationset-crd.yaml

# 4. Apply insecure mode config (required for Traefik HTTP ingress)
kubectl apply -f bootstrap/argocd/argocd-cmd-params.yaml

# 5. Restart ArgoCD server and applicationset controller
kubectl rollout restart deployment argocd-server -n argocd
kubectl rollout restart deployment argocd-applicationset-controller -n argocd

# 6. Wait for ArgoCD to be ready
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# 7. Apply app-of-apps
kubectl apply -f argocd/app-of-apps.yaml
```
