# Zero-downtime deployment with Argo Rollouts

Ye project ek Kubernetes app ko zero-downtime tareeqe se update karne ka hands-on proof hai. Teen strategies test ki gayi hain: rolling, blue-green aur canary. Har test mein load generator ne lagatar requests bheji aur failures count kiye.

Ye sab local **minikube** cluster par chala hai. EKS par same flow ka verification baad mein hoga.

## Problem

Deployment ke dauran agar purane pods ek saath band hon, ya naya version bina check ke sab users ko jaye, to users ko errors milte hain. Zero downtime ka matlab hai ke update ke dauran koi request fail na ho, aur kharab version ka asar limited rahe.

## Architecture

```
Load generator (cluster ke andar, busybox)
   | GET /api/version (lagatar)
   v
Service (stable address)
   |
   v
Rollout (Argo Rollouts)   strategy: canary | blueGreen
   |
   v
Pods (hello-devops-web, readiness probe: /api/health)
```

- **Rollout** pods banata aur update karta hai (kaun sa image, kitne pods, kaise update).
- **Service** ek stable address deta hai. Pods ko label (`app: hello-web`) se pakadta hai.
- **Readiness probe** tabhi traffic deta hai jab pod ready ho.
- **preStop sleep** pod band hone se pehle current requests khatam hone deta hai.

Rollout aur Service alag files mein hain, kyunki pods (kitchen) aur traffic (gate) alag kaam hain.

## Strategies

| Strategy | Kaise | Result file |
|---|---|---|
| Rolling (canary block, maxSurge 1, maxUnavailable 0) | Pods ek ek karke replace | `results/rolling-v25-to-v28.txt` |
| Blue-green | Active (v28) aur preview (v25) alag services, manual promote | `results/bluegreen-promote-abort.txt` |
| Canary (33, pause 30s, 66, pause 30s, 100) | Naye version ko dheere traffic | `results/canary-clean-v25-to-v28.txt` |

## Results

| Test | Load | Failed requests | Note |
|---|---|---|---|
| Rolling v25 to v28 | ~5 req/s, cluster ke andar | 0 (1450 requests) | Basic verification |
| Blue-green | Manual promote aur abort | Active par koi error observed nahi | Preview aur active ek saath verify hue |
| Canary clean run (06:20:11 to 06:21:22 UTC) | ~5 req/s | 0 (4600 requests tak) | Canary window poora loadgen ke andar |

### Honest notes

- Load halka tha (ek client, ~5 req/s). Zyada load ka test baad mein plan hai.
- Ek pehle canary run mein FAIL aaye the, jo cluster restart (sandbox change aur readiness failures) ki wajah se the, canary ki wajah se nahi. Us run ko proof nahi maana, aur clean run alag se save kiya.
- Blue-green mein `undo` ne ek naya preview banaya (rollback bhi ek naya deployment hai). Abort se active par koi asar nahi pada.

## Repo structure

```
k8s/
  rollout.yaml           Rollout (current strategy aur image)
  service.yaml           Service (rolling aur canary ke liye)
  service-active.yaml    Blue-green: active service
  service-preview.yaml   Blue-green: preview service
results/
  rolling-v25-to-v28.txt
  bluegreen-promote-abort.txt
  canary-clean-v25-to-v28.txt
```

## Steps to reproduce

Prerequisites: minikube, kubectl, Argo Rollouts controller aur kubectl plugin.

```bash
# 1. Argo Rollouts install (server-side apply, kyunki CRD annotation badi hai)
kubectl create namespace argo-rollouts
kubectl apply --server-side -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# 2. Namespace aur services
kubectl create namespace zero-downtime
kubectl apply -f k8s/service.yaml -f k8s/rollout.yaml

# 3. Load generator (alag terminal mein chhod do)
kubectl run loadgen -n zero-downtime --image=busybox:1.36 --restart=Never -- sh -c 'ok=0; fail=0; while true; do if wget -q -T 2 -O- http://hello-web/api/version >/dev/null 2>&1; then ok=$((ok+1)); else fail=$((fail+1)); echo "$(date -u +%T) FAIL fail=$fail"; fi; if [ $((ok % 50)) -eq 0 ]; then echo "$(date -u +%T) ok=$ok fail=$fail"; fi; sleep 0.2; done'

# 4. Update (image tag badlo k8s/rollout.yaml mein), phir watch
kubectl apply -f k8s/rollout.yaml
kubectl argo rollouts get rollout hello-web -n zero-downtime --watch

# 5. Result
kubectl logs loadgen -n zero-downtime | grep -c FAIL
```

## Learnings

- Rollout ka canary mein `maxSurge` aur `maxUnavailable` hote hain. `rollingUpdate` block is version mein nahi hota.
- Blue-green mein `undo` seedha switch nahi karta. Pehle ek naya preview banta hai, phir promote se switch hota hai.
- Zero downtime ke liye readiness probe aur `maxUnavailable: 0` dono zaroori hain.
- Container non-root ho to pod-level `securityContext` mein numeric `runAsUser` chahiye (kubelet naam se verify nahi karta).
- Events (`kubectl get events`) aur loadgen logs ke timestamps match karna hi clean proof banata hai.

## Next

- EKS par same test, ALB ke saath (Ingress ke zariye traffic).
- Zyada load ke saath test (kai clients).
- Argo Rollouts analysis (Prometheus error rate) se automatic rollback.
