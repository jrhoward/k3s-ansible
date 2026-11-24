# addon_kata

Installs [Kata Containers](https://katacontainers.io/) from its upstream release as the `kata` containerd runtime, and creates a `kata` RuntimeClass. Pods that set `runtimeClassName: kata` run in their own lightweight VM with their own guest kernel, instead of sharing the host kernel.

## Usage

Enable it on every node with KVM. The role fails on a host without `/dev/kvm`, so turn on VT-x or AMD-V (SVM) in the BIOS first.

```yaml
k3s_cluster:
  vars:
    addon_kata_enabled: true
```

```bash
ansible-playbook playbooks/addons.yml -i inventory.yml
```

Then opt pods in:

```yaml
spec:
  runtimeClassName: kata
```

The RuntimeClass selects nodes labelled `katacontainers.io/kata-runtime=true`, which the role sets, so Kata pods only land on Kata nodes. It also adds a pod overhead of `250m` CPU and `160Mi` memory, so the scheduler counts the VM.

## What it does

1. Loads `kvm_intel` or `kvm_amd`, and fails if `/dev/kvm` still doesn't exist.
2. Downloads `kata-static-<version>-amd64.tar.zst` (about 1 GB), checks it against the pinned SHA-256, unpacks it to `/opt/kata` (2.3 GiB), and deletes the tarball. A version file in `/opt/kata` makes later runs skip this until `addon_kata_version` changes, and a new version replaces `/opt/kata`.
3. Adds the `kata` runtime to K3s's containerd through a drop-in in `config-v3.toml.d/`, pointing at the release's Rust shim (`runtime-rs`).
4. Restarts K3s if any of that changed, and waits for the node to be `Ready`. It runs one host at a time.
5. Labels the node, and creates the `kata` RuntimeClass from the first server.

It uses the release's default configuration (`/opt/kata/share/defaults/kata-containers/runtime-rs/configuration.toml`): QEMU, virtio-fs, the release's guest kernel and image, 1 vCPU and 2 GiB of guest memory per pod by default. Pods can't change hypervisor settings, because the drop-in passes no pod annotations.

## Variables

| Variable | Default | Notes |
| --- | --- | --- |
| `addon_kata_enabled` | `false` | Turns the role on for a host. |
| `addon_kata_version` | `4.3.1` | Kata release tag. |
| `addon_kata_sha256` | 4.3.1's digest | SHA-256 of the amd64 `kata-static` tarball. The release publishes no checksum file, so take it from GitHub's asset digest. Change it with the version. |

## SELinux

Unlike gVisor, Kata keeps SELinux on. With K3s's `--selinux`, the shim starts QEMU and virtiofsd in `container_kvm_t` (from Fedora's `container-selinux`), each pod with its own MCS categories, so an escape from the VM is still confined. Nothing is relabelled and no policy module is added.

QEMU logs one AVC denial per pod start, `{ create }` on an `io_uring` anon inode. It's harmless: QEMU falls back and the pod starts.

## Why the upstream release, not Fedora's package

Fedora's `kata-containers` package builds the guest from the host kernel with dracut at install time. On Fedora 44 that needed `dbus-daemon`, missed `kata-agent`'s `libseccomp`, only ran its builder for `kubelet.service` (which K3s doesn't have), and still stalled at the guest network setup. The upstream release ships a tested guest kernel and image, and works as is.

## Limits

- Needs KVM, and so bare metal or nested virtualisation.
- GPU pods can't use it. The release has NVIDIA GPU configurations, but this role doesn't set them up.
- Each pod is a VM: slower to start than runc (a few seconds) and it costs memory, so keep the overhead in mind on small nodes.
- Falco on the host sees QEMU and virtiofsd, not the processes inside the VM.
- Removing the role doesn't uninstall `/opt/kata`, the drop-in, the label or the RuntimeClass.
