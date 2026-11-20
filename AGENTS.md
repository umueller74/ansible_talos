# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## What this is

Ansible playbooks and roles that set up a Talos Linux Kubernetes cluster (homelab). Talos nodes have no SSH, so nothing runs *on* the nodes: every play runs on the controller (`localhost` or `delegate_to: localhost`) and talks to the nodes via `talosctl` over the Talos API (port 50000).

## Commands

There is no `ansible.cfg`, so always pass the inventory explicitly. Run from the repo root (`talos_output_dir` is based on `playbook_dir`).

```sh
ansible-galaxy collection install -r requirements.yml               # once (ansible.windows, used by talos_client)
ansible-playbook -i inventory/hosts.yml setup_cluster.yml           # full pipeline
ansible-playbook -i inventory/hosts.yml generate_machineconfig.yml  # single step
ansible-playbook -i inventory/hosts.yml configure_workers.yml --limit 192.168.0.241
ansible-playbook -i inventory/hosts.yml setup_cluster.yml --syntax-check
ansible-lint                                                          # homelab/ is excluded via .ansible-lint
```

To test config generation without touching `homelab/`, copy `homelab/{controlplane,worker}.yaml,talosconfig` to a scratch dir and run `generate_machineconfig.yml -e talos_output_dir=<dir>`. It needs only a local `talosctl`, no nodes.

`.venv` is a symlink to `../homelab/.venv`, which may not exist. `ansible-playbook` may not be on PATH outside that venv. The `roles/*/tests/` dirs are untouched `ansible-galaxy init` boilerplate, so there is no real test suite.

## Pipeline (`setup_cluster.yml` imports these in order)

Plays that target Talos nodes use `connection: local`: every task runs on the controller with `inventory_hostname` as the node IP.

1. `install_talosctl.yml` → role `talos_client`: downloads the `talosctl` binary from GitHub releases for Linux, macOS or Windows (needs `become`). The version defaults to `talos_version`.
2. `generate_machineconfig.yml` (hosts `talos`) → role `talos_machineconfig`:
   - `talosctl gen config` into `talos_output_dir` (default `./<talos_cluster_name>/`, i.e. `homelab/`), `run_once` and guarded by `creates:`, so cluster secrets are generated only once. `controlplane.yaml` and `worker.yaml` stay unmodified base configs.
   - Per node: renders `templates/node-patch.yaml.j2` to `patches/<ip>.yaml` (install disk, static IP via `deviceSelector: {physical: true}`, default route, nameservers, NTP, `allowSchedulingOnControlPlanes` for controlplanes, and a `HostnameConfig` document with `auto: "off"` and `hostname[.domain]`), then `talosctl machineconfig patch` builds `nodes/<ip>.yaml` and `talosctl validate --mode metal` checks it.
3. `configure_controlplane.yml` / `configure_workers.yml` → `talosctl apply-config --insecure --file nodes/<ip>.yaml`.
4. `wait_for_controlplane.yml`: retries `talosctl version` against the first controlplane with the generated talosconfig until it answers, i.e. until the node has installed and rebooted (maintenance mode does not accept it).
5. `bootstrap_cluster.yml` → role `talos_bootstrap`: `talosctl bootstrap` against the first controlplane with explicit `--talosconfig`/`--endpoints`/`--nodes`; an `AlreadyExists` error counts as success.

Roles `talos_common`, `talos_kubeconfig` and `talos_upgrade` are empty scaffolds. No playbook uses them, and `talos_common` is not a dependency of anything; its defaults are only a reference copy.

## Configuration

- `inventory/hosts.yml`: groups `talos_controlplane` and `talos_worker` under `talos`. Each host is keyed by IP and carries `hostname`, `disk`, `network.ip` (CIDR) and `network.gateway`; all are used in that node's patch.
- `inventory/group_vars/all.yaml`: cluster name, Talos and Kubernetes versions, endpoints, `dns`, `domain`, `ntp_servers`. Group vars override the role defaults (the implicit `localhost` also gets them).
- The first controlplane (`groups['talos_controlplane'][0]`) is the cluster endpoint and the bootstrap target.
- `HostnameConfig` cannot have both `auto` and `hostname` set; `talosctl validate` rejects it. The patch must keep `auto: "off"`.
- Talos uses predictable interface names, so don't patch by `interface: eth0`. The patch uses `deviceSelector: {physical: true}`, which assumes one physical NIC per node.

## Secrets

`homelab/` (`talosconfig`, `controlplane.yaml`, `worker.yaml`) contains live cluster PKI material and tokens, and it is committed to git. Don't print its contents, and don't add more generated secrets to commits. The generated `*/nodes/` and `*/patches/` dirs are git-ignored.
