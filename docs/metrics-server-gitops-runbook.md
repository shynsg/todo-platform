# Metrics Server GitOps Runbook

Muc tieu:

```text
metrics-server duoc quan ly boi Argo CD
config nam trong todo-platform/monitoring/metrics-server
khong apply manual URL tren VPS
```

## 1. Cau Truc File

```text
todo-platform/
├── monitoring/
│   ├── README.md
│   └── metrics-server/
│       ├── kustomization.yaml
│       └── metrics-server-k3s-patch.yaml
├── argocd/
│   └── metrics-server-application.yaml
└── scripts/
    └── remove-metrics-server.sh
```

`monitoring/metrics-server/kustomization.yaml` pin metrics-server version:

```text
v0.8.1
```

Patch cho K3s/VPS lab:

```text
--kubelet-insecure-tls
```

Ly do:

```text
K3s lab/VPS hay dung kubelet cert self-signed.
metrics-server co the loi x509 neu verify TLS kubelet.
```

## 2. Mental Model

Chi co 1 Argo CD Application:

```text
metrics-server Application
-> repo todo-platform
-> path monitoring/metrics-server
-> deploy metrics-server vao kube-system
```

Khong dung `platform-root` trong lesson nay.

Sau nay neu co nhieu tool nhu Datadog, cert-manager, external-secrets thi moi hoc app-of-apps/root app.

## 3. Commit Config Len Todo Platform

Tren local:

```bash
cd lesson-12-capstone-platform/todo-platform
```

Kiem tra file:

```bash
ls monitoring/metrics-server
ls argocd/metrics-server-application.yaml
```

Commit va push:

```bash
git add monitoring/metrics-server argocd/metrics-server-application.yaml docs/metrics-server-gitops-runbook.md monitoring/README.md scripts/remove-metrics-server.sh
git commit -m "add metrics-server gitops"
git push
```

## 4. Tao Metrics Server Application Trong Argo CD UI

Mo Argo CD:

```text
http://SERVER_IP/argocd
```

Tao app:

```text
Applications
-> New App
```

Dien:

```text
Application Name: metrics-server
Project: default
Sync Policy: Automatic
Prune Resources: checked
Self Heal: checked
Repository URL: https://github.com/shynsg/todo-platform.git
Revision: main
Path: monitoring/metrics-server
Cluster URL: https://kubernetes.default.svc
Namespace: kube-system
```

Sau do:

```text
Create
-> Sync
```

## 5. Tao Metrics Server Application Bang Kubectl

Neu muon tao bang CLI tren VPS:

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-server
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/shynsg/todo-platform.git
    targetRevision: main
    path: monitoring/metrics-server
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
EOF
```

## 6. Kiem Tra Sync

Tren VPS:

```bash
kubectl -n argocd get application metrics-server
kubectl -n argocd describe application metrics-server
```

Kiem tra resource:

```bash
kubectl -n kube-system get deploy metrics-server
kubectl -n kube-system get pods -l k8s-app=metrics-server
```

Kiem tra rollout:

```bash
kubectl -n kube-system rollout status deploy/metrics-server
```

Kiem tra metrics:

```bash
kubectl top nodes
kubectl top pods -A
kubectl -n apps-prod top pods
```

Neu `kubectl top` chua co data ngay, doi 30-60 giay roi chay lai.

## 7. Debug

Neu Application fail:

```bash
kubectl -n argocd describe application metrics-server
kubectl -n argocd logs deploy/argocd-repo-server --tail=100
kubectl -n argocd logs deploy/argocd-application-controller --tail=100
```

Neu metrics-server pod fail:

```bash
kubectl -n kube-system describe pod -l k8s-app=metrics-server
kubectl -n kube-system logs deploy/metrics-server --tail=100
```

Neu `kubectl top nodes` loi x509:

```bash
kubectl -n kube-system get deploy metrics-server \
  -o jsonpath='{.spec.template.spec.containers[0].args}'
```

Phai thay:

```text
--kubelet-insecure-tls
```

Neu khong thay, Argo CD chua sync dung path hoac patch chua apply.

## 8. Remove Metrics Server

Xoa bang Argo CD UI:

```text
Applications
-> metrics-server
-> Delete
-> checked cascade/prune neu UI hoi
```

Hoac chay script tren VPS/local co kubectl:

```bash
cd todo-platform
./scripts/remove-metrics-server.sh
```

Script nay xoa Argo CD Application:

```bash
kubectl -n argocd delete application metrics-server
```

Vi Application co finalizer:

```yaml
resources-finalizer.argocd.argoproj.io
```

nen Argo CD se prune resource metrics-server.

Kiem tra da xoa:

```bash
kubectl -n argocd get application metrics-server
kubectl -n kube-system get deploy metrics-server
kubectl top nodes
```

Sau khi xoa, `kubectl top` se khong con dung duoc.

## 9. Checklist

```text
[ ] metrics-server config nam trong todo-platform/monitoring/metrics-server
[ ] metrics-server Application YAML nam trong todo-platform/argocd
[ ] metrics-server Application duoc tao trong Argo CD
[ ] Application metrics-server Synced
[ ] Application metrics-server Healthy
[ ] deploy/metrics-server Running
[ ] kubectl top nodes chay duoc
[ ] kubectl top pods -A chay duoc
[ ] kubectl -n apps-prod top pods chay duoc
```
