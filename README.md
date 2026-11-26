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
| `clusters/atmos/flux-system/` | Flux itself (written by `flux bootstrap`); Git access through the GitHub App (patch in `kustomization.yaml`) |
| `clusters/atmos/apps/` | HelmRelease of the application (chart `charts/atmos/`) |
| `clusters/atmos/monitoring/` | HelmReleases of Prometheus (with kube-state-metrics and Alertmanager) and Grafana, with their values (namespace `monitoring`) |
| `charts/atmos/` | Helm chart: frontend, backend, MongoDB, Redis, Ingress, NetworkPolicy, memory limits |

Flux applies everything under `clusters/atmos/`; the chart and the values are not applied directly.

| Address | Service |
|---|---|
| `http://atmos.lan` | Application (`/` → frontend, `/api` → backend) |
| `http://grafana.atmos.lan` | Dashboards |

## Delivery

1. Jenkins pushes a new image to Nexus (`<MAJOR.MINOR>.<commits>-<short-sha>`, for example `1.0.82-8d3c3cb`).
2. Flux scans Nexus every minute (`ImageRepository`), selects the highest `MAJOR.MINOR.<commits>` (`ImagePolicy`, semver) and commits the tag to the marked lines in `spec.values` of the HelmRelease in `main` (`ImageUpdateAutomation`, commit `Deploy <tag>`).
3. Flux upgrades the Helm release; the pods are replaced one by one (rolling update). A failed upgrade is retried twice and rolled back to the last good release.

From a merge in `develop` to new pods: about 5 minutes (Jenkins polls Git every 5 minutes).

```bash
flux get helmreleases -A      # deployed version and failures (READY False with the reason)
flux get images all -A        # scanned tags, selected image, last automation run
helm history atmos -n atmos   # upgrades and rollbacks
```

**Rollback:**

```bash
flux suspend image update --all -n flux-system   # stop Flux from selecting the newest image again
git revert <Deploy commit>, Pull Request, merge  # restore the previous version
flux resume image update --all -n flux-system    # after the fix is built
```

**Logs:** `kubectl logs -n atmos deploy/backend` (add `--previous` after a restart). There is no central log storage.

## Monitoring and alerts

Installed and upgraded by Flux from `clusters/atmos/monitoring/`; chart versions pinned there:

| Release | Chart | Version |
|---|---|---|
| `prometheus` (with kube-state-metrics) | `prometheus-community/prometheus` | 30.0.1 (Prometheus v3.16.0) |
| `grafana` | `grafana-community/grafana` | 14.6.0 (Grafana 13.3.2) |

- Prometheus keeps 3 days on a 5Gi volume on `k8s-w1`. Targets outside the cluster (label `server`): node-exporter on the four servers, windows_exporter on the host PC, Docker Engine on `jenkins` and `nexus-traefik`, Traefik, and Jenkins and Nexus through Traefik over HTTPS.
- The CA of the platform (public certificate) is the ConfigMap `atmos-ca`, created before Flux installs monitoring: `kubectl create configmap atmos-ca -n monitoring --from-file=ca.crt=$HOME/.atmos/pki/ca.crt`.
- The password of the read-only Nexus user `prometheus-metrics` is the Secret `nexus-metrics` (key `password`), created from Ansible Vault in `atmos-ansible`: `kubectl create secret generic nexus-metrics -n monitoring --from-literal=password=(ansible localhost -m debug -a 'msg={{ vault_nexus_metrics_password }}' | grep msg | cut -d'"' -f4)`.
- Grafana has no volume: the data source and the dashboards (folder Atmos: servers, cluster, nodes, pods, host PC, Docker, Traefik, Jenkins, Nexus) are provisioned from the values.
- Alertmanager (part of the `prometheus` release) sends the alerts to the Telegram bot `@atmos_alerts_bot`; the rules are in `clusters/atmos/monitoring/prometheus.yaml`. The bot token is the Secret `alertmanager-telegram` (key `token`), created from Ansible Vault in `atmos-ansible`: `kubectl create secret generic alertmanager-telegram -n monitoring --from-literal=token=(ansible localhost -m debug -a 'msg={{ vault_telegram_bot_token }}' | grep msg | cut -d'"' -f4)`.
- Test the Telegram channel after a change: `kubectl port-forward -n monitoring svc/prometheus-alertmanager 9093:9093`, then `curl -X POST http://localhost:9093/api/v2/alerts -H 'Content-Type: application/json' -d '[{"labels":{"alertname":"TestAlert","server":"mac"}}]'`; FIRING arrives in about 30 seconds, RESOLVED after 5 minutes.
- The dashboards come from grafana.com and are older than some of our versions, so a few panels stay empty: Nexus JVM pools and Jetty (Nexus 3.98 uses G1 and other metric names), Jenkins uptime, CPU frequency of the host PC. The host PC CPU works through a rename rule in `clusters/atmos/monitoring/prometheus.yaml` (windows_exporter 0.32 renamed the metric).
- Grafana admin password: Secret `grafana-admin` (keys `admin-user`, `admin-password`), created before Flux installs monitoring:

```bash
kubectl create secret generic grafana-admin -n monitoring --from-literal=admin-user=admin --from-literal=admin-password=(openssl rand -base64 24)
```

## Secrets

Not stored in Git. Created by hand with `kubectl create secret`, before Flux installs what needs them; the platform ones come from Ansible Vault in `atmos-ansible` (run these commands in that folder).

The application reads the Secret `atmos-secrets` (keys `openweather-api-key`, `mongodb-password`). Create it once, before the first install (fish; the key is typed hidden, the password is generated):

```bash
read -s -P "OpenWeather API key: " key
kubectl create secret generic atmos-secrets -n atmos --from-literal=openweather-api-key=$key --from-literal=mongodb-password=(openssl rand -hex 24)
set -e key
```

The MongoDB password is set only when the volume is created: changing it in the Secret later does not change it in MongoDB.

Secrets of Flux (namespace `flux-system`):

```bash
# Git access: GitHub App atmos-flux (App ID 5934648, installation 186078775)
ansible localhost -m ansible.builtin.copy -a "content={{ vault_flux_github_app_private_key }} dest=/tmp/flux-app.pem mode=0600" > /dev/null
kubectl create secret generic flux-github-app -n flux-system --from-literal=githubAppID=5934648 --from-literal=githubAppInstallationID=186078775 --from-file=githubAppPrivateKey=/tmp/flux-app.pem
rm /tmp/flux-app.pem
# Nexus scan: read-only user and the platform CA
kubectl create secret docker-registry nexus-pull -n flux-system --docker-server=nexus.atmos.lan:5000 --docker-username=cluster-pull --docker-password=(ansible localhost -m debug -a 'msg={{ vault_nexus_pull_password }}' | grep msg | cut -d'"' -f4)
kubectl create secret generic nexus-ca -n flux-system --from-file=ca.crt=$HOME/.atmos/pki/ca.crt
```

## Install Flux on a new cluster

The patch in `clusters/atmos/flux-system/kustomization.yaml` already points Git access to the GitHub App, so its Secret must exist before the bootstrap.

1. `kubectl create namespace flux-system`, then create the Secrets of Flux above (and the Secrets of `atmos` and `monitoring`).
2. Create a fine-grained token for Kernel-Club, only `atmos-k8s`, Administration and Contents read and write, 7 days; copy it.
3. Bootstrap (the image automation controllers are extra components):

```bash
set -x GITHUB_TOKEN (pbpaste)
flux bootstrap github --token-auth --owner=Kernel-Club --repository=atmos-k8s --branch=main --path=clusters/atmos --components-extra=image-reflector-controller,image-automation-controller
set -e GITHUB_TOKEN
```

4. Delete the token on GitHub and the Secret with it: `kubectl delete secret flux-system -n flux-system`.

## Related

| Repository | Role |
|---|---|
| [atmos-dev](https://github.com/Kernel-Club/atmos-dev) | Application source and on-premise docs (`docs/on-premise/`) |
| [atmos-ansible](https://github.com/Kernel-Club/atmos-ansible) | Servers and k3s installation |
| [atmos-jenkins](https://github.com/Kernel-Club/atmos-jenkins) | CI pipeline that publishes the images |

Branching: `main` is the state of the cluster; changes via `feature/*` / `fix/*` and a Pull Request. Flux commits image tags directly to `main` ([branching model](https://github.com/Kernel-Club/atmos-dev/blob/develop/docs/on-premise/branching-model.md)).
