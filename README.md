# Fluent-bit & Loki — GitOps Manifests

Kubernetes manifests for deploying [Fluent-bit](https://fluentbit.io/) (log collector) and [Grafana Loki](https://grafana.com/oss/loki/) (log aggregation) via [ArgoCD](https://argo-cd.readthedocs.io/).

Fluent-bit runs as a DaemonSet on every node, collects container and systemd logs, and forwards them to Loki. Loki stores and indexes the logs for querying in Grafana.

![Fluent-bit](https://fluentbit.io/images/hero.svg)

![Grafana Loki Architecture](https://www.atatus.com/blog/content/images/size/w1000/2022/02/grafana-loki-work.png)


---

## Project Structure

```
.
├── argocd/
│   ├── fluentbit-app.yaml       # ArgoCD Application for Fluent-bit
│   └── loki-app.yaml            # ArgoCD Application for Loki
│
├── fluentbit/
│   ├── base/                    # Base Fluent-bit manifests
│   │   ├── clusterrole.yaml
│   │   ├── clusterrolebinding.yaml
│   │   ├── configmap.yaml       # Fluent-bit pipeline config
│   │   ├── daemonset.yaml
│   │   ├── service.yaml
│   │   ├── serviceaccount.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       └── prod/
│           └── kustomization.yaml   # Prod: resource limits, labels
│
└── loki/
    ├── base/                    # Base Loki manifests
    │   ├── configmap.yaml       # Loki server config
    │   ├── deployment.yaml
    │   ├── pvc.yaml             # 10Gi NFS-backed storage
    │   ├── service.yaml
    │   └── kustomization.yaml
    └── overlays/
        └── prod/
            └── kustomization.yaml   # Prod: 50Gi PVC, resource limits, labels
```

The `base/` directories contain environment-agnostic manifests. The `overlays/prod/` directories patch prod-specific values (resource limits, storage size) and apply common labels (`environment=production`, `app.kubernetes.io/instance=prod`).

---

## Deploying via ArgoCD

Update the `repoURL` in both `argocd/` manifests to point to this repository, then apply them once:

```bash
kubectl apply -f argocd/fluentbit-app.yaml
kubectl apply -f argocd/loki-app.yaml
```

ArgoCD will sync and reconcile both applications automatically from that point on. To trigger a manual sync:

```bash
argocd app sync fluent-bit
argocd app sync loki
```

### Applying the prod overlay manually

```bash
kubectl apply -k fluentbit/overlays/prod
kubectl apply -k loki/overlays/prod
```

---

## Validating the Deployment

### Fluent-bit

Check all DaemonSet pods are running (one per node):

```bash
kubectl get daemonset fluent-bit -n default
kubectl get pods -n default -l app.kubernetes.io/name=fluent-bit
```

Tail logs to confirm it is shipping to Loki without errors:

```bash
kubectl logs -n default -l app.kubernetes.io/name=fluent-bit --tail=50
```

Hit the built-in health endpoint on any pod:

```bash
kubectl exec -n default <fluent-bit-pod> -- curl -s http://localhost:2020/api/v1/health
# Expected: {"healthy":true}
```

### Loki

Check the Loki pod is ready:

```bash
kubectl get deployment loki -n monitoring
kubectl get pods -n monitoring -l app=loki
```

Query the ready endpoint directly:

```bash
kubectl exec -n monitoring <loki-pod> -- wget -qO- http://localhost:3100/ready
# Expected: ready
```

Verify Loki is receiving log streams (returns a list of active label values):

```bash
kubectl exec -n monitoring <loki-pod> -- \
  wget -qO- "http://localhost:3100/loki/api/v1/labels"
```

### Grafana

![Loki Drill Down](https://grafana.com/media/docs/loki/get-started-drill-down-container.png?w=1040)


1. **Add Loki as a data source** — navigate to *Connections → Data sources → Add data source → Loki* and set the URL to `http://loki.monitoring.svc.cluster.local:3100`. Click *Save & test* — a green banner confirms connectivity.

2. **Explore logs** — open *Explore*, select the Loki data source, and run a basic query to confirm logs are flowing:
   ```logql
   {job="fluentbit"}
   ```
   You should see a live stream of log lines from across the cluster.
