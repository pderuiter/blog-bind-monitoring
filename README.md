# blog-bind-monitoring

Companion code for the blog post
[Monitoring BIND DNS with Prometheus and Grafana](https://blog.bsdserver.nl/monitoring-bind-dns-with-prometheus-and-grafana/).

It builds the post's monitoring setup on one DNS server:

- BIND with the statistics channel on 127.0.0.1:8053 and [::1]:8053 (`allow { localhost; }`),
  `zone-statistics full`, and the post's query and security logging (`querylog yes`, so it survives
  `rndc reconfig`)
- bind_exporter v0.7.0 on port 9119, either
  - in the Docker Compose stack (default), with `network_mode: host` so it can reach the localhost-only
    statistics channel, or
  - from the release tarball with the post's systemd unit (`bind_exporter_method: systemd`)
- the Docker Compose stack from the post: Prometheus (with the post's two alerting rule files),
  Alertmanager (minimal configuration) and Grafana (with a provisioned Prometheus datasource).
  Prometheus reaches the exporter on the host through `host.docker.internal` (`host-gateway`).

Pick one exporter method: both listen on port 9119.

## Requirements

- Terraform 1.6+ and a Proxmox VE cluster with an API token (only if you want Terraform to create the VM)
- Ansible core 2.16+ on your workstation
- A target running Ubuntu 24.04 (tested), with 4 GB RAM for the full stack
- Internet access from the server (Ubuntu packages, container images or the exporter tarball)

## Quick start

```bash
# 1. Create the VM (optional, skip if you already have a server)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Set the Grafana admin password
cd ../ansible
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
ansible-vault encrypt group_vars/all/vault.yml   # after setting the password

# 3. Configure BIND and deploy the monitoring stack
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --ask-vault-pass

# 4. Run the checks (statistics channel, bind_up, Prometheus targets and rules, Alertmanager, Grafana)
ansible-playbook ../tests/verify.yml --ask-vault-pass
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and edit it.

Prometheus is on port 9090, Alertmanager on 9093 and Grafana on 3000 (user `admin`). Import dashboard
12309 in Grafana if you want the community dashboard; see the post for the `instance` label caveat.

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `bind_exporter_method` | `compose` | `compose` (exporter in the stack) or `systemd` (tarball + systemd unit) |
| `bind_exporter_version` | `0.7.0` | Exporter release for the systemd method |
| `bind_statistics_port` | `8053` | Statistics channel port |
| `bind_statistics_querylog` | `true` | Write `querylog yes;` |
| `monitoring_stack_enabled` | `true` | Deploy Prometheus, Alertmanager and Grafana with Docker Compose |
| `monitoring_stack_dir` | `/opt/bind-monitoring` | Compose project directory |
| `monitoring_stack_instance` | inventory hostname | `instance` label of the BIND target |
| `monitoring_stack_*_image` | pinned tags | Prometheus v3.16.0, Alertmanager v0.33.1, Grafana 13.3.1, bind-exporter v0.7.0 |
| `vault_monitoring_stack_grafana_admin_password` | (vault) | Grafana admin password, written to the compose `.env` file |

## Layout

- `terraform/` creates Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider) and outputs an Ansible inventory
- `ansible/roles/bind_install` installs BIND
- `ansible/roles/bind_statistics` configures the statistics channel and logging
- `ansible/roles/bind_exporter` installs the exporter with systemd (when selected)
- `ansible/roles/monitoring_stack` installs Docker and deploys the compose stack
- `tests/verify.yml` runs the functional checks

## License

MIT
