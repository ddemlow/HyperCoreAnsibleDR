# hypercore-ansible-dr-failover

Ansible playbook for automated disaster recovery failover on Scale Computing HyperCore.

When run on a schedule, the playbook monitors the replication connection from a DR (target)
cluster back to the primary (source) cluster. If the primary is confirmed unreachable, it
automatically clones all replicated VMs on the DR cluster and powers them on — with no
manual intervention required.

## How it works

The playbook runs entirely against the **DR (target) cluster** and proceeds through six phases:

1. **Replication link check** — if all monitored source clusters are `ESTABLISHED`, exit immediately (no action needed)
2. **Primary node ping** — dynamically discovers source cluster node IPs from the replication connection record and pings each one; if any node responds, exit (primary is still reachable)
3. **Optional application-alive check** — if `http_check_host` is configured and responds on the specified port, exit (application is still running at the primary site)
4. **DR cluster health check** — pings all local DR node IPs to confirm the DR cluster is healthy before attempting failover
5. **Idempotency check** — if VMs tagged with `dr_vm_tag` already exist, exit (failover already ran)
6. **Failover** — clones all VMs that have a `replication_source_vm_uuid` (i.e., are replication targets), preserving MAC addresses, then powers them on

## Requirements

- Scale Computing HyperCore cluster running ICOS v9.4 or later
- Replication configured from the primary cluster to the DR cluster
- Ansible `ansible-core >= 2.16`
- `scale_computing.hypercore` collection >= 1.3.0 (required for `replication_source_vm_uuid` in `vm_info`)
- The Ansible controller must be able to reach the DR cluster API and ping the primary cluster node IPs

## Install

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

## Configure

### Inventory

Create an inventory file with your DR cluster(s) as hosts. The playbook connects to the
HyperCore API using `https://{{ inventory_hostname }}`, so use the cluster IP or FQDN as
the host name.

```
cp inventory/inventory.example.yml inventory/inventory.yml
# edit inventory/inventory.yml
```

For credentials, set `scale_user` and `scale_pass` per host or group, or export them as
environment variables:

```
export SC_USERNAME=admin
export SC_PASSWORD=your_password
```

Using Ansible Vault for passwords is recommended for production use:

```
ansible-vault encrypt_string 'your_password' --name 'scale_pass'
```

### Variables

All variables have defaults and can be overridden in inventory group_vars, host_vars, or
passed at runtime with `-e`.

| Variable | Default | Description |
|---|---|---|
| `scale_user` | `admin` | HyperCore API username |
| `scale_pass` | `admin` | HyperCore API password |
| `source_cluster_name` | `""` | Monitor only this named remote cluster. Empty = monitor all remote clusters. |
| `ping_retries` | `3` | Times to retry pinging a primary node before declaring it down |
| `ping_delay` | `5` | Seconds between ping retries |
| `dr_vm_tag` | `DR-failover` | Tag applied to all cloned failover VMs (also used for idempotency) |
| `dr_exclude_tag` | `""` | VMs with this tag are skipped during failover. Useful for infrastructure VMs (DNS, NTP, monitoring) that should not be auto-cloned. |
| `http_check_host` | `""` | Optional application-alive check. If this host responds on `http_check_port`, failover is aborted. |
| `http_check_port` | `80` | TCP port for the application-alive check |

### Multiple source clusters

If multiple source clusters replicate to the same DR cluster, the playbook monitors all of
them by default (or a specific named one if `source_cluster_name` is set). Node IPs from
all disconnected clusters are pinged; if any respond, failover is aborted.

> **Note:** The HyperCore API on the target cluster does not expose which source cluster a
> replicated VM originated from. If multiple source clusters are in use and only one goes
> down, all replicated VMs (from all clusters) will be failed over. Tag source VMs on the
> primary with the cluster name before replication if you need per-cluster VM selection.

## Run

### Manual run

```
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml
```

Override variables at runtime:

```
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  -e "source_cluster_name=primary-site dr_exclude_tag=dr-exclude ping_retries=5"
```

Limit to a specific DR cluster if inventory has multiple:

```
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  -l dr-cluster-charlotte
```

### Scheduled execution (cron)

Run every 5 minutes via cron on the Ansible controller:

```
*/5 * * * * cd /opt/hypercore-ansible-dr && .venv/bin/ansible-playbook \
  -i inventory/inventory.yml automated_dr_failover.yml >> /var/log/dr-failover.log 2>&1
```

### Scheduled execution (AWX / Ansible Automation Platform)

Create a Job Template pointing to this playbook. Set the schedule to run every 5 minutes.
The playbook is idempotent — if failover has already occurred, it exits cleanly on the
idempotency check without making any changes.

## Excluding VMs from failover

Set `dr_exclude_tag` to a tag name and apply that tag to any VMs on the **DR cluster** that
should not be auto-cloned during failover (e.g., shared infrastructure that runs at both
sites):

```
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  -e "dr_exclude_tag=dr-exclude"
```

## Cleaning up after failover / failback

The playbook is intentionally limited to the failover direction. After the primary site is
restored:

1. Power off and delete the DR failover VMs (those tagged `dr_vm_tag`)
2. Verify replication has resumed and is current
3. The next scheduled run will exit cleanly (primary is `ESTABLISHED` again)

Manual failback (returning DR VMs to the primary) is not automated in this playbook.
See [Known limitations](#known-limitations).

## Known limitations

- **VM startup ordering** — VMs are powered on in arbitrary order. Applications that require
  a specific boot sequence (database before application before web tier) must be started
  manually after failover. Ordered startup is planned as a future enhancement.

- **VM-to-source-cluster mapping** — When multiple source clusters replicate to the same DR
  cluster, there is no API-level way to determine which source cluster a replicated VM came
  from. All replicated VMs are failed over when any source cluster is confirmed down. Tag
  source VMs on the primary with their cluster name before replication to work around this.

- **Graceful failover** — The playbook clones from the most recent replication snapshot.
  Any writes to the source VMs since the last snapshot will be lost. A graceful failover
  (shut down source, wait for final snapshot to replicate, then clone) is not currently
  automated.

- **Failback** — Returning failed-over VMs to the primary site after it is restored is not
  automated. Manual steps are required.

- **Post-failover health checks** — The playbook powers on cloned VMs but does not verify
  that applications are responding. Adding per-VM TCP health checks after power-on is a
  planned enhancement.

## Repository contents

```
automated_dr_failover.yml   # main playbook
draft_dr_exploration.yml    # original prototype / reference
inventory/
  inventory.example.yml     # example inventory template
requirements.yml            # Ansible collection dependencies
ansible.cfg                 # Ansible defaults for this project
```

## License

GNU General Public License v3.0 — see [LICENSE](LICENSE).
