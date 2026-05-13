# Datadog Operator GitOps Runbook

Muc tieu:

```text
Datadog Operator va Datadog Agent duoc quan ly boi Argo CD
config nam trong todo-platform/monitoring
API key khong commit vao Git
metrics/logs/events/APM gui ve Datadog SaaS
```

Docs chinh thuc:

```text
https://docs.datadoghq.com/containers/kubernetes/installation/?tab=datadogoperator
https://docs.datadoghq.com/containers/kubernetes/operator_configuration/
```

## 1. Flow Dung

Datadog Operator gom 2 lop:

```text
datadog-operator
-> Kubernetes controller
-> cai CRD DatadogAgent
-> doc DatadogAgent config
-> tu tao Agent DaemonSet / Cluster Agent / Cluster Checks Runner

DatadogAgent
-> custom resource
-> file config that su cua Agent
-> site, clusterName, api secret, logs, APM, otel...
```

Vi vay GitOps se co 2 Argo CD Application:

```text
datadog-operator
-> source: Helm repo https://helm.datadoghq.com
-> chart: datadog-operator

datadog-agent
-> path: monitoring/datadog-agent
```

Tao `datadog-operator` truoc, sau do moi tao `datadog-agent`.

## 2. Cau Truc File

```text
todo-platform/
├── monitoring/
│   └── datadog-agent/
│       ├── kustomization.yaml
│       └── datadog-agent.yaml
├── argocd/
│   ├── datadog-operator-application.yaml
│   └── datadog-agent-application.yaml
└── scripts/
    └── remove-datadog.sh
```

`argocd/datadog-operator-application.yaml` tro thang toi Helm repo chinh thuc cua Datadog.

Repo minh khong can luu file `.tgz` cua chart.

`monitoring/datadog-agent/datadog-agent.yaml` la file quan trong nhat:

```yaml
apiVersion: datadoghq.com/v2alpha1
kind: DatadogAgent
```

## 3. Tao Secret Mot Lan Tren VPS

API key khong nam trong Git.

Tren VPS:

```bash
kubectl create namespace datadog --dry-run=client -o yaml | kubectl apply -f -
```

Tao secret:

```bash
kubectl -n datadog create secret generic datadog-secret \
  --from-literal api-key='YOUR_DATADOG_API_KEY' \
  --dry-run=client -o yaml | kubectl apply -f -
```

Kiem tra:

```bash
kubectl -n datadog get secret datadog-secret
```

## 4. Sua Site Neu Can

Mo file:

```text
monitoring/datadog-agent/datadog-agent.yaml
```

Dang de:

```yaml
site: us5.datadoghq.com
```

Neu account cua mày o site khac thi doi:

```text
datadoghq.com
datadoghq.eu
us3.datadoghq.com
us5.datadoghq.com
ap1.datadoghq.com
ddog-gov.com
```

Rule nhanh:

```text
app.datadoghq.com    -> datadoghq.com
app.us5.datadoghq.com -> us5.datadoghq.com
app.datadoghq.eu     -> datadoghq.eu
```

## 5. Commit Va Push Config

Tren local:

```bash
cd lesson-12-capstone-platform/todo-platform
```

Commit:

```bash
git add monitoring/datadog-agent \
  argocd/datadog-operator-application.yaml \
  argocd/datadog-agent-application.yaml \
  docs/datadog-gitops-runbook.md \
  scripts/remove-datadog.sh \
  monitoring/README.md

git commit -m "add datadog operator gitops"
git push
```

## 5.1. Luu Y Ve RUM Config

Trong `DatadogAgent`, doan `ddTraceConfigs` la config Datadog recommend neu mày bat APM auto instrumentation kem RUM:

```yaml
ddTraceConfigs:
  - name: "DD_RUM_ENABLED"
    value: "true"
  - name: "DD_RUM_APPLICATION_ID"
    value: "..."
  - name: "DD_RUM_CLIENT_TOKEN"
    value: "..."
  - name: "DD_RUM_SITE"
    value: "us5.datadoghq.com"
```

Khong nen copy y nguyen ID/token tu docs/demo. Neu mày muon dung RUM cho frontend, lay gia tri that tu Datadog UI:

```text
Digital Experience
-> Real User Monitoring
-> New Application
```

Sau do thay vao:

```text
DD_RUM_APPLICATION_ID
DD_RUM_CLIENT_TOKEN
DD_RUM_REMOTE_CONFIGURATION_ID
DD_RUM_SITE
```

Neu chua hoc RUM, co the tam thoi bo `ddTraceConfigs`, APM/log/metrics van chay.

## 6. Tao App Datadog Operator Trong Argo CD UI

Mo Argo CD:

```text
http://SERVER_IP/argocd
```

Tao app thu nhat:

```text
Applications
-> New App
```

Dien:

```text
Application Name: datadog-operator
Project: default
Sync Policy: Automatic
Prune Resources: checked
Self Heal: checked
Repository URL: https://helm.datadoghq.com
Chart: datadog-operator
Version: 2.22.2
Cluster URL: https://kubernetes.default.svc
Namespace: datadog
```

Neu UI hien Helm Values, dien:

```yaml
installCRDs: true
```

Sau do:

```text
Create
-> Sync
```

Cho Operator Ready:

```bash
kubectl -n datadog get pods
kubectl get crd | grep datadog
```

Can thay CRD:

```text
datadogagents.datadoghq.com
```

## 7. Tao App Datadog Agent Trong Argo CD UI

Chi lam buoc nay sau khi `datadog-operator` da Synced/Healthy.

Tao app thu hai:

```text
Application Name: datadog-agent
Project: default
Sync Policy: Automatic
Prune Resources: checked
Self Heal: checked
Repository URL: https://github.com/shynsg/todo-platform.git
Revision: main
Path: monitoring/datadog-agent
Cluster URL: https://kubernetes.default.svc
Namespace: datadog
```

Sau do:

```text
Create
-> Sync
```

## 8. Tao Bang Kubectl Neu Khong Dung UI

Neu muon tao app bang CLI tren VPS:

```bash
kubectl apply -f argocd/datadog-operator-application.yaml
```

Doi operator ready:

```bash
kubectl -n argocd get application datadog-operator
kubectl -n datadog get pods
kubectl get crd | grep datadogagents
```

Roi apply agent app:

```bash
kubectl apply -f argocd/datadog-agent-application.yaml
```

## 9. Kiem Tra Tren VPS

Kiem tra Argo:

```bash
kubectl -n argocd get application datadog-operator
kubectl -n argocd get application datadog-agent
```

Kiem tra custom resource:

```bash
kubectl -n datadog get datadogagent
kubectl -n datadog describe datadogagent datadog
```

Kiem tra pods:

```bash
kubectl -n datadog get pods
kubectl -n datadog get daemonset
kubectl -n datadog get deploy
```

Ky vong:

```text
Datadog Operator pod Running
Datadog Agent DaemonSet co pod Running
Datadog Cluster Agent Deployment Running
```

Logs:

```bash
kubectl -n datadog logs deploy/datadog-operator --tail=100
kubectl -n datadog logs daemonset/datadog-agent --tail=100
kubectl -n datadog logs deploy/datadog-cluster-agent --tail=100
```

Neu ten resource khac, xem ten that:

```bash
kubectl -n datadog get pods,deploy,ds
```

## 10. Kiem Tra Tren Datadog UI

Trong Datadog UI, vao:

```text
Infrastructure
-> Kubernetes
```

Hoac:

```text
Infrastructure
-> Containers
```

Tim:

```text
k3s-vps-learning
apps-prod
backend
frontend
postgres
redis
```

Logs:

```text
Logs
-> Search
```

Thu query:

```text
kube_namespace:apps-prod
```

## 11. Debug Loi Hay Gap

### CRD Chua Co

Neu `datadog-agent` sync fail vi khong biet kind `DatadogAgent`:

```bash
kubectl get crd | grep datadogagents
kubectl -n argocd get application datadog-operator
```

Fix:

```text
Sync datadog-operator truoc
doi CRD tao xong
roi sync datadog-agent
```

### Secret Thieu Hoac Sai

Neu pod Datadog loi API key:

```bash
kubectl -n datadog get secret datadog-secret
kubectl -n datadog describe datadogagent datadog
kubectl -n datadog get pods
```

Secret phai co key:

```text
api-key
```

### Sai Datadog Site

Neu Agent chay nhung Datadog UI khong thay data, check:

```yaml
site: us5.datadoghq.com
```

Site phai khop account Datadog.

## 12. Remove Datadog

Xoa bang Argo CD UI:

```text
Applications
-> datadog-agent
-> Delete

Applications
-> datadog-operator
-> Delete
```

Hoac script:

```bash
cd todo-platform
./scripts/remove-datadog.sh
```

Script se giu namespace `datadog` de khong xoa nham `datadog-secret`.

Neu muon xoa het:

```bash
kubectl delete namespace datadog
```

## 13. Checklist

```text
[ ] co Datadog API key
[ ] tao namespace datadog
[ ] tao secret datadog-secret voi key api-key
[ ] site trong datadog-agent.yaml dung voi account
[ ] commit va push config
[ ] tao Argo app datadog-operator
[ ] datadog-operator Synced/Healthy
[ ] CRD datadogagents.datadoghq.com ton tai
[ ] tao Argo app datadog-agent
[ ] datadog-agent Synced/Healthy
[ ] DatadogAgent datadog ton tai
[ ] Agent pod Running
[ ] Cluster Agent Running
[ ] Datadog UI thay Kubernetes cluster
[ ] Datadog UI thay apps-prod pods
[ ] Logs co data kube_namespace:apps-prod
```
