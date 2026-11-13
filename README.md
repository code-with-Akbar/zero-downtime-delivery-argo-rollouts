# Zero-Downtime Deployment with Argo Rollouts

This project is a hands-on demonstration of updating a Kubernetes application without downtime. Three deployment strategies were tested: rolling, blue-green, and canary. During each test, a load generator sent continuous requests to the application and counted any failures.

All tests ran on a local **minikube** cluster. The same flow on Amazon EKS, with an Application Load Balancer, is planned as the next step.

## Problem

If old pods are terminated before new ones are ready, users see errors during a deployment. If a new version is sent to all users without any checks, a bad release affects everyone at once.

Zero downtime means that no request fails during an update, and a faulty version has limited impact.

## Architecture

```
Load generator (busybox pod inside the cluster)
   | GET /api/version, continuously
   v
Service (stable address)
   |
   v
Rollout (Argo Rollouts)        strategy: canary | blueGreen
   |
   v
Pods (hello-devops-web, readiness probe on /api/health)
```

- **Rollout**: creates and updates the pods. It defines the image, the replica count, and how the update happens.
- **Service**: provides a stable address. It selects pods by the label `app: hello-web`.
- **Readiness probe**: a pod receives traffic only after it reports ready.
- **preStop hook**: waits before a pod shuts down, so in-flight requests can finish.

The Rollout and the Service are separate objects. The pods do the work, and the Service decides where traffic goes.

## Strategies

| Strategy | How it works | Result file |
|---|---|---|
| Rolling (canary block with maxSurge 1, maxUnavailable 0) | Pods are replaced one at a time | `results/rolling-v25-to-v28.txt` |
| Blue-green | Active service (v28) and preview service (v25) run side by side. Promotion is manual | `results/bluegreen-promote-abort.txt` |
| Canary (33%, pause 30s, 66%, pause 30s, 100%) | The new version receives traffic gradually | `results/canary-clean-v25-to-v28.txt` |

## Results

| Test | Load | Failed requests | Notes |
|---|---|---|---|
| Rolling, v25 to v28 | About 5 requests/s from inside the cluster | 0 out of 1450 | Basic verification |
| Blue-green | Manual promote and abort | No errors observed on the active service | Preview and active verified side by side |
| Canary, clean run (06:20:11 to 06:21:22 UTC) | About 5 requests/s | 0 out of 4600 | Load ran for the full canary window |

### Honest notes

- The load was light: one client at about 5 requests/s. A heavier load test with multiple clients is planned.
- An earlier canary run showed a few failures. Those came from a cluster restart (pod sandbox changes and readiness failures), not from the canary itself. That run is not used as proof. The clean run is saved separately.
- In the blue-green test, `undo` created a new preview. A rollback is itself a new deployment, so it does not switch traffic instantly. Aborting the rollback left the active service unchanged.

## Repository structure

```
k8s/
  rollout.yaml           Rollout (current strategy and image)
  service.yaml           Service (used for rolling and canary)
  service-active.yaml    Blue-green: active service
  service-preview.yaml   Blue-green: preview service
results/
  rolling-v25-to-v28.txt
  bluegreen-promote-abort.txt
  canary-clean-v25-to-v28.txt
```

## Steps to reproduce

Prerequisites: minikube, kubectl, the Argo Rollouts controller, and the kubectl plugin.

```bash
# 1. Install Argo Rollouts (server-side apply, because the CRD annotations are large)
kubectl create namespace argo-rollouts
kubectl apply --server-side -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 2. Namespace and services
kubectl create namespace zero-downtime
kubectl apply -f k8s/service.yaml -f k8s/rollout.yaml

# 3. Load generator (run it in a separate terminal)
kubectl run loadgen -n zero-downtime --image=busybox:1.36 --restart=Never -- sh -c 'ok=0; fail=0; while true; do if wget -q -T 2 -O- http://hello-web/api/version >/dev/null 2>&1; then ok=$((ok+1)); else fail=$((fail+1)); echo "$(date -u +%T) FAIL fail=$fail"; fi; if [ $((ok % 50)) -eq 0 ]; then echo "$(date -u +%T) ok=$ok fail=$fail"; fi; sleep 0.2; done'

# 4. Trigger an update (change the image tag in k8s/rollout.yaml), then watch
kubectl apply -f k8s/rollout.yaml
kubectl argo rollouts get rollout hello-web -n zero-downtime --watch

# 5. Check the result
kubectl logs loadgen -n zero-downtime | grep -c FAIL
```

## Learnings

- In this Argo Rollouts version, the canary strategy takes `maxSurge` and `maxUnavailable`. There is no `rollingUpdate` block under `strategy`.
- In blue-green, `undo` does not switch traffic directly. It first creates a new preview, and a promote is needed to switch.
- For zero downtime, both a readiness probe and `maxUnavailable: 0` are required.
- For a non-root container, the pod-level `securityContext` needs a numeric `runAsUser`. The kubelet cannot verify a user by name.
- Clean proof comes from matching Kubernetes event timestamps with the load generator logs.

## Next steps

- Run the same test on EKS, with traffic routed through an ALB.
- Test with higher load, using multiple clients.
- Add Argo Rollouts analysis based on Prometheus error rates, for automatic rollback.
