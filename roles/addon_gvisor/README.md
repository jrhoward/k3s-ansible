# addon_gvisor

Installs [gVisor](https://gvisor.dev/) on the host as the `runsc` containerd runtime, and creates a `gvisor` RuntimeClass. Pods that set `runtimeClassName: gvisor` run under gVisor's user-space kernel instead of sharing the host kernel, which limits what a compromised container can reach.

## Usage

Enable it on every node that can run sandboxed pods. The RuntimeClass has no node selector, so a `gvisor` pod scheduled on a node without `runsc` fails to start.

```yaml
k3s_cluster:
  vars:
    addon_gvisor_enabled: true
```

```bash
ansible-playbook playbooks/addons.yml -i inventory.yml
```

Then opt pods in:

```yaml
spec:
  runtimeClassName: gvisor
```

## What it does

1. Downloads the release tarball for the host's architecture and checks it against the release's `SHA256SUMS`.
2. Unpacks `runsc`, `containerd-shim-runsc-v1` and the `gvisor-bin/` helpers to `/usr/local/bin`, and installs the `runsc-strip-selinux` wrapper there.
3. Adds the `runsc` runtime to K3s's containerd through a drop-in in `config-v3.toml.d/`, and writes `runsc.toml` next to it.
4. Restarts K3s if any of that changed, and waits for the node to be `Ready`. It runs one host at a time.
5. Creates the `gvisor` RuntimeClass, with handler `runsc`, from the first server.

## Variables

| Variable | Default | Notes |
| --- | --- | --- |
| `addon_gvisor_enabled` | `false` | Turns the role on for a host. |
| `addon_gvisor_version` | `20261102.0` | gVisor release, without the `release-` prefix. |

## SELinux

`runsc` refuses to start a container whose spec has an SELinux process label, and K3s's `--selinux` gives every container one. The shim therefore calls `runsc-strip-selinux`, set as `binary_name` in `runsc.toml`. The wrapper removes `selinuxLabel` from the spec, then runs `runsc`.

As a result, gVisor's sandbox processes run in containerd's `container_runtime_t` domain, not `container_t`. gVisor's own isolation (its user-space kernel, seccomp filters and namespaces) still applies, and SELinux still confines every pod that doesn't use the `gvisor` class. The cost is that if something escaped gVisor, SELinux wouldn't contain it the way it contains a normal container breakout. On hosts without SELinux the wrapper does nothing.

## Limits

- It uses gVisor's default `systrap` platform, which needs no KVM.
- GPU pods can't use it. gVisor's `nvproxy` isn't configured, and a pod has only one runtime class, so it can't be `nvidia` and `gvisor` at once.
- Not every workload runs under gVisor. Syscall-heavy and I/O-heavy pods are slower, and some syscalls are unsupported.
- Removing the role doesn't uninstall the binaries, the drop-in or the RuntimeClass.
