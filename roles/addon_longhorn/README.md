# addon_longhorn

Installs [Longhorn](https://longhorn.io/) distributed block storage, based on the [K3s storage docs](https://docs.k3s.io/add-ons/storage). Those docs apply a pinned `longhorn.yaml`. This role uses Longhorn's Helm chart through K3s's `HelmChart` controller, at a newer version, and also prepares the hosts.

On every node it installs the iSCSI, NFS and cryptsetup packages, loads `iscsi_tcp` and `dm_crypt`, and starts `iscsid`. It then waits for the Longhorn CSI plugin to be ready.

## Usage

```yaml
addon_longhorn_enabled: true
```

```bash
ansible-playbook playbooks/addons.yml -i inventory.yml
```

Run it after a CNI is working. `addon_calico` runs first in `playbooks/addons.yml`.

## Variables

| Variable | Default | Notes |
| --- | --- | --- |
| `addon_longhorn_enabled` | `false` | Turns the role on. |
| `addon_longhorn_version` | `1.13.0` | Chart version. Needs Kubernetes 1.34 or later. |

## Disk space

Nothing needs partitioning. Longhorn stores data in `/var/lib/longhorn` on the root filesystem by default, and these defaults apply:

| Setting | Default | Effect |
| --- | --- | --- |
| `storageReservedPercentageForDefaultDisk` | 30% | Share of the disk kept away from Longhorn. |
| `storageMinimalAvailablePercentage` | 25% | No new replicas once real free space falls below this. |
| `storageOverProvisioningPercentage` | 100% | Total volume size can't exceed usable capacity. |

Volumes default to 3 replicas. On a 3-node cluster every volume has a copy on every node, so the smallest node limits total volume size.

## Default storage class

Longhorn is for workloads that need distributed storage, so it isn't the default. K3s's `local-path` stays the default class, and a PVC has to ask for Longhorn:

```yaml
storageClassName: longhorn
```

## SELinux

Tested on enforcing Fedora 44 with K3s's `--selinux`, with no denials. Older combinations can leave volumes stuck in an attach/detach loop. If that happens, look for `iscsiadm` `dac_override` denials and see the [Longhorn KB](https://longhorn.io/kb/troubleshooting-volume-attachment-fails-due-to-selinux-denials/).

## Limits

- Only tested on Fedora. Package names for other OS families come from Longhorn's docs.
- ARM32 isn't supported.
- If `multipathd` is running, it must be configured to ignore Longhorn's devices. The role doesn't do this, so see the [Longhorn KB](https://longhorn.io/kb/troubleshooting-volume-with-multipath/).
- Airgapped clusters aren't supported.

## Removing it

`playbooks/cluster-reset.yml` deletes `/var/lib/longhorn` on every node, which destroys all volume data. Deleting the manifest file doesn't uninstall Longhorn: follow [Longhorn's uninstall guide](https://longhorn.io/docs/1.13.0/deploy/uninstall/).
