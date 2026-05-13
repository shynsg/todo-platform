# Monitoring For 6GB Server

Capstone platform dung monitoring nhe truoc.

Thu tu hoc:

```text
Lesson 13: metrics-server + kubectl top + debug workflow
Lesson 14: Datadog Agent bang GitOps
Lesson 15: Datadog logs, APM, dashboard, alert
```

Metrics-server GitOps manifest nam tai:

```text
./metrics-server
../argocd/metrics-server-application.yaml
../docs/metrics-server-gitops-runbook.md
```

Doc lesson 13 tai:

```text
../../../lesson-13-monitoring-basic/README.md
../../../lesson-13-monitoring-basic/COMMANDS.md
```

Khong nen self-host kube-prometheus-stack full tren VPS 6GB cho bai nay. Production team nho thuong dung Datadog/New Relic/Grafana Cloud de giam cong van hanh.
