# Kubernetes Notes: Interview Prep and Production Troubleshooting

Each folder covers one topic in the same format:

- a one-liner summary
- the concept, with a Mermaid diagram
- example YAML
- common commands
- a troubleshooting table, plus debug commands
- interview Q&A
- a quick-revision table

| # | Topic | Covers |
| --- | --- | --- |
| 01 | [Architecture](01-Architecture/) | Control plane, worker nodes, `kubectl apply` flow, top interview Q&A |
| 02 | [Pod](02-POD/) | Lifecycle, init/sidecar, QoS, exit codes, CrashLoopBackOff |
| 03 | [Deployment](03-Deployment/) | Rolling updates, rollback, zero downtime, PDB |
| 04 | [ReplicaSet](04-ReplicaSet/) | Reconciliation, selectors, `FailedCreate` |
| 05 | [DaemonSet](05-DaemonSet/) | Per-node agents, tolerations, `OnDelete` |
| 06 | [StatefulSet](06-StatefullSet/) | Stable identity, `volumeClaimTemplates`, partition, AZ issues |
| 06b | [DB backup and restore](06-StatefullSet/DB-Backup-Restore.md) | DB crash/corruption scenarios, snapshots, pg_dump, PITR (CloudNativePG, WAL-G, binlogs), Velero, recovery runbook |
| 07 | [Service: NodePort](07-Serivce-NodePort/) | Ports, `externalTrafficPolicy`, firewalling |
| 08 | [Service: ClusterIP](08-Service-ClusterIP/) | Virtual IPs, EndpointSlices, kube-proxy, debug order |
| 09 | [Service: Headless](09-Service-HeadLess/) | Per-Pod DNS, StatefulSets, gRPC load balancing |
| 10 | [Service: ExternalName](10-Service-External/) | CNAME aliases, selectorless Services |
| 11 | [Service: LoadBalancer](11-Service-LB/) | NLB/Azure LB, target modes, `<pending>` |
| 12 | [Ingress / Gateway API](12-IngressController/) | Controllers, TLS, 404/502/503/504, HTTPRoute |
| 13 | [RBAC](13-RBAC/) | Roles, bindings, `auth can-i`, escalation risks |
| 14 | [NetworkPolicy](14-NetworkPolicy/) | Default deny, DNS egress, AND vs OR selectors |
| 15 | [ConfigMap and Secret](15-ConfigMap-Secret/) | Env vs volume, updates, External Secrets, encryption |
| 16 | [PV / PVC / StorageClass](16-PV-PVC-StorageClass/) | CSI, access modes, `WaitForFirstConsumer`, Multi-Attach |
| 17 | [Probes](17-Probes/) | Startup, readiness, liveness, common mistakes |
| 18 | [Autoscaling](18-Autoscaling-HPA-VPA-CA-Karpenter/) | HPA, VPA, KEDA, Cluster Autoscaler, Karpenter |
| 19 | [Job and CronJob](19-Job-CronJob/) | Retries, deadlines, concurrency, time zones |
| 20 | [Scheduling](20-Scheduling-Taints-Affinity/) | Affinity, taints, topology spread, priority, `FailedScheduling` |
| 21 | [Namespace, Quota, LimitRange](21-Namespace-ResourceQuota-LimitRange/) | Multi-tenancy, quota errors, stuck namespaces |
| 22 | [ServiceAccount and cloud identity](22-ServiceAccount-IRSA-WorkloadIdentity/) | IRSA, EKS Pod Identity, AKS Workload Identity |
| 23 | [Pod Security](23-PodSecurity-SecurityContext/) | securityContext, PSA levels, Kyverno/Gatekeeper |
| 24 | [CoreDNS](24-CoreDNS/) | Resolution flow, `ndots`, NodeLocal DNSCache |
| 25 | [Node troubleshooting](25-Node-Troubleshooting/) | NotReady, pressure, eviction, on-node debugging |
| 26 | [Cluster operations](26-Cluster-Ops-Backup-Upgrade/) | etcd backup, Velero, upgrades, certificates, DR |
| 27 | [Helm](27-Helm/) | Charts, values, releases, stuck upgrades |

## Universal debugging toolkit

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -30
kubectl get pods -A -o wide | grep -v -E 'Running|Completed'
kubectl describe <kind> <name> -n <ns>          # read the Events section first
kubectl logs <pod> -n <ns> [-c container] --previous
kubectl debug -it <pod> -n <ns> --image=busybox:1.36 --target=<container>
kubectl debug node/<node> -it --image=busybox:1.36
kubectl auth can-i <verb> <resource> -n <ns> --as=<user|system:serviceaccount:ns:sa>
kubectl top nodes && kubectl top pods -A
```

> Commands are written for recent Kubernetes (v1.30+) on EKS and AKS. Version-specific details are noted where they matter. Test commands in a non-production cluster first.
