# Runbook VPS: Deploy Todo Platform Bang IP, Argo CD Da Cai San

Runbook nay bat dau tu tinh trang:

```text
VPS da cai K3s
VPS da cai Argo CD
Chua co domain
Chi dung SERVER_IP
Chua thay namespace apps-prod
```

Dieu quan trong can nho:

```text
apps-prod khong tu xuat hien khi cai Argo CD.
apps-prod chi xuat hien sau khi Argo CD Application duoc tao va sync dung repo/path.
```

Ket qua cuoi:

```text
http://SERVER_IP/            -> frontend
http://SERVER_IP/api/health  -> backend
http://SERVER_IP/api/todos   -> todo API
```

## 0. Dien Bien Can Xay Ra

Flow dung tren production:

```text
GitHub Actions da build image
-> image da co tren GHCR
-> todo-platform repo da co YAML
-> Argo CD doc todo-platform repo
-> Argo CD apply YAML vao K3s
-> K3s tao namespace apps-prod
-> K3s pull image tu GHCR
-> Pod chay
```

Server khong can:

```text
git pull source backend/frontend
npm install
docker build
```

Server can:

```text
K3s
Argo CD
Traefik ingress
Network ra internet de pull image
```

## 1. SSH Vao VPS

Tu may ca nhan:

```bash
ssh root@SERVER_IP
```

Kiem tra dang o dung server:

```bash
hostname
whoami
```

## 2. Kiem Tra Kubectl/K3s

Chay:

```bash
kubectl get nodes
```

Neu bao `kubectl: command not found`, dung:

```bash
k3s kubectl get nodes
```

Neu muon tien hon:

```bash
alias kubectl='k3s kubectl'
```

Ky vong:

```text
STATUS
Ready
```

Kiem tra namespace hien tai:

```bash
kubectl get ns
```

Luc nay **chua thay `apps-prod` la binh thuong** neu chua tao/sync Argo CD Application.

## 3. Kiem Tra Argo CD Dang Chay

```bash
kubectl -n argocd get pods
```

Ky vong cac pod chinh dang `Running`:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-redis
```

Kiem tra service/ingress Argo CD:

```bash
kubectl -n argocd get svc
kubectl -n argocd get ingress
```

Neu mày expose Argo CD bang `/argocd`, test:

```bash
curl -I http://SERVER_IP/argocd
```

Neu UI mo duoc:

```text
http://SERVER_IP/argocd
```

## 4. Kiem Tra Traefik Ingress Cua K3s

K3s mac dinh co Traefik:

```bash
kubectl -n kube-system get pods | grep traefik
kubectl -n kube-system get svc | grep traefik
```

Test port 80:

```bash
curl -I http://SERVER_IP
```

Co the thay `404 Not Found`, cai do ok. Nghia la Traefik co phan hoi.

Neu timeout/refused:

```bash
sudo ufw status
```

Can mo port:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

## 5. Kiem Tra Image Da Co Tren GHCR

Truoc khi sync app, image phai ton tai.

Can co 2 image:

```text
ghcr.io/shynsg/todo-backend:v1.0
ghcr.io/shynsg/todo-frontend:v1.0
```

Neu mày dang dung tag khac, vi du `latest`, thi phai sua trong platform repo:

```text
todo-platform/apps/backend-stack/overlays/prod/kustomization.yaml
```

Dang mong muon:

```yaml
images:
  - name: ghcr.io/shynsg/todo-backend
    newTag: v1.0
  - name: ghcr.io/shynsg/todo-frontend
    newTag: v1.0
```

Neu GHCR package private, K8s se bi `ImagePullBackOff`. De hoc nhanh, nen set package public truoc.

## 5.1. Tao Version Image Bang Git Tag

Workflow backend/frontend da ho tro version tag dang:

```text
v1.0.0
v1.0.1
v1.1.0
```

Khi push len branch `main`, workflow se tao:

```text
ghcr.io/shynsg/todo-backend:latest
ghcr.io/shynsg/todo-backend:<short-sha>
ghcr.io/shynsg/todo-backend:v0.0.<github-run-number>
```

Khi push Git tag `v1.0.1`, workflow se tao them:

```text
ghcr.io/shynsg/todo-backend:v1.0.1
```

Neu muon auto version, chi can push `main`.

Vi du backend:

```bash
cd lesson-12-capstone-platform/source/todo-backend
git add .
git commit -m "update backend"
git push origin main
```

Sau khi workflow chay xong, vao GitHub Actions lay run number. Vi du run number la `42`, image tag se la:

```text
ghcr.io/shynsg/todo-backend:v0.0.42
```

Vi du frontend:

```bash
cd lesson-12-capstone-platform/source/todo-frontend
git add .
git commit -m "update frontend"
git push origin main
```

Neu frontend workflow run number la `27`, image tag se la:

```text
ghcr.io/shynsg/todo-frontend:v0.0.27
```

Sau khi GitHub Actions green, sua platform repo:

```yaml
images:
  - name: ghcr.io/shynsg/todo-backend
    newTag: v0.0.42
  - name: ghcr.io/shynsg/todo-frontend
    newTag: v0.0.27
```

Push platform repo:

```bash
cd lesson-12-capstone-platform/todo-platform
git add apps/backend-stack/overlays/prod/kustomization.yaml
git commit -m "release auto version images"
git push
```

Argo CD se sync va Kubernetes se rollout Pod moi.

Neu muon version dep kieu `v1.0.1`, van co the push Git tag thu cong:

```bash
git tag v1.0.1
git push origin v1.0.1
```

## 6. Kiem Tra Todo Platform Repo

Argo CD can doc repo GitOps/platform, vi du:

```text
https://github.com/shynsg/todo-platform.git
```

Repo nay phai co path:

```text
apps/backend-stack/overlays/prod
```

Trong path do phai co:

```text
kustomization.yaml
namespace.yaml
```

File namespace phai tao:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: apps-prod
```

Neu repo platform private, Argo CD phai duoc add credential GitHub truoc.

## 7. Cach A: Tao Application Bang Argo CD UI

Mo UI:

```text
http://SERVER_IP/argocd
```

Vao:

```text
Settings
-> Repositories
-> Connect Repo
```

Neu repo public:

```text
Connection method: HTTPS
Repository URL: https://github.com/shynsg/todo-platform.git
```

Neu repo private:

```text
Connection method: HTTPS
Repository URL: https://github.com/shynsg/todo-platform.git
Username: GitHub username
Password: GitHub token
```

Sau do tao app:

```text
Applications
-> New App
```

Dien:

```text
Application Name: backend-prod
Project: default
Sync Policy: Automatic
Prune Resources: checked
Self Heal: checked
Repository URL: https://github.com/shynsg/todo-platform.git
Revision: main
Path: apps/backend-stack/overlays/prod
Cluster URL: https://kubernetes.default.svc
Namespace: apps-prod
```

Neu co option:

```text
Auto-create namespace: checked
```

Panel `KUSTOMIZE` co the xuat hien ben duoi. Day la cho Argo CD override them len tren file `kustomization.yaml`.

Trong bai nay co 2 cach dung.

Cach nen dung: de Git quan ly image tag trong file:

```text
apps/backend-stack/overlays/prod/kustomization.yaml
```

Neu file tren Git da dung image/tag roi, trong panel `KUSTOMIZE` de nhu sau:

```text
Version: default
Name Prefix: de trong
Name Suffix: de trong
Namespace: de trong
Images: khong can sua
Digest?: khong tick
```

Khi do Argo CD se doc dung `kustomization.yaml` trong repo.

Neu panel dang hien `ghcr.io/YOUR_ORG/...` va tag `CHANGE_ME`, co nghia la Git repo/branch/path ma Argo CD dang doc van con placeholder. Co 2 cach fix:

```text
Cach 1: sua file kustomization.yaml trong todo-platform repo, commit va push.
Cach 2: override tam thoi ngay tren panel KUSTOMIZE.
```

Neu override tam thoi tren UI, dien:

```text
Image: ghcr.io/YOUR_ORG/todo-backend
New Image: ghcr.io/shynsg/todo-backend
New Tag: v1.0
Digest?: khong tick
```

```text
Image: ghcr.io/YOUR_ORG/todo-frontend
New Image: ghcr.io/shynsg/todo-frontend
New Tag: v1.0
Digest?: khong tick
```

Voi `postgres` va `redis`, de nguyen:

```text
postgres -> postgres:16-alpine
redis    -> redis:7-alpine
```

Khong tick `Digest?` tru khi mày deploy bang image digest dang `sha256:...`.

Sau do:

```text
Create
-> Sync
```

Sau khi sync xong, quay lai VPS kiem tra:

```bash
kubectl get ns
```

Bay gio moi phai thay:

```text
apps-prod
```

## 8. Cach B: Tao Application Bang Kubectl Tren VPS

Neu khong muon tao bang UI, co the tao Argo CD Application truc tiep tren VPS.

Chay lenh nay tren VPS, nho sua repo URL neu repo mày khac:

```bash
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shynsg/todo-platform.git
    targetRevision: main
    path: apps/backend-stack/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: apps-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

Kiem tra Application:

```bash
kubectl -n argocd get applications
kubectl -n argocd get application backend-prod
```

Neu Application da tao nhung chua sync ngay, co the vao UI bam `Sync`.

Neu co Argo CD CLI thi dung duoc:

```bash
argocd app sync backend-prod
```

Nhung dang hoc thi UI de nhin loi ro hon.

## 9. Neu Vẫn Khong Thay apps-prod

Chay:

```bash
kubectl -n argocd get applications
```

Neu khong thay `backend-prod`:

```text
Chua tao Argo CD Application.
Quay lai buoc 7 hoac buoc 8.
```

Neu thay `backend-prod`, xem chi tiet:

```bash
kubectl -n argocd describe application backend-prod
```

Can de y cac loi:

```text
repoURL sai
repo private nhung chua add credential
path sai
kustomization.yaml loi
image tag khong ton tai
permission khong tao duoc namespace
```

Kiem tra log Argo CD controller:

```bash
kubectl -n argocd logs deploy/argocd-application-controller
```

Kiem tra repo server:

```bash
kubectl -n argocd logs deploy/argocd-repo-server
```

## 10. Kiem Tra Sau Khi apps-prod Xuat Hien

```bash
kubectl -n apps-prod get all
```

Ky vong:

```text
deployment.apps/backend
deployment.apps/frontend
statefulset.apps/postgres
statefulset.apps/redis
job.batch/backend-migration
service/backend
service/frontend
service/db
service/redis
```

Kiem tra Pod:

```bash
kubectl -n apps-prod get pods -o wide
```

Ky vong:

```text
backend-xxx             1/1 Running
frontend-xxx            1/1 Running
postgres-0              1/1 Running
redis-0                 1/1 Running
backend-migration-xxx   0/1 Completed
```

## 11. Kiem Tra Migration

```bash
kubectl -n apps-prod get jobs
```

Ky vong:

```text
backend-migration   Complete
```

Lay log migration:

```bash
POD=$(kubectl -n apps-prod get pods -l job-name=backend-migration -o jsonpath='{.items[0].metadata.name}')
kubectl -n apps-prod logs "$POD"
```

Neu Job fail:

```bash
kubectl -n apps-prod describe job backend-migration
kubectl -n apps-prod describe pod "$POD"
```

## 12. Kiem Tra Service Noi Bo

```bash
kubectl -n apps-prod get svc
```

Can thay:

```text
backend    ClusterIP   3000/TCP
frontend   ClusterIP   8080/TCP
db         ClusterIP   5432/TCP
redis      ClusterIP   6379/TCP
```

Test backend noi bo bang temporary pod:

```bash
kubectl -n apps-prod run curl-test --rm -it --image=curlimages/curl -- sh
```

Trong shell cua pod:

```bash
curl http://backend:3000/health
curl http://backend:3000/api/todos
exit
```

## 13. Kiem Tra Ingress App Bang IP

```bash
kubectl -n apps-prod get ingress
kubectl -n apps-prod describe ingress todo
```

Vì chua co domain, Ingress khong nen co host.

Kiem tra YAML:

```bash
kubectl -n apps-prod get ingress todo -o yaml
```

Dung case IP thi nen thay:

```yaml
rules:
  - http:
      paths:
```

Khong nen thay:

```yaml
host: todo.YOUR_DOMAIN.com
```

## 14. Test Tu VPS

```bash
curl http://127.0.0.1/api/health
curl http://SERVER_IP/api/health
curl http://SERVER_IP/api/todos
```

Ky vong health:

```json
{"ok":true}
```

Neu `127.0.0.1` ok nhung `SERVER_IP` fail, kha nang la firewall/security group cua VPS.

## 15. Test Tu May Ca Nhan

Mo browser:

```text
http://SERVER_IP/
```

Test API:

```bash
curl http://SERVER_IP/api/health
```

## 16. Debug Theo Loi Hay Gap

### Khong Co apps-prod

```bash
kubectl -n argocd get applications
```

Neu khong co `backend-prod`:

```text
Chua tao Application.
```

Neu co `backend-prod`:

```bash
kubectl -n argocd describe application backend-prod
```

### ImagePullBackOff

```bash
kubectl -n apps-prod get pods
kubectl -n apps-prod describe pod POD_NAME
```

Hay gap:

```text
GHCR image private
image name sai
tag sai
image chua duoc build/push
```

### Pod Backend CrashLoopBackOff

```bash
kubectl -n apps-prod logs deploy/backend
kubectl -n apps-prod describe deploy backend
```

Hay gap:

```text
DATABASE_URL sai
Postgres chua Ready
Migration loi
```

### API Qua IP Khong Duoc

```bash
kubectl -n apps-prod get ingress
kubectl -n apps-prod describe ingress todo
kubectl -n kube-system get pods | grep traefik
curl -I http://127.0.0.1
curl -I http://SERVER_IP
```

Hay gap:

```text
port 80 chua mo
Ingress chua sync
Traefik chua chay
path /api bi sai
```

## 17. Checklist Tren VPS

```text
[ ] SSH vao VPS duoc
[ ] kubectl get nodes thay Ready
[ ] argocd namespace ton tai
[ ] argocd pods Running
[ ] Argo CD UI vao duoc qua http://SERVER_IP/argocd
[ ] Traefik Running trong kube-system
[ ] port 80 co response
[ ] GHCR image backend ton tai
[ ] GHCR image frontend ton tai
[ ] todo-platform repo da push len GitHub
[ ] Argo CD connect duoc todo-platform repo
[ ] Argo CD Application backend-prod da tao
[ ] Application source repoURL dung
[ ] Application path dung apps/backend-stack/overlays/prod
[ ] Application sync thanh cong
[ ] namespace apps-prod xuat hien
[ ] backend Deployment Running
[ ] frontend Deployment Running
[ ] postgres StatefulSet Running
[ ] redis StatefulSet Running
[ ] backend-migration Job Complete
[ ] ingress todo ton tai
[ ] ingress todo khong co host/domain
[ ] curl http://SERVER_IP/api/health ok
[ ] browser http://SERVER_IP/ vao duoc frontend
```
