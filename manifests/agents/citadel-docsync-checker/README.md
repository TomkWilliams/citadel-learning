# citadel-docsync-checker

Skeleton manifests for the first Citadel-hosted agent.

This directory is intentionally non-running until the image, context bundle,
outbox storage, and model endpoint are chosen and reviewed.

Design source:

```text
/home/tomkw/src/Operations/agent-compat/agent-designs/citadel-docsync-checker.md
```

Security baseline:

```text
/home/tomkw/src/Operations/agent-compat/container-security-baseline.md
```

## Intended Shape

| Field | Value |
| --- | --- |
| Namespace | `citadel-learning` |
| Workload | Manual `Job` |
| Agent level | Level 0 reader/report writer |
| Model | `qwen3:8b` by default |
| Context | `/read/context`, read-only |
| Output | `/outbox/reports/docsync/` |
| Kubernetes API | No access for first version |
| Secrets | None |
| Network | Default deny; allow only approved model endpoint if needed |

## Files

| File | Purpose |
| --- | --- |
| `kustomization.yaml` | Review-only kustomization; resources are not enabled yet |
| `serviceaccount.yaml` | Dedicated workload identity with token automount disabled |
| `configmap-context-profile.yaml` | Source list for the `multi-agent-core` context profile |
| `job.yaml` | Hardened manual Job skeleton |
| `networkpolicy.yaml` | Default-deny ingress/egress skeleton |

## Apply Mode

`kustomization.yaml` has `resources: []`, so `kubectl apply -k .` deploys nothing.
That protection only holds for `-k`. Do not run `kubectl apply -f` or
`kubectl apply -R -f` against the files in this directory directly — that
bypasses the empty resource list and would create the ServiceAccount, ConfigMap,
and NetworkPolicy for real (the Job would fail on the placeholder image, but the
rest would apply). Only use `-k` until the checklist below is complete and the
resources are uncommented.

## Activation Checklist

- [ ] Build or select the agent image.
- [ ] Pin the image tag or digest.
- [ ] Decide how `/read/context` is populated.
- [ ] Decide how `/outbox` is stored and reviewed.
- [ ] Decide model endpoint service/DNS name.
- [ ] Verify the cluster CNI enforces Kubernetes NetworkPolicy.
- [ ] Enable only the required NetworkPolicy egress.
- [ ] Replace the placeholder image before enabling `job.yaml`.
- [ ] Run the Operations evaluation harness with safe sample context.
- [ ] Run Claude continuity check.
- [ ] Uncomment resources in `kustomization.yaml`.

## Kill Switch

For a manual Job:

```bash
kubectl -n citadel-learning delete job citadel-docsync-checker
```

For GitOps rollback, remove or comment the resources in `kustomization.yaml`
and let reconciliation prune them.
