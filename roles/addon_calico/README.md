# addon_calico

Replaces Flannel with [Calico](https://docs.tigera.io/calico/latest/about/). It puts Calico's CRD manifest and a `tigera-operator` `HelmChart` in K3s's manifests directory, then waits until every node is `Ready` and Calico reports `Available`.

## Usage

Decide this before building the cluster, because K3s can't change CNI on a running one. Set these in inventory (see `inventory-addons-sample.yml`):

```yaml
addon_calico_enabled: true
server_config_yaml: |
  flannel-backend: none
  disable-network-policy: true
  cluster-cidr: 10.42.0.0/16
```

```bash
ansible-playbook playbooks/cluster.yml -i inventory.yml   # new cluster
ansible-playbook playbooks/addons.yml -i inventory.yml    # add-ons only, no K3s restart
```

The role stops with an error if the server config doesn't match. `site.yml` on its own leaves nodes `NotReady` until `addons.yml` has run.

## Variables

| Variable | Default | Notes |
| --- | --- | --- |
| `addon_calico_enabled` | `false` | Turns the role on. |
| `addon_calico_version` | `v3.33.0` | Used for the CRDs and the chart. v3.33 is tested on Kubernetes 1.35 to 1.37, so check it against `k3s_version`. |
| `addon_calico_encapsulation` | `VXLAN` | `VXLAN`, `VXLANCrossSubnet` or `None`. BGP is on for the last two. |

The IP pool is taken from `cluster-cidr` in `server_config_yaml`.

## Firewall

With `manage_firewall` on (the default), the role opens these on ufw or firewalld:

| Port | Used for | Opened when |
| --- | --- | --- |
| `5473/tcp` | Typha | Always |
| `4789/udp` | VXLAN | `VXLAN`, `VXLANCrossSubnet` |
| `179/tcp` | BGP | `VXLANCrossSubnet`, `None` |

## SELinux

On hosts with SELinux enabled, the role labels `/var/lib/calico` as `container_file_t`, because the operator can't read `/var/lib/calico/mtu` otherwise when K3s runs with `--selinux`. The file context rule stays on the host after a reset, which is harmless.

To confine containers on enforcing hosts, set `--selinux` in inventory. Tested this way on Fedora 44:

```yaml
extra_server_args: "--selinux"
extra_agent_args: "--selinux"
```

## Limits

- K3s v1.36 or later. The operator's v3 CRD mode needs `MutatingAdmissionPolicy`, which is off by default before Kubernetes 1.36.
- Single-stack IPv4 only.
- No IP-in-IP, because the ufw module can't open IP protocol 4.
- Airgapped clusters aren't supported, because the charts are pulled from the network.
- Calico finds each node's address from the node's `InternalIP`. On multi-NIC hosts, set K3s's `node-ip` to choose the interface.

## Removing it

`playbooks/cluster-reset.yml` runs the upstream `reset.yml`, then clears Calico's files, interfaces and iptables rules from each node. It destroys the cluster. Deleting the manifest files doesn't uninstall Calico.
