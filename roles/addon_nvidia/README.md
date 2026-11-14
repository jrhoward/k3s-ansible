# addon_nvidia

Installs the NVIDIA driver and container toolkit on the host, so pods can use the GPU through K3s's `nvidia` runtime. The driver comes from [RPM Fusion](https://rpmfusion.org/Howto/NVIDIA) as an `akmod`, and the toolkit from [NVIDIA's repository](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).

It doesn't install the GPU Operator or a device plugin, which belong in the cluster. If you run the GPU Operator, turn off its driver and toolkit, because this role provides them:

```yaml
driver:
  enabled: false
toolkit:
  enabled: false
```

## Usage

Enable it per host, on the nodes that have an NVIDIA GPU:

```yaml
agent:
  hosts:
    gpu-node:
      addon_nvidia_enabled: true
    old-gpu-node:
      addon_nvidia_enabled: true
      addon_nvidia_driver_branch: 580xx
```

```bash
ansible-playbook playbooks/addons.yml -i inventory.yml
```

It runs in its own play in `playbooks/addons.yml`, one host at a time, after the other add-ons.

## What it does

1. Checks that the host runs Fedora, has an NVIDIA GPU, and has Secure Boot off.
2. Adds the RPM Fusion free and nonfree repositories and NVIDIA's toolkit repository.
3. Installs `akmod-nvidia`, `xorg-x11-drv-nvidia-cuda` and `nvidia-container-toolkit`, then upgrades every package on the host.
4. Builds the kernel module for the default boot kernel with `akmods`.
5. Reboots if the loaded driver isn't the installed version, or the running kernel isn't the default boot kernel. That covers the first install, where `nouveau` holds the GPU, and every driver or kernel upgrade.
6. Restarts K3s if the packages changed without a reboot.
7. Waits for the `nvidia` runtime in K3s's containerd config, checks `nvidia-smi -L`, and waits for the node to be `Ready`.

## Variables

| Variable | Default | Notes |
| --- | --- | --- |
| `addon_nvidia_enabled` | `false` | Turns the role on for a host. |
| `addon_nvidia_driver_branch` | `""` | RPM Fusion driver branch. Empty for the current driver; `580xx` for Maxwell, Pascal and Volta GPUs. |

## Driver branches

NVIDIA's 580 series is the last driver branch for Maxwell, Pascal and Volta GPUs (GTX 9xx and 10xx, Titan V). Set `addon_nvidia_driver_branch: 580xx` on those hosts. The current branch supports Turing and newer.

CUDA 13 also dropped those older GPUs, so pods on a `580xx` host need images built against CUDA 12.

## Versions

Nothing is pinned or version-locked. Every run upgrades the whole host, the kernel and the NVIDIA packages included, and reboots the node for a new driver or kernel. Before you run the role, check that the newest driver supports the CUDA version in your images.

The full upgrade isn't optional. `akmod-nvidia` needs `kernel-devel` for the newest kernel, which pulls in only `kernel-core` without the rest of the kernel's modules. A node left like that can boot a kernel with no network driver.

`dnf update` upgrades them too. It builds the new module, but the old one stays loaded until the next reboot.

## Container runtime

K3s finds `nvidia-container-runtime` when it starts and adds the `nvidia` runtime to its containerd config, so the role doesn't run `nvidia-ctk runtime configure`. K3s also creates the `nvidia` RuntimeClass. Pods need `runtimeClassName: nvidia` unless something sets it as the default runtime.

## Limits

- Fedora only, because the driver comes from RPM Fusion.
- Secure Boot must be off. `akmods` builds unsigned modules, and this role doesn't enrol a signing key.
- Reboots drop the node's pods. The role doesn't drain the node first.
- Removing the role doesn't uninstall the packages.
