# ITL.Container.Registry

A plain [Docker Registry v2](https://github.com/distribution/distribution) deployed to the ITL Talos cluster, exposed through the public Traefik Ingress with a real Let's Encrypt certificate (`letsencrypt-issuer`, HTTP-01). No auth, no third-party dependency - originally stood up as a workaround while `nweistra/itlwebsite` was rate-limited on Docker Hub, kept as its own repo since it's generally useful for any image that needs a quick push target reachable from the cluster.

## Why Ingress instead of NodePort + insecure-registries

A NodePort registry needs every Docker client (your workstation, each Talos node's containerd) to be configured to trust it as an insecure/HTTP registry - Docker Desktop requires an `insecure-registries` entry plus a restart, and Talos needs a `machine.registries.mirrors` patch applied to every node. Going through Ingress with a real cert means both sides trust it the normal way: no client-side config, no per-node machine config patches.

## Deploy

```bash
helm upgrade --install itl-registry itl.registry -n local-registry --create-namespace
```

Requires [cert-manager](https://github.com/ITlusions/ITL.K8s) with a `letsencrypt-issuer` ClusterIssuer, and a public DNS record for each `ingress.hosts` entry pointing at the cluster's public Traefik (NAT'd to `10.99.99.27`, see `ITlusions/ITL.K8s`).

## Configuration

| Value | Default | Description |
|---|---|---|
| `namespace` | `local-registry` | Namespace to deploy into |
| `image.repository` / `image.tag` | `registry` / `2` | The registry image itself |
| `service.port` | `5000` | Registry API port |
| `service.nodePort` | `null` | Set to also expose a NodePort, e.g. for direct node-to-node pulls without Ingress |
| `ingress.enabled` | `true` | Expose via Ingress |
| `ingress.hosts` | `registry.itlusions.nl`, `registry.itlusions.com` | Hostnames to route and issue the cert for |
| `persistence.enabled` | `false` | `emptyDir` by default - images are lost on pod restart. Set `true` + `storageClassName` for real persistence |

## Status

Currently **not persistent** (`emptyDir`): fine as a scratch/temporary push target, not for images you need to keep. Enable `persistence.enabled` with a real `storageClassName` (see [ITL.K8s storage classes](https://github.com/ITlusions/ITL.K8s/blob/main/docs/4-storage/4.1-storage-classes.md), e.g. `ha-dbs-lh`) before relying on this long-term.

No authentication is configured - anyone who can reach the hostnames can push and pull. Fine while it's a dev scratch registry; add [registry auth](https://distribution.github.io/distribution/about/deploying/#native-basic-auth) or restrict it behind the internal Traefik instance before storing anything sensitive.
