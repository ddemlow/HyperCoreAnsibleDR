# CLAUDE.md

This file provides guidance to Claude Code when working in this repository.

## What this project is

Ansible runbook suite for Scale Computing HyperCore DR failover and failback.
All playbooks run against live HyperCore clusters via the `scale_computing.hypercore`
Ansible collection. Operator-triggered; not an autonomous system.

## Running playbooks

```bash
# Install collection
ansible-galaxy collection install -r requirements.yml

# Failover (emergency — source unreachable)
ansible-playbook automated_dr_failover.yml -i inventory/inventory.yml

# Failover (planned — graceful, e.g. ahead of a storm)
ansible-playbook automated_dr_failover.yml -i inventory/inventory.yml \
  -e "planned_failover=true"   # (future flag — see docs/FAILBACK_DESIGN.md)

# Failback — three sequential playbooks
ansible-playbook automated_dr_failback_prepare.yml -i inventory/inventory.yml
ansible-playbook automated_dr_failback_cutover.yml -i inventory/inventory.yml
ansible-playbook automated_dr_failback_cleanup.yml -i inventory/inventory.yml

# Live integration tests (requires two live clusters)
ansible-playbook tests/test_failover_live.yml -i inventory/test_inventory.yml

# Logic unit tests (no cluster needed)
ansible-playbook tests/test_failover_logic.yml -i inventory/inventory.yml
```

## Repository layout

```
automated_dr_failover.yml          Emergency failover runbook (clone replicated VMs on DR)
automated_dr_failback_prepare.yml  Failback Phase 1-3: conflict check, reverse replication, sync
automated_dr_failback_cutover.yml  Failback Phase 4: shutdown, final snapshot, clone, power on
automated_dr_failback_cleanup.yml  Failback Phase 5: delete DR VMs and stubs (irreversible)

inventory/
  inventory.example.yml            Template for production inventory (two groups: source, dr)
  test_inventory.example.yml       Template for test inventory

tests/
  test_failover_live.yml           End-to-end live integration test with summary
  test_failover_logic.yml          Unit tests: name regex, filter logic (no cluster needed)

docs/
  FAILBACK_DESIGN.md               Full failback design: phases, API limitations, error map
  GETTING_STARTED.md               Step-by-step cluster setup guide
  TROUBLESHOOTING.md
```

## Inventory structure

Two inventory groups are required by the failback playbooks:

```yaml
# inventory/inventory.yml
all:
  children:
    source:          # primary (production) cluster
      hosts:
        10.0.0.1:
          scale_user: admin
          scale_pass: admin
    dr:              # disaster recovery cluster
      hosts:
        10.0.0.2:
          scale_user: admin
          scale_pass: admin
```

The failover playbook (`automated_dr_failover.yml`) runs on `hosts: all` and uses
`SC_HOST: "https://{{ inventory_hostname }}"` — it targets the DR cluster by design.

## Key Ansible collection modules

Use `scale_computing.hypercore` wherever possible. Direct REST via `ansible.builtin.uri`
is appropriate only for:
- Progress polling (`GET /rest/v1/VirDomainReplication`)
- VM rename (`PATCH /rest/v1/VirDomain/<uuid>` — no collection module exists)
- Cross-cluster queries from within a play (target a different host than `SC_HOST`)

```yaml
scale_computing.hypercore.vm_info            # list VMs
scale_computing.hypercore.vm_clone           # clone with preserve_mac_address: true
scale_computing.hypercore.vm_params          # power state changes
scale_computing.hypercore.vm_replication     # configure/disable replication
scale_computing.hypercore.remote_cluster_info  # verify ESTABLISHED, resolve cluster name
scale_computing.hypercore.vm_snapshot_info   # list snapshots (no type filter — use count delta)
scale_computing.hypercore.vm                 # create/delete VMs
```

## HyperCore domain knowledge

### Tags are a list, not a string

Critical: `vm_info` returns `tags` as a **list**, not a string. Always use:
```yaml
# CORRECT:
| selectattr('tags', 'contains', dr_vm_tag)

# WRONG (silently drops every VM):
| selectattr('tags', 'string')
| selectattr('tags', 'search', dr_vm_tag)
```

### DR VM naming convention

Failover clones are named: `<original-name>-YYYY-MM-DD_HHMM-DR`

Strip regex (used everywhere):
```yaml
dr_name_suffix_regex: '^(.*)-\d{4}-\d{2}-\d{2}_\d{4}-DR$'
```

In Jinja2: `{{ vm.vm_name | regex_replace('^(.*)-\\d{4}-\\d{2}-\\d{2}_\\d{4}-DR$', '\\1') }}`

### Replication progress polling

No collection module for replication progress. Use URI:
```yaml
ansible.builtin.uri:
  url: "https://{{ inventory_hostname }}/rest/v1/VirDomainReplication"
  method: GET
  url_username: "{{ scale_user | default('admin') }}"
  url_password: "{{ scale_pass | default('admin') }}"
  validate_certs: false
# Extract: result.json | selectattr('virDomainUUID', '==', vm_uuid) | map(attribute='progress.percentComplete')
```

### Shutdown snapshot detection

HyperCore takes a SHUTDOWN snapshot after graceful power-off, but `vm_snapshot_info`
has no type filter. Pattern: record snapshot count before shutdown, poll until count
increases. The new snapshot is the shutdown snapshot.

### VM rename

No collection module. Use:
```yaml
ansible.builtin.uri:
  url: "https://{{ inventory_hostname }}/rest/v1/VirDomain/{{ vm_uuid }}"
  method: PATCH
  body_format: json
  body: {"name": "new-name"}
```

### `vm_replication` requires cluster name, not IP

Resolve at runtime via `remote_cluster_info`. Never hardcode.

### Idempotency

The failover playbook is idempotent: checks for VMs tagged `DR-failover` before
cloning. If any exist, it skips the clone phase and exits with a message.
Re-running after failover is safe and expected.

## Operational model

These are **human-triggered runbooks**, not autonomous systems. See README.md and
`docs/GETTING_STARTED.md` for the distinction and the requirements for scheduled/
autonomous operation (which requires additional split-brain fencing and alerting).

### Two failover modes (both use the same playbook)

**Emergency failover** — source is unreachable, failover triggered by operator or cron:
- Playbook verifies source is DISCONNECTED (exits if ESTABLISHED)
- Pings source nodes (exits if any respond — split-brain guard)
- Clones replication targets and powers on

**Planned failover** — operator-initiated ahead of a planned outage (e.g. storm, maintenance):
- Same clone + power-on logic but the operator pre-shuts down source VMs first
- Final shutdown snapshot replicates → clone from zero-RPO state
- Future: `planned_failover=true` flag to skip disconnect/ping checks

The failback playbook suite (`prepare → cutover → cleanup`) serves as the
structured return path after **either** emergency or planned failover.

## Error handling conventions

- Use `block/rescue` for VM-level operations that should not abort the entire play
- Hard-fail (`ansible.builtin.fail`) on safety gates: replication not at 100%, source
  VMs not RUNNING when cleanup is about to delete DR VMs, etc.
- `ignore_errors: true` only for non-critical post-cutover operations (e.g. replication
  restore after source VMs are already running)

## PR process

Before merging any branch:
1. Run `tests/test_failover_live.yml` against a test cluster and confirm PASS
2. Run `/review-pr` in Claude Code for a structured review
3. Merge after addressing any CHANGES REQUESTED items
