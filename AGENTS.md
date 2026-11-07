# AGENTS.md

This repo is `jrhoward/k3s-ansible`, a fork of [`k3s-io/k3s-ansible`](https://github.com/k3s-io/k3s-ansible) (Ansible collection `k3s.orchestration`). Upstream deliberately stays a vanilla K3s install. This fork adds cluster add-ons (Calico) and bootstrapped services (Argo CD) on top of it.

## The main rule: add files, don't edit upstream ones

Every upstream file stays byte-identical to `upstream/main`, so that syncing with upstream never conflicts. This covers:

- `roles/{prereq,airgap,raspberrypi,k3s_server,k3s_agent,k3s_upgrade}/`
- `playbooks/{site,upgrade,reset,reboot}.yml`
- `tests/*.yml` and `.github/` files that already exist
- `README.md`, `CHANGELOG.md`, `inventory-sample.yml`, `Vagrantfile`, `galaxy.yml`, `ansible.cfg`, `.ansible-lint`, `.yamllint`, `collections/requirements.yml`, `meta/`

Before changing behaviour, check the existing hooks, in this order:

1. **Inventory variables** the roles already read: `server_config_yaml`, `agent_config_yaml`, `extra_server_args`, `extra_agent_args`, `extra_install_envs`, `extra_service_envs`, `extra_manifests`, `registries_config_yaml`.
2. **A new role or playbook** that runs before or after the upstream ones through `import_playbook`.
3. **Overriding an upstream role's defaults** from inventory.

If none of these can do it, stop and ask the user. Don't patch an upstream file on your own initiative. If the user agrees to a change, keep it small and generic enough to offer upstream, and list it under "Upstream deviations" at the end of this file.

To check that the rule held, run `git diff --stat upstream/main...HEAD --diff-filter=MD`. It should list only entries from "Upstream deviations".

## Layout for fork code

`roles/addon_argocd`, the workflow and any later add-ons don't exist yet. Create them as needed, using these names:

```text
roles/addon_calico/             # one role per add-on: defaults/, tasks/, templates/, README.md
roles/addon_longhorn/
roles/addon_argocd/
playbooks/addons.yml            # add-ons only, for an existing cluster
playbooks/cluster.yml           # import_playbook site.yml, then addons.yml
playbooks/cluster-reset.yml     # import_playbook reset.yml, then add-on host cleanup
inventory-addons-sample.yml     # sample vars, since inventory-sample.yml is upstream's
tests/addons.yml                # inventory for a new workflow
.github/workflows/addons.yml    # a new workflow; don't edit integration.yml
```

`roles_path` is `./roles`, so new roles are found automatically, and they ship in the collection as `k3s.orchestration.addon_*`. Keep the upstream namespace, because changing it means editing `galaxy.yml`.

Document add-ons in each role's `README.md`, not the top-level one.

## How add-ons are installed

Prefer K3s's own deploy controller: template a `HelmChart` CR or plain manifest into `/var/lib/rancher/k3s/server/manifests/` on the `server` group.

- This needs no new collections and no `helm` or `kubectl` on the control node.
- Name files `addon-<name>.yaml` so they can't collide with K3s's packaged manifests.
- Use directory mode `0700` and file mode `0600`, as `prereq` does.
- That path is correct even with a custom `k3s_server_location`, because `prereq` symlinks it.
- Deleting a file from that directory doesn't delete its resources from the cluster. Removing an add-on needs an explicit delete.
- Pin every chart and image version in role defaults. Never use `latest`.
- Use `extra_manifests` only for static files. It is a plain `copy` with no templating, it runs in `prereq` on every node, and relative paths resolve from the playbook directory.
- For cluster commands and waits, run `k3s kubectl` on a server with `become: true` and `run_once: true`, the way `k3s_server` does.

## Behaviour of the upstream code to design around

- **`site.yml` restarts every K3s server on each run.** Iterate on add-ons with `addons.yml`, not `cluster.yml`.
- **`k3s_server_serial`:** on HA clusters pass `-e k3s_server_serial=1`. It is a play keyword, so inventory values are ignored.
- **`site.yml` finishes with nodes `NotReady` when there's no CNI.** Its checks look at node listing and `EtcdIsVoter`, never `Ready`, so add-on roles must do their own waiting.
- **Upstream role defaults aren't visible to other roles.** `api_port`, `server_group` and `agent_group` live in `roles/*/defaults`, so a new role must define its own or read them from inventory.
- **Facts aren't available when `addons.yml` runs alone.** `prereq` gathers them in `site.yml`. Set `gather_facts: true` and call `ansible.builtin.service_facts` before any firewall logic.
- **`k3s_upgrade` rewrites `config.yaml`** from the same `server_config_yaml`, so keep those settings in inventory rather than editing nodes by hand.

## Calico

- **Server config has to come from inventory.** The server role runs before any add-on role, so `addon_calico` can't set these, and K3s can't switch CNI on a live cluster. Changing CNI means `reset` and rebuild. Assert these values at the start of the role and fail with a clear message if they're wrong:

  ```yaml
  server_config_yaml: |
    flannel-backend: none
    disable-network-policy: true
    cluster-cidr: 10.42.0.0/16
  ```

- **CIDR:** the Calico IP pool must equal `cluster-cidr`. Calico's own default is `192.168.0.0/16` and K3s's is `10.42.0.0/16`. Derive the pool from `server_config_yaml` rather than setting it twice. `prereq` already reads `cluster-cidr` from there for its firewall rules.
- **Install:** since v3.32 the `tigera-operator` chart doesn't ship CRDs, so they go in first, and the role waits for `installations.operator.tigera.io` to be `Established`. Use the `projectcalico.org.v3` CRDs, as Calico's K3s and Helm docs do, from the release's `manifests/v3_projectcalico_org.yaml`. Don't install them as a `HelmChart`: the release is over Helm's 1 MiB Secret limit. The operator is a `HelmChart` with `spec.bootstrap: true`, because its install job has to run before a CNI exists.
- **Versions:** the role needs K3s v1.36 or later. The operator crashes in v3 CRD mode without `MutatingAdmissionPolicy`, which Kubernetes 1.34 and 1.35 leave off by default, even though Calico's docs list a manifest for them. `inventory-sample.yml` pins an older K3s, so don't copy its `k3s_version`.
- **IP forwarding:** K3s requires it in the CNI config. With the operator, set `calicoNetwork.containerIPForwarding: Enabled`.
- **Encapsulation:** set it explicitly on the IP pool rather than relying on the default. BGP is off for `VXLAN` and on for everything else.
- **Firewall:** `prereq` opens only Flannel's ports. When `manage_firewall` is true and ufw or firewalld is active, `addon_calico` must open what the chosen mode needs: `179/tcp` for BGP, `4789/udp` for VXLAN, and `5473/tcp` for Typha. IP-in-IP isn't supported, because the ufw module can't open IP protocol 4. Follow the ufw and firewalld blocks in `roles/prereq/tasks/main.yml`.
- **CNI paths:** leave them at Calico's defaults, `/etc/cni/net.d` and `/opt/cni/bin`. A previous working K3s and Calico cluster on these nodes used them, so don't set `cni.binDir` or `cni.confDir`.
- **MTU:** leave it on auto-detect. That cluster got `1450` on a 1500 NIC with VXLAN.
- **Stale CNI config:** these files survive `reset.yml`. A leftover `10-calico.conflist` can make a node report `Ready` before Calico runs, so `Ready` alone doesn't prove Calico is up, which is why the role also waits for `tigerastatus/calico`. The file can also chain plugins from the old cluster, such as `istio-cni`.
- **SELinux:** with K3s's `--selinux`, the operator runs as `container_t` and can't read `/var/lib/calico/mtu` unless it's labelled `container_file_t`. Calico then stays `Degraded` while every pod looks healthy. No denial is logged, so check `tigerastatus`, not `ausearch`.
- **NetworkManager:** on hosts that run it, configure it to ignore `cali*` and `vxlan.calico` interfaces.
- **Finish:** wait for all nodes to be `Ready` before the role ends. Everything after it depends on that.
- **Reset:** `reset.yml` doesn't know about Calico. `cluster-reset.yml` must also clear `/etc/cni/net.d/*calico*`, `/var/lib/calico`, and Calico's interfaces and iptables rules.

## Longhorn

- **Install:** a `HelmChart` with the chart's defaults, after the CNI, in `addons.yml`. The one exception is `persistence.defaultClass: false`: Longhorn is a special case for distributed storage, so `local-path` stays the default class. The K3s docs pin an old `longhorn.yaml`, so check the version against `k3s_version` rather than copying theirs.
- **Hosts:** every node needs `iscsid` running with `iscsi_tcp` loaded. On Fedora, starting `iscsid` also generates `/etc/iscsi/initiatorname.iscsi`.
- **Testing:** a `Bound` PVC or a running pod doesn't prove the data path. Write data, then compare checksums after moving the pod and after a replica rebuild, and treat an empty checksum as a failure.
- **SELinux:** Fedora 44 with `--selinux` needed no workaround. Only add the `iscsid_t` `dac_override` policy from Longhorn's KB if the denial actually appears.

## Argo CD

- **Ordering:** run it after Calico, in `addons.yml`, so pods have a network.
- **Install:** use the `argo-cd` chart through a `HelmChart`, with `targetNamespace: argocd` and `createNamespace: true`.
- **Root app:** wait for CRD `applications.argoproj.io` to be `Established` before applying the root `Application`.
- **Ownership boundary:** Ansible owns the nodes, K3s, Calico, Argo CD and one root `Application`. Argo CD owns everything else. Don't add workloads to this repo, and don't have Argo CD manage what the `HelmChart` CRs manage.
- **Secrets:** repo credentials come from ansible-vault variables, are written as `argocd.argoproj.io/secret-type: repository` Secrets, and any task handling them uses `no_log: true`. Never commit plaintext.
- **Ingress:** if Argo CD will manage ingress or the load balancer, disable Traefik or ServiceLB with `disable:` in `server_config_yaml`.

## Conventions

- Prefix new variables with the role name, for example `addon_calico_version`. Upstream's unprefixed variables are legacy, marked with `# noqa var-naming[no-role-prefix]`. Don't copy that in new roles.
- Use FQCN modules (`ansible.builtin.*`), and give every task a `name`.
- Start YAML files with `---`, use only `true`/`false` for booleans, and quote file modes (`"0600"`). `.yamllint` forbids octal values.
- Make tasks idempotent and set `changed_when` on commands. Waits use `until`/`retries`/`delay` with `changed_when: false`.
- Keep check mode working: guard tasks that can't run in it with `when: not ansible_check_mode`.
- Each add-on has an enable flag, for example `addon_calico_enabled`, defaulting to `false` so that `cluster.yml` on a vanilla inventory matches `site.yml`.
- Don't add collection dependencies. That would mean editing `galaxy.yml` and `collections/requirements.yml`.
- Airgap isn't supported for add-ons yet, because `HelmChart` pulls from the network. Raise it with the user if an airgapped cluster needs add-ons.
- Pin actions by commit SHA in new workflows, as the existing ones do.

## Commands

```bash
# Tooling, same as CI. Not installed by default on this machine.
pip install ansible ansible-lint yamllint netaddr
ansible-galaxy collection install -r collections/requirements.yml

# Required before calling a change done
yamllint .
ansible-lint

# Run
ansible-playbook playbooks/cluster.yml -i inventory.yml   # new cluster
ansible-playbook playbooks/addons.yml -i inventory.yml    # add-ons only
```

`inventory.yml` is gitignored, so it can hold real hosts. Vault-encrypt any secrets in it anyway.

`tests/*.yml` are inventories for CI's privileged Docker nodes, not unit tests. They can check install order and role logic for Calico, if you run `mount --make-rshared /` in each container first, but not real networking: test that on real nodes. For Vagrant, use a separate file through `VAGRANT_VAGRANTFILE` instead of editing `Vagrantfile`, whose `extra_vars` are hardcoded.

Don't run playbooks against a real inventory without being asked, especially `reset.yml` or `cluster-reset.yml`, which destroy the cluster.

## Syncing and git

Upstream isn't configured as a remote yet:

```bash
git remote add upstream https://github.com/k3s-io/k3s-ansible.git
git fetch upstream
git merge upstream/main   # merge, don't rebase: main is published
```

After a sync, re-read the "Behaviour of the upstream code" section against what changed.

By default this fork only pulls from upstream. All commits, branches and PRs go to `jrhoward/k3s-ansible`. The user may contribute back later, but don't open PRs or issues on `k3s-io/k3s-ansible`, or push to the `upstream` remote, without their explicit instruction.

Pass the repo explicitly when opening a PR, because `gh` can default to the parent repo: `gh pr create --repo jrhoward/k3s-ansible`.

## Upstream deviations

None.
