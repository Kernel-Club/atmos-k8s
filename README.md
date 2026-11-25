# atmos-k8s

Everything that runs inside the on-premise Kubernetes cluster of **Atmos** (nodes `k8s-cp` and `k8s-w1`), delivered by Flux (GitOps).

```text
Nexus (nexus-traefik) ─► Flux: new tag ─► commit to main ─► Helm upgrade ─► http://atmos.lan
```

> k3s itself is installed by [atmos-ansible](https://github.com/Kernel-Club/atmos-ansible) (`roles/k3s/`). This repository holds only the workloads.
>
> The cluster pulls images only from Nexus on the `nexus-traefik` VM, so it keeps running while the `jenkins` VM is stopped.

## Contents

| Path | What it does |
|---|---|
| `flux-system/` | Flux configuration, HelmRelease of the application, image automation |
| `charts/atmos/` | Helm chart: frontend, backend, MongoDB, Redis, Ingress, NetworkPolicy, memory limits |
| `monitoring/` | Prometheus, Grafana, kube-state-metrics (namespace `monitoring`) |

| Address | Service |
|---|---|
| `http://atmos.lan` | Application (`/` → frontend, `/api` → backend) |
| `http://grafana.atmos.lan` | Dashboards |

## Delivery

1. Jenkins pushes a new image to Nexus (`<MAJOR.MINOR>.<commits>-<short-sha>`, for example `1.0.82-8d3c3cb`).
2. Flux selects the highest version and commits the tag to `spec.values` of the HelmRelease in `main`.
3. Flux upgrades the Helm release and the pods are replaced.

```bash
flux get helmreleases -A      # deployed version and failures
flux get images all -A        # scanned tags and selected image
```

**Rollback:**

```bash
flux suspend image update --all -n flux-system   # stop Flux from selecting the newest image again
git revert <tag-commit> && git push              # restore the previous version
flux resume image update --all -n flux-system    # after the fix is built
```

**Logs:** `kubectl logs -n atmos deploy/backend` (add `--previous` after a restart). There is no central log storage.

## Monitoring (before Flux)

Installed by hand with Helm from the values in `monitoring/`; versions pinned here:

| Release | Chart | Version |
|---|---|---|
| `prometheus` (with kube-state-metrics) | `prometheus-community/prometheus` | 30.0.1 (Prometheus v3.16.0) |
| `grafana` | `grafana-community/grafana` | 14.6.0 (Grafana 13.3.2) |

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm upgrade --install prometheus prometheus-community/prometheus --version 30.0.1 -n monitoring --create-namespace -f monitoring/prometheus-values.yaml
helm upgrade --install grafana grafana-community/grafana --version 14.6.0 -n monitoring -f monitoring/grafana-values.yaml
```

- Prometheus keeps 3 days on a 5Gi volume on `k8s-w1` and scrapes node-exporter on all four servers (job `node-exporter`, label `server`).
- Grafana has no volume: the data source and the dashboards (folder Atmos: servers, cluster, nodes, pods) are provisioned from the values.
- Grafana admin password: Secret `grafana-admin` (keys `admin-user`, `admin-password`), created before the install:

```bash
kubectl create secret generic grafana-admin -n monitoring --from-literal=admin-user=admin --from-literal=admin-password=(openssl rand -base64 24)
```

## Secrets

Not stored in Git. Created by hand with `kubectl create secret`: OpenWeather key, MongoDB password, Grafana admin password, Nexus pull credentials and CA for Flux.

The application reads the Secret `atmos-secrets` (keys `openweather-api-key`, `mongodb-password`). Create it once, before the first install (fish; the key is typed hidden, the password is generated):

```bash
read -s -P "OpenWeather API key: " key
kubectl create secret generic atmos-secrets -n atmos --from-literal=openweather-api-key=$key --from-literal=mongodb-password=(openssl rand -hex 24)
set -e key
```

The MongoDB password is set only when the volume is created: changing it in the Secret later does not change it in MongoDB.

## Manual install (before Flux)

```bash
helm lint charts/atmos
helm upgrade --install atmos charts/atmos -n atmos --create-namespace
```

## Related

| Repository | Role |
|---|---|
| [atmos-dev](https://github.com/Kernel-Club/atmos-dev) | Application source and on-premise docs (`docs/on-premise/`) |
| [atmos-ansible](https://github.com/Kernel-Club/atmos-ansible) | Servers and k3s installation |
| [atmos-jenkins](https://github.com/Kernel-Club/atmos-jenkins) | CI pipeline that publishes the images |

Branching: `main` is the state of the cluster; changes via `feature/*` / `fix/*` and a Pull Request. Flux commits image tags directly to `main` ([branching model](https://github.com/Kernel-Club/atmos-dev/blob/develop/docs/on-premise/branching-model.md)).
