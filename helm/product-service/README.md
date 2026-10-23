# product-service

Helm chart for the Product Service, the first service extracted from the OrderManager monolith (see [`docs/decomposition-plan.md`](../../docs/decomposition-plan.md)).

## Resources

| Template | Kind | Notes |
|----------|------|-------|
| `deployment.yaml` | Deployment | Startup/liveness/readiness probes on `/health`, resource requests/limits, `RollingUpdate` (`maxUnavailable: 0`), non-root read-only container, zone/host spread, rolls on ConfigMap change |
| `service.yaml` | Service | `ClusterIP`, port 80 → container 8080 |
| `hpa.yaml` | HorizontalPodAutoscaler | 2–10 replicas, 70% CPU target |
| `configmap.yaml` | ConfigMap | `config:` map, loaded with `envFrom` |
| `serviceaccount.yaml` | ServiceAccount | IRSA via `serviceAccount.irsaRoleArn` → `eks.amazonaws.com/role-arn` |
| `networkpolicy.yaml` | NetworkPolicy | Ingress only from `allowedClients` pods, `monitoring`, and (optionally) `ingress-nginx`; egress to DNS, HTTPS and `egress.extraRules` |
| `pdb.yaml` | PodDisruptionBudget | `minAvailable: 1` |
| `servicemonitor.yaml` | ServiceMonitor | Prometheus scrape of `/metrics` |
| `ingress.yaml` | Ingress | Disabled by default (traffic goes through `api-gateway`) |

## Usage

```bash
# default values
helm upgrade --install product-service helm/product-service -n decomposition-dev

# production
helm upgrade --install product-service helm/product-service -n decomposition-prod \
  -f helm/product-service/values.yaml -f helm/product-service/values-production.yaml
```

ArgoCD: [`argocd/product-service-production.yaml`](../../argocd/product-service-production.yaml).

## Production prerequisites

- IAM role `workshop-prod-product-service` with a trust policy for the EKS OIDC provider and subject `system:serviceaccount:decomposition-prod:product-service`.
- Secret `product-service-db` in `decomposition-prod` holding `ConnectionStrings__DefaultConnection`.
- The default `decomposition-prod` ResourceQuota (`requests.cpu: 2`, `limits.cpu: 4`) caps this service below 10 replicas at production sizing (250m/1 CPU per pod); raise it in `platform-engineering-shared-services` (`cdk/bin/cdk.ts`) before relying on `maxReplicas: 10`.
