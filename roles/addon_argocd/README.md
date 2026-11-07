# addon_argocd

Installs [Argo CD](https://argo-cd.readthedocs.io/) through K3s's `HelmChart` controller, then waits until it's running. It can take its Helm values and a root `Application` from files in your GitOps repo, so they stay defined in one place.

## Usage

```yaml
addon_argocd_enabled: true
addon_argocd_values_file: /path/to/lab/argocd/values.yaml
addon_argocd_root_app_file: /path/to/lab/home-lab-platform-bootstrap.yaml
```

```bash
ansible-playbook playbooks/addons.yml -i inventory.yml
```

It needs a working CNI, and runs after `addon_calico` in `playbooks/addons.yml`.

## Variables

| Variable | Default | Notes |
| --- | --- | --- |
| `addon_argocd_enabled` | `false` | Turns the role on. |
| `addon_argocd_version` | `10.11.6` | Chart version, which is Argo CD v3.5.6. |
| `addon_argocd_values_file` | `""` | Helm values file on the control node. Leave empty for chart defaults. |
| `addon_argocd_root_app_file` | `""` | Root `Application` manifest on the control node, applied unchanged. Leave empty to install with nothing to sync. |

## Root Application

The role applies your manifest as it is, so its sync policy is yours to set. With a manual root, nothing deploys until you sync it in Argo CD.

Ansible owns Argo CD and this one `Application`. Don't add workloads to this repo.

## Access

The UI isn't exposed outside the cluster:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:80
```

Open `http://localhost:8080` and log in as `admin`. Change the password after the first login. With `server.insecure: "true"` in your values, Argo CD serves plain HTTP and leaves TLS to the ingress. With chart defaults, forward `8080:443` and use `https://`.

## Limits

- Public repos only for now. A private repo needs a `argocd.argoproj.io/secret-type: repository` Secret, with credentials from ansible-vault.
- Single replicas, not HA.
- Airgapped clusters aren't supported.

## Removing it

Deleting the manifest files doesn't uninstall Argo CD or stop it syncing. `playbooks/cluster-reset.yml` removes it with the rest of the cluster.
