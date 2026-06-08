---
name: k8s-debug
description: Kubernetes troubleshooting playbook. Load when investigating cluster issues — pods crashing, services unreachable, deployments failing, resource pressure. Provides a systematic diagnostic order.
compatibility: opencode
metadata:
  scope: kubernetes
  use-case: debugging
---

# Kubernetes debugging playbook

## Always start with context

Before any diagnosis :
```bash
kubectl config current-context     # which cluster?
kubectl get nodes -o wide          # nodes healthy?
kubectl get ns                     # which namespaces?
```

If wrong context, fix that first. Many homelab incidents are "I was looking at the wrong cluster".

## Systematic diagnostic order

Always go from the outside in :

### 1. Workload state (Deployment/StatefulSet/DaemonSet)

```bash
kubectl get deploy -n <ns>
kubectl describe deploy <name> -n <ns>
kubectl rollout status deploy/<name> -n <ns>
```

Check : `DESIRED` vs `AVAILABLE` replicas. If mismatch, drill down.

### 2. Pod state

```bash
kubectl get pods -n <ns> -o wide
kubectl describe pod <pod> -n <ns>
```

Read the `Events` section at the bottom of describe — it's where the real info is.

Status meanings :
- `Pending` → scheduling failure (no node fits, resource quota, taints)
- `ContainerCreating` → image pull or volume mount issue
- `CrashLoopBackOff` → container starts then dies repeatedly
- `Error` / `OOMKilled` → check exit code (137 = OOM, 1/2 = app error)
- `ImagePullBackOff` → wrong image name, missing secret, network

### 3. Container logs

```bash
kubectl logs <pod> -n <ns>
kubectl logs <pod> -n <ns> -c <container>     # if multiple containers
kubectl logs <pod> -n <ns> --previous          # logs from previous crash
kubectl logs <pod> -n <ns> -f                  # follow live
```

For multi-replica deployments, check all pods :
```bash
kubectl logs -l app=<label> -n <ns> --tail=100
```

### 4. Events (cluster-wide signals)

```bash
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl get events -A --field-selector type=Warning
```

Events are the cluster's diagnostic log. Read them in time order.

### 5. Resource pressure

```bash
kubectl top nodes
kubectl top pods -n <ns>
```

OOMKills with no obvious app cause usually mean the pod is at its memory limit. Check `resources.limits.memory` in the spec.

### 6. Networking

```bash
kubectl get svc -n <ns>                        # services
kubectl get endpoints <svc> -n <ns>            # are pods registered?
kubectl get ingress -n <ns>                    # ingress rules
kubectl get networkpolicies -n <ns>            # NetPol blocking?
```

If `kubectl get endpoints` shows no IPs : your service selector doesn't match any pod labels.

### 7. Storage (for stateful apps)

```bash
kubectl get pvc -n <ns>
kubectl get pv
kubectl describe pvc <name> -n <ns>
```

`Pending` PVCs usually mean no matching StorageClass or no available PV.

## Common patterns

### Pod stuck in `Pending`
- `kubectl describe` → check Events
- Usually : insufficient resources on nodes, or PVC unbound, or node taints
- `kubectl get nodes -o yaml | grep -A 5 taints` to see taints

### Pod in `CrashLoopBackOff`
- `kubectl logs <pod> --previous` to see why it died
- Check exit code in `kubectl describe pod` → `Last State`
- `137` = OOM, `139` = segfault, `1/2/126/127` = app errors

### Service not reachable
1. `kubectl get endpoints <svc>` — empty = selector doesn't match
2. From inside the cluster : `kubectl run debug -it --rm --image=nicolaka/netshoot --restart=Never -- curl svc-name.ns:port`
3. Check NetworkPolicies if they exist
4. Check DNS : `nslookup svc-name.ns.svc.cluster.local` from netshoot

### Image pull errors
- Wrong image name : `kubectl describe pod` shows pull URL
- Private registry : check `imagePullSecrets` in pod spec and `kubectl get secret <name>`
- Rate-limited (Docker Hub) : add auth or mirror to a private registry

## Tools to install on your debug toolbelt

- `k9s` — TUI for kubectl, much faster than CLI for exploration
- `stern` — multi-pod log tailing : `stern <pattern>`
- `kubectx` / `kubens` — fast context/namespace switching
- `netshoot` — debug pod image with all network tools

## Read-only principle

When investigating, **do not modify resources** without an explicit decision. Specifically :
- Don't restart deployments
- Don't delete pods
- Don't edit configmaps
- Don't apply yaml

Investigate first, build a hypothesis with evidence, present the recommended action, let the operator apply it.

## When using mcp-grafana alongside kubectl

`mcp-grafana` for the metric/log view (PromQL/LogQL), `kubectl` for the object state. They're complementary :
- "What" happened : kubectl events + describe
- "When" and "how often" : Grafana metrics
- "Why" the app crashed : kubectl logs

Correlate timestamps. An OOMKill in `kubectl events` at 14:32 should align with a memory spike in Grafana right before.
