# ansible_talos

Set up a [Talos Linux](https://www.talos.dev/) Kubernetes cluster with Ansible.

Talos nodes have no SSH and no shell. All plays therefore run on the Ansible controller and drive the nodes through `talosctl` over the Talos API (TCP 50000).

## Requirements

- Ansible on the controller (Linux, macOS or Windows for the `talosctl` install step)
- Nodes booted from the Talos ISO/image and reachable in maintenance mode
- Network access from the controller to the nodes on port 50000 and to the first controlplane on port 6443

`install_talosctl.yml` downloads `talosctl` itself, so it does not need to be installed beforehand.

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

The first host in `talos_controlplane` is the cluster endpoint and the bootstrap target.

### Cluster variables

Set these in `inventory/group_vars/all.yaml`:

| Variable | Description |
|---|---|
| `talos_cluster_name` | Cluster name; also the name of the output directory for generated configs |
| `talos_version` / `kubernetes_version` | Talos and Kubernetes versions |
| `talos_cluster_endpoint` | Kubernetes API endpoint, default `https://<first controlplane>:6443` |
| `talos_output_dir` | Where generated configs go, default `./<talos_cluster_name>` |
| `dns` | Nameservers written into the controlplane config |
| `ntp_servers` | NTP servers written into the controlplane config |
| `controlplane_is_worker` | Allow workloads on controlplane nodes (default `false`) |

The `talosctl` version is set separately in `roles/talos_client/vars/main.yml` (`talosctl_version`). Keep it in sync with `talos_version`.

## Usage

Run the whole setup:

```sh
ansible-playbook -i inventory/hosts.yml setup_cluster.yml
```

Or run the steps one by one, in this order:

| Playbook | What it does |
|---|---|
| `install_talosctl.yml` | Downloads `talosctl` to the controller (needs sudo) |
| `generate_machineconfig.yml` | Runs `talosctl gen config` once, then sets install disk, static IP, gateway, DNS and NTP in `controlplane.yaml` (a timestamped backup is kept) |
| `configure_controlplane.yml` | Applies `controlplane.yaml` to the controlplane nodes |
| `wait_for_controlplane.yml` | Waits until the Talos API on the first controlplane is back |
| `bootstrap_cluster.yml` | Bootstraps etcd on the first controlplane |
| `configure_workers.yml` | Applies `worker.yaml` to the worker nodes |

Configs are only generated if `<talos_output_dir>/controlplane.yaml` does not exist yet. Delete the directory to start over with new cluster secrets.

## Generated files and secrets

`talosctl gen config` writes `controlplane.yaml`, `worker.yaml` and `talosconfig` to `talos_output_dir`. These files contain the cluster CA keys and join tokens. Keep them out of public repositories, or encrypt them (for example with Ansible Vault or SOPS).

## Status

Work in progress. Some playbooks still have bugs, and the `talos_kubeconfig`, `talos_upgrade` and `talos_common` roles are empty placeholders.

## License

GPL-3.0. See [LICENSE](LICENSE).
