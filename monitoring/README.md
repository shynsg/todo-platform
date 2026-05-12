# Monitoring For 6GB Server

Start light.

## Phase 1: Metrics Server

Install metrics-server:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Check:

```bash
kubectl top nodes
kubectl top pods -A
```

## Phase 2: Logs

Bat dau bang:

```bash
kubectl logs
journalctl -u k3s
```

Sau do co the them Loki/Grafana neu server con du RAM.

## Phase 3: Prometheus/Grafana

Dung Helm chart `kube-prometheus-stack`, nhung can tune resource.

Tren server 6GB, chi cai khi da on dinh app chinh.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Sau do tao values nhe rieng truoc khi install.
