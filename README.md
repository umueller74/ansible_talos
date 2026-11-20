# ansible_talos

Set up a [Talos Linux](https://www.talos.dev/) Kubernetes cluster with Ansible.

Talos nodes have no SSH and no shell. All plays therefore run on the Ansible controller and drive the nodes through `talosctl` over the Talos API (TCP 50000).

## Requirements

- Ansible on the controller (Linux, macOS or Windows for the `talosctl` install step)
- Nodes booted from the Talos ISO/image and reachable in maintenance mode
- Network access from the controller to the nodes on port 50000 and to the first controlplane on port 6443

`install_talosctl.yml` downloads `talosctl` itself, so it does not need to be installed beforehand.

Install the required collection once:

```sh
ansible-galaxy collection install -r requirements.yml
```

## Configuration

### Inventory

Define the nodes in `inventory/hosts.yml`. Each host is keyed by its IP:

```yaml
talos:
  children:
    talos_controlplane:
      hosts:
        192.168.0.240:
          hostname: k8s-master
          disk: /dev/sda
          network:
            ip: 192.168.0.240/22
            gateway: 192.168.0.1
    talos_worker:
      hosts:
        192.168.0.241:
          hostname: k8s-node01
          disk: /dev/sda
          network:
            ip: 192.168.0.241/22
            gateway: 192.168.0.1
```

Every node gets its own config with its `hostname`, `disk`, IP and gateway. The first host in `talos_controlplane` is the cluster endpoint and the bootstrap target.

### Cluster variables

Set these in `inventory/group_vars/all.yaml`:

| Variable | Description |
|---|---|
| `talos_cluster_name` | Cluster name; also the name of the output directory for generated configs |
| `talos_version` / `kubernetes_version` | Talos and Kubernetes versions |
| `talos_cluster_endpoint` | Kubernetes API endpoint, default `https://<first controlplane>:6443` |
| `talos_output_dir` | Where generated configs go, default `./<talos_cluster_name>` |
| `dns` | Nameservers for all nodes |
| `domain` | Appended to each node's `hostname` (optional) |
| `ntp_servers` | NTP servers for all nodes |
| `controlplane_is_worker` | Allow workloads on controlplane nodes (default `false`) |

The installed `talosctl` version follows `talos_version` (override with `talosctl_version`).

## Usage

Run the whole setup:

```sh
ansible-playbook -i inventory/hosts.yml setup_cluster.yml
```

Or run the steps one by one, in this order:

| Playbook | What it does |
|---|---|
| `install_talosctl.yml` | Downloads `talosctl` to the controller (needs sudo) |
| `generate_machineconfig.yml` | Runs `talosctl gen config` once, then builds and validates one config per node in `<talos_output_dir>/nodes/` (hostname, install disk, static IP, gateway, DNS, NTP) |
| `configure_controlplane.yml` | Applies each controlplane node's config |
| `wait_for_controlplane.yml` | Waits until the first controlplane has installed and answers with the cluster `talosconfig` |
| `bootstrap_cluster.yml` | Bootstraps etcd on the first controlplane |
| `configure_workers.yml` | Applies each worker node's config |

Cluster secrets are only generated if `<talos_output_dir>/controlplane.yaml` does not exist yet; delete the directory to start over with new ones. The per-node configs are rebuilt on every run, so inventory changes are picked up.

## Generated files and secrets

`talosctl gen config` writes `controlplane.yaml`, `worker.yaml` and `talosconfig` to `talos_output_dir`; the per-node configs go to `nodes/` (git-ignored). These files contain the cluster CA keys and join tokens. Keep them out of public repositories, or encrypt them (for example with Ansible Vault or SOPS).

## Status

Work in progress. The `talos_kubeconfig`, `talos_upgrade` and `talos_common` roles are empty placeholders. Node networking assumes one physical NIC per node.

## License

GPL-3.0. See [LICENSE](LICENSE).
