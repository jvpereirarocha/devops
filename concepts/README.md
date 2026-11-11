# Concepts

Plain-language explanations of the concepts behind the homelab: networking, Kubernetes,
storage, security and the tools around them. Each file starts from an everyday analogy,
explains how the concept works step by step and ends with commands to try it yourself.

For step-by-step guides of the changes made to the cluster, see [`../tutorials/`](../tutorials/).

Environment-specific values (IPs, internal domains, passwords and tokens) appear as
placeholders, for example `<LOADBALANCER_IP>` or `rabbitmq.<your-local-domain>`.

## Index

- [DNS (Domain Name System)](dns.md): how names become IP addresses, the record types (A, AAAA, CNAME, MX, NS, TXT, PTR, SRV...), how Linux decides where to look, DNS inside Kubernetes, and the commands to test each network layer.
- [Kubernetes: advanced concepts](kubernetes-advanced-concepts.md): RBAC, scheduling (affinity, taints), autoscaling (HPA/VPA), NetworkPolicy, Pod security, CRDs and operators, Helm/Kustomize, GitOps and how a request becomes a running Pod.
- [Kubernetes: basic concepts](kubernetes-basic-concepts.md): cluster, node, Pod, ReplicaSet, Deployment, labels, namespaces and Services (ClusterIP, NodePort, LoadBalancer, ExternalName).
- [Kubernetes: intermediate concepts](kubernetes-intermediate-concepts.md): ConfigMap, Secret, volumes (PV/PVC/StorageClass), probes, requests and limits, Ingress, Job/CronJob, StatefulSet, DaemonSet and rolling updates.
