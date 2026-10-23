# app_dotnet-angular-monolith-iac
Service-specific IaC for the OrderManager monolith. Helm charts, Dockerfile, ArgoCD manifests, CI/CD pipeline conforming to platform-engineering-shared-services standard.

## GitOps deployment (Argo CD)

`argocd/application-<env>.yaml` registers one Argo CD `Application` per environment against the platform Argo CD (`argocd` namespace, `default` project). Each one renders `helm/ordermanager` from this repo (`main`) with `values.yaml` + `values-<env>.yaml` and deploys it to `decomposition-<env>`.

```bash
kubectl apply -f argocd/application-dev.yaml
kubectl -n argocd get applications
```

**Sync policy.** Automated sync with `prune` and `selfHeal`, so Git is the only source of truth: manual edits in the cluster are reverted and resources removed from the chart are deleted (`PruneLast`, foreground propagation). Failed syncs retry 5 times with exponential backoff (10s to 3m). The `resources-finalizer` makes deleting an Application delete its resources.

**Sync waves.** Each wave must be applied and Healthy before the next one starts:

| Wave | Resources |
|------|-----------|
| 0 | `Namespace` (`namespace.create: true`, set by the Application; never pruned or deleted, so platform quotas and policies stay) |
| 1 | `ConfigMap` (app settings from `config:`, loaded with `envFrom`), `NetworkPolicy` |
| 2 | `Deployment`, `HorizontalPodAutoscaler`, `PodDisruptionBudget` (`minAvailable: 1`) |
| 3 | `Service`, `ServiceMonitor` |
| 4 | `Ingress` |
| PostSync | smoke-test `Job` |

**Health checks.** Argo CD marks the sync successful only when:

- the Deployment is Healthy: the new ReplicaSet is fully rolled out and its pods pass the readiness probe (`GET /api/products`, which reads the database). A startup probe covers first-start seeding, and `/health` liveness restarts a hung process. A rollout that doesn't finish within `deployment.progressDeadlineSeconds` (300s) turns the Deployment, and the Application, Degraded;
- the Ingress has a load balancer address from ingress-nginx;
- the PostSync `smoke-test` Job (`smokeTest.*`) can `curl` `/health` and `/api/products` through the Service. If the Job fails, the sync fails.

For a plain `helm install` (no Argo CD), leave `namespace.create` off and use `--create-namespace`; the smoke test then runs as a Helm post-install/upgrade hook.
