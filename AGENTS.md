# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## What this is

Ansible playbooks and roles that set up a Talos Linux Kubernetes cluster (homelab). Talos nodes have no SSH, so nothing runs *on* the nodes: every play runs on the controller (`localhost` or `delegate_to: localhost`) and talks to the nodes via `talosctl` over the Talos API (port 50000).

## Commands

There is no `ansible.cfg`, so always pass the inventory explicitly. Run from the repo root (`talos_output_dir` is based on `playbook_dir`).

```sh
ansible-playbook -i inventory/hosts.yml setup_cluster.yml          # full pipeline
ansible-playbook -i inventory/hosts.yml generate_machineconfig.yml # single step
ansible-playbook -i inventory/hosts.yml configure_workers.yml --limit 192.168.0.241
ansible-lint                                                         # lint
```

`.venv` is a symlink to `../homelab/.venv`, which may not exist. `ansible-playbook` may not be on PATH outside that venv. The `roles/*/tests/` dirs are untouched `ansible-galaxy init` boilerplate, so there is no real test suite.

## Pipeline (`setup_cluster.yml` imports these in order)

1. `install_talosctl.yml` → role `talos_client`: downloads the `talosctl` binary from GitHub releases for Linux, macOS or Windows (needs `become`).
2. `generate_machineconfig.yml` → role `talos_machineconfig`: runs `talosctl gen config` into `talos_output_dir` (default `./<talos_cluster_name>/`, i.e. `homelab/`). This step is guarded by `creates:`, so it only runs once. It then backs up `controlplane.yaml` with a timestamp suffix, loads it with `from_yaml_all`, patches only the **first** YAML document (install disk, nameservers, NTP, static `eth0` IP and default route, `allowSchedulingOnControlPlanes` from `controlplane_is_worker`) using values from the first controlplane host, and writes it back.
3. `configure_controlplane.yml` / `configure_workers.yml` → `talosctl apply-config --insecure` with `controlplane.yaml` / `worker.yaml`.
4. `wait_for_controlplane.yml`: waits for port 50000 on the first controlplane.
5. `bootstrap_cluster.yml` → role `talos_bootstrap`: `talosctl bootstrap` against the first controlplane (`run_once`).

Roles `talos_common`, `talos_kubeconfig` and `talos_upgrade` are empty scaffolds. No playbook uses them, and `talos_common` is not a dependency of anything; its defaults are only a reference copy. Nothing uses the large static files in `roles/talos_{controlplane,worker}/templates/` either; they are copies of generated configs and contain no Jinja.

## Configuration

- `inventory/hosts.yml`: groups `talos_controlplane` and `talos_worker` under `talos`. Each host is keyed by IP and carries `hostname`, `disk`, `network.ip` (CIDR) and `network.gateway`.
- `inventory/group_vars/all.yaml`: cluster name, Talos and Kubernetes versions, endpoints, `dns`, `domain`, `ntp_servers`. Group vars override the role defaults. `talos_client` pins its own `talosctl_version` in `vars/main.yml`, which takes high precedence. Keep it in sync with `talos_version`.
- The "first controlplane" (`groups['talos_controlplane'][0]`) is the cluster endpoint, the bootstrap target and the source of the network values in the patch step. Worker configs are **not** patched; `worker.yaml` is used exactly as generated.

## Known defects (as of the current state, worth fixing when touching these files)

- `talos_controlplane_vip` in `group_vars/all.yaml` and `talos_common/defaults` contains `][0]] }}`, which is invalid Jinja and fails as soon as the variable is evaluated.
- `configure_workers.yml` targets `hosts: talos_workers`, but the inventory group is `talos_worker`, so the play matches no hosts.
- `install_talosctl.yml` loads `vars_files: vars/talos.yml`, which does not exist.
- In `wait_for_controlplane.yml`, `host: groups['talos_controlplane'][0]` is a literal string; it is not wrapped in `{{ }}`.
- The apply-config tasks pass `--talosconfig=./{{ talos_output_dir }}/talosconfig`, but `talos_output_dir` is already an absolute path.
- `talos_bootstrap` calls `talosctl bootstrap` without `--talosconfig` or `--endpoints`, so it depends on the user's default talosconfig.

## Secrets

`homelab/` (`talosconfig`, `controlplane.yaml*`, `worker.yaml`) and the role templates contain live cluster PKI material and tokens, and they are committed to git. Don't print their contents, and don't add more generated secrets to commits.
