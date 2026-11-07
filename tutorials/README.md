# Tutorials

Step-by-step guides for the changes made to the homelab k3s cluster: new services,
changes to existing services and cluster configuration. Each tutorial explains
**what** was done, **how** to reproduce it and **why** each technical decision was made.

Environment-specific values (IPs, internal domains, passwords and tokens) appear as
placeholders, for example `<LOADBALANCER_IP>` or `argocd.<your-local-domain>`.
Replace them with the values from your own environment.

Problems found along the way are recorded in [`../troubleshooting/`](../troubleshooting/).

## Index

- [How to deploy ArgoCD on the Kubernetes cluster](how-to-deploy-argocd-on-the-kubernetes-cluster.md): install ArgoCD with Kustomize in a namespace with a ResourceQuota, expose it through Traefik and create the first Application.
- [How to deploy OpenObserve on the Kubernetes cluster](how-to-deploy-openobserve-on-the-kubernetes-cluster.md): run OpenObserve in local mode as a StatefulSet, expose its UI through Traefik, scrape it with Prometheus and verify log ingestion end to end.
