# Metrics Server GitOps Runbook

Muc tieu:

```text
metrics-server duoc quan ly boi GitOps
khong apply manual bang URL tren VPS
Argo CD quan ly cai dat, sync, prune va self-heal
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
│   ├── kustomization.yaml
│   ├── platform-root-application.yaml
│   └── metrics-server-application.yaml
└── scripts/
    └── remove-metrics-server.sh
```

`monitoring/metrics-server/kustomization.yaml` pin metrics-server version:

```text
v0.8.1
```

Khong dung `latest`, vi GitOps nen co version co dinh.

Patch K3s/VPS:

```text
--kubelet-insecure-tls
```

Ly do:

```text
K3s lab/VPS hay dung kubelet cert self-signed.
metrics-server co the loi x509 neu verify TLS kubelet.
```

## 2. App-Of-Apps: Quan Ly Application Bang Git

Co 2 lop Application:

```text
platform-root
-> quan ly folder argocd/
-> tao/cap nhat/xoa cac Argo CD Application con

metrics-server
-> quan ly monitoring/metrics-server
-> cai metrics-server vao kube-system
```

Day la pattern production hay dung:

```text
bootstrap 1 root Application
-> sau do them/xoa app bang Git
```

Van can bootstrap `platform-root` mot lan dau. Sau do khong can tao Application moi bang tay nua.

## 3. Sua Repo URL Neu Can

File can sua:

```text
argocd/platform-root-application.yaml
argocd/metrics-server-application.yaml
```

Neu repo cua mày khac, doi:

```yaml
repoURL: https://github.com/shynsg/todo-platform.git
```

thanh repo platform that.

Commit va push:

```bash
git add monitoring/metrics-server argocd docs/metrics-server-gitops-runbook.md scripts/remove-metrics-server.sh
git commit -m "add metrics-server gitops"
git push
```

## 4. Bootstrap Platform Root Mot Lan

Day la buoc manual duy nhat de Argo CD bat dau quan ly folder `argocd/`.

Tren VPS, neu co repo platform:

```bash
kubectl apply -f argocd/platform-root-application.yaml
```

Neu khong pull repo tren VPS, tao qua Argo CD UI:

```text
Applications
-> New App
```

Dien:

```text
Application Name: platform-root
Project: default
Sync Policy: Automatic
Prune Resources: checked
Self Heal: checked
Repository URL: https://github.com/shynsg/todo-platform.git
Revision: main
Path: argocd
Cluster URL: https://kubernetes.default.svc
Namespace: argocd
```

Sau khi `platform-root` sync, no se tao:

```text
backend-prod
metrics-server
```

Kiem tra:

```bash
kubectl -n argocd get applications
```

## 5. Metrics Server Application

Sau khi `platform-root` quan ly folder `argocd/`, muon them metrics-server thi chi can commit:

```text
argocd/metrics-server-application.yaml
monitoring/metrics-server/
```

Khong can tao app bang tay nua.

## 6. Kiem Tra Sync

```bash
kubectl -n argocd get application platform-root
kubectl -n argocd get application metrics-server
```

Neu `metrics-server` chua xuat hien:

```bash
kubectl -n argocd describe application platform-root
```

Kiem tra metrics-server:

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
kubectl -n argocd describe application platform-root
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

```text
kiem tra patch --kubelet-insecure-tls da render/apply chua
```

Lenh check args:

```bash
kubectl -n kube-system get deploy metrics-server \
  -o jsonpath='{.spec.template.spec.containers[0].args}'
```

Phai thay:

```text
--kubelet-insecure-tls
```

## 8. Remove Metrics Server Theo GitOps

Cach dung GitOps:

```text
xoa argocd/metrics-server-application.yaml khoi repo
xoa dong metrics-server-application.yaml trong argocd/kustomization.yaml
git commit
git push
platform-root auto sync
Argo CD prune metrics-server Application
metrics-server Application prune metrics-server resources
```

Day la cach nen dung khi muon quan ly server config bang Git.

## 9. Remove Metrics Server Bang Script

Neu Argo CD Application co finalizer:

```yaml
finalizers:
  - resources-finalizer.argocd.argoproj.io
```

thi xoa Application se prune resource metrics-server.

Chay script:

```bash
cd todo-platform
./scripts/remove-metrics-server.sh
```

Luu y:

```text
Neu platform-root van quan ly argocd/metrics-server-application.yaml,
thi script xoa xong Argo CD co the tao lai metrics-server.
Muon xoa han, phai xoa manifest trong Git theo buoc 8.
```

Hoac lenh truc tiep:

```bash
kubectl -n argocd delete application metrics-server
```

Kiem tra da xoa:

```bash
kubectl -n argocd get application metrics-server
kubectl -n kube-system get deploy metrics-server
kubectl top nodes
```

Sau khi xoa, `kubectl top` se khong con dung duoc.

## 10. Checklist

```text
[ ] metrics-server config nam trong todo-platform/monitoring/metrics-server
[ ] metrics-server Application nam trong todo-platform/argocd
[ ] platform-root Application da tao
[ ] platform-root quan ly folder argocd/
[ ] Application metrics-server Synced
[ ] Application metrics-server Healthy
[ ] deploy/metrics-server Running
[ ] kubectl top nodes chay duoc
[ ] kubectl top pods -A chay duoc
[ ] kubectl -n apps-prod top pods chay duoc
```
