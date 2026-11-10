# Concepts

Plain-language explanations of the concepts behind the homelab: networking, Kubernetes,
storage, security and the tools around them. Each file starts from an everyday analogy,
explains how the concept works step by step and ends with commands to try it yourself.

For step-by-step guides of the changes made to the cluster, see [`../tutorials/`](../tutorials/).

Environment-specific values (IPs, internal domains, passwords and tokens) appear as
placeholders, for example `<LOADBALANCER_IP>` or `rabbitmq.<your-local-domain>`.

## Index

- [DNS (Domain Name System)](dns.md): how names become IP addresses, the record types (A, AAAA, CNAME, MX, NS, TXT, PTR, SRV...), how Linux decides where to look, DNS inside Kubernetes, and the commands to test each network layer.
