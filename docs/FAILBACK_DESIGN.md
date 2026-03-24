# DR Failback Design

End-to-end plan for the `feature/automated-dr-failback` branch.

---

## Overview

Failback is the reverse of failover: returning production workloads from the DR cluster
back to the restored primary (source) cluster. It is deliberately split into three separate
playbooks to give the operator control over each stage and to prevent accidental data loss.

```
automated_dr_failback_prepare.yml   Phase 1-3: conflict check, reverse replication, sync
automated_dr_failback_cutover.yml   Phase 4:   shutdown DR VMs, wait for final snapshot,
                                               clone on source, power on, restore replication
automated_dr_failback_cleanup.yml   Phase 5:   delete DR VMs and source stubs (manual, last)
tests/test_failback_live.yml        Live integration test for all phases
```

Operator flow:
```
1. ansible-playbook automated_dr_failback_prepare.yml -i inventory/inventory.yml
   (re-runnable; waits until all DR VMs replicate to source at 100% — may take hours)

2. ansible-playbook automated_dr_failback_cutover.yml -i inventory/inventory.yml
   (requires operator presence; service outage starts here; explicit YES confirmation)

3. [operator verifies applications on source are healthy]

4. ansible-playbook automated_dr_failback_cleanup.yml -i inventory/inventory.yml
   (permanent deletion — only after step 3 is confirmed; requires typing DELETE)
```

---

## Planned vs. Emergency Failover — Shared Failback Path

The failback playbook suite works identically whether the preceding failover was
**emergency** (source unreachable, cloned from last periodic snapshot) or
**planned** (operator-initiated ahead of a storm or maintenance window, source shut
down cleanly and final shutdown snapshot replicated before cloning).

The key difference is the RPO of the DR VMs:

| Failover type | RPO at failover | Notes |
|---|---|---|
| Emergency | Last replication snapshot (minutes–hours old) | Source went down uncontrolled |
| Planned graceful | ~0 (shutdown snapshot fully replicated) | Operator shuts source VMs, waits for 100% |

In both cases, after the DR site has been running for some time, the DR VMs will
have accumulated new writes. Failback must replicate those writes back to source
before cutting over — the same `prepare → cutover → cleanup` sequence applies.

### Planned failover flow (future enhancement to `automated_dr_failover.yml`)

A planned failover needs a `planned_failover=true` mode that:

1. **Skips** the disconnect/ping guard checks (source is intentionally still online)
2. **Gracefully shuts down source VMs** (same pattern as cutover Play 2: graceful → force stop)
3. **Waits for the shutdown snapshot to appear and reach 100% replication** (same URI
   poll as cutover Play 2 steps 4–5)
4. **Then clones** — same as the standard failover clone + power-on

This gives zero-RPO failover for planned events (hurricanes, maintenance, migrations)
using the same DR infrastructure already in place.

Implementation sketch for `automated_dr_failover.yml`:
```yaml
vars:
  planned_failover: false   # set true via -e for operator-initiated graceful failover

# When planned_failover: true:
#   - Skip Phase 1 (cluster disconnect check) and Phase 2 (ping guard)
#   - Add new Phase 1.5: shut down source VMs, wait for shutdown snapshot at 100%
#   - Phase 3+ (clone, power on) proceeds identically
```

The same DR VM naming convention, tags, and idempotency logic apply regardless of mode.
Failback is identical in both cases.

---

## Name Mapping Convention

The failover playbook names DR clones as:
```
<original-name>-YYYY-MM-DD_HHMM-DR
```

Example: `web-01` → `web-01-2024-06-15_1430-DR`

Stripping regex (used in every failback playbook):
```yaml
dr_name_suffix_regex: '^(.*)-\d{4}-\d{2}-\d{2}_\d{4}-DR$'
```

In Jinja2:
```
{{ item.vm_name | regex_replace('^(.*)-\\d{4}-\\d{2}-\\d{2}_\\d{4}-DR$', '\\1') }}
```

This must be unit-tested first in `tests/test_failback_logic.yml` before any live testing.
Edge cases: VM names containing hyphens (`my-web-app` → `my-web-app-2024-06-15_1430-DR`).

---

## Known API / Collection Limitations

**1. No replication progress module.**
Progress polling must use `ansible.builtin.uri` against `GET /rest/v1/VirDomainReplication`
and extract `progress.percentComplete`. Same pattern as `test_failover_live.yml:262-272`.

**2. No snapshot type filter.**
HyperCore takes a "SHUTDOWN" snapshot after graceful VM power-off but `vm_snapshot_info`
does not filter by type. Workaround: record snapshot count before shutdown, then poll until
count increases. The new snapshot is the shutdown snapshot.

**3. `vm_replication` requires a cluster name, not an IP.**
The remote cluster name must be resolved via `remote_cluster_info` at runtime, not
hardcoded. Both clusters have each other as named remote clusters.

**4. VM rename is not in the Ansible collection.**
Renaming conflicting source VMs uses `ansible.builtin.uri` with
`PATCH /rest/v1/VirDomain/<uuid>` body `{"name": "new-name"}`.

**5. Replication target name collision.**
When reverse replication (DR→source) is enabled, HyperCore creates a replication target VM
on source named after the DR VM (e.g., `web-01-2024-06-15_1430-DR`). This is distinct from
the original `web-01` and from the restored clone. Phase 1 (conflict check) handles
collisions of the *original* name only; the `-DR` suffix replication targets don't collide.

---

## Shared Variables (all three playbooks)

```yaml
dr_vm_tag: "DR-failover"
dr_name_suffix_regex: '^(.*)-\d{4}-\d{2}-\d{2}_\d{4}-DR$'
# credentials from inventory: scale_user / scale_pass
```

---

## Phase 1–3: `automated_dr_failback_prepare.yml`

### Purpose
Check for name conflicts on the restored source cluster, configure replication from DR back
to source, and wait for all VMs to reach 100%. Re-runnable/idempotent.

### Variables

```yaml
conflict_action: "abort"          # "abort" or "rename_and_proceed"
rename_prefix: "original"         # prefix for renamed conflict VMs: original-TIMESTAMP-<name>
graceful_shutdown_retries: 30     # × shutdown_poll_delay for shutting down running conflicts
graceful_shutdown_delay: 10
replication_poll_retries: 360     # × replication_poll_delay = 1 hour default
replication_poll_delay: 10
source_remote_cluster_name: ""    # resolved dynamically if empty
dr_remote_cluster_name: ""        # resolved dynamically if empty
```

### Play structure

**Pre-Play (hosts: dr)** — collect DR failover VM names and compute original names; store as facts accessible to source play via hostvars.

**Play 1 (hosts: source)** — conflict check

1. Receive DR failover VM list + original_name mapping from DR hostvars
2. Query all source VMs
3. Identify `running_conflicts` (power_state RUNNING) and `stopped_conflicts` (not RUNNING) by matching original_names
4. Display conflict report
5. **Abort if conflicts and `conflict_action == "abort"`** (default) — list conflicting VMs and instruct operator
6. **Operator pause** if `conflict_action == "rename_and_proceed"` and running conflicts exist
7. Gracefully shut down running conflicts (`vm_params: power_state: shutdown`)
8. Wait up to timeout, then force stop remaining (`vm_params: power_state: stop`)
9. **Rename** conflicting VMs via REST PATCH: `original-YYYYMMDDHHMMSS-<name>`
10. Re-query source VMs and assert no conflicts remain

**Play 2 (hosts: dr)** — reverse replication setup + progress monitoring

1. Verify source cluster connection is ESTABLISHED (`remote_cluster_info`)
2. Resolve source cluster name as seen from DR
3. Get DR failover VMs (tagged `dr_vm_tag`)
4. Idempotency check: if reverse replication already configured and at 100%, skip setup
5. Display VM list + original_name mapping
6. **Operator confirmation pause**
7. Enable reverse replication for each DR failover VM (`vm_replication: state: enabled, remote_cluster: <source>`)
8. Wait for replication target VMs to appear on source (cross-cluster URI poll)
9. Show initial replication progress
10. **Poll until all at 100%** (retries: `replication_poll_retries` × `replication_poll_delay`)
11. Summary: ready for cutover

---

## Phase 4: `automated_dr_failback_cutover.yml`

### Purpose
The actual cutover. Shuts down DR VMs, waits for the final shutdown snapshot to fully
replicate to source, clones with original names on source, powers on, restores forward
replication. Requires operator presence.

### Variables

```yaml
graceful_shutdown_retries: 30     # × graceful_shutdown_delay (default 5 min total)
graceful_shutdown_delay: 10
force_stop_after_graceful: true
snapshot_wait_retries: 60         # × snapshot_wait_delay = 10 min for shutdown snapshot
snapshot_wait_delay: 10
final_replication_retries: 120    # × final_replication_delay = 20 min
final_replication_delay: 10
restore_replication_enabled: true # set false to skip re-enabling forward replication
```

### Play structure

**Play 1 (hosts: dr)** — pre-flight verification

1. Verify source cluster ESTABLISHED
2. Get DR failover VMs
3. Check current replication progress — **fail if any VM not at 100%** (must run prepare first)
4. Verify replication target VMs exist on source
5. **Explicit YES confirmation** — operator must type "YES" (not just press Enter)

**Play 2 (hosts: dr)** — shutdown + final snapshot replication

1. Gracefully shut down DR failover VMs (`vm_params: power_state: shutdown`, skip if already SHUTOFF)
2. Wait for SHUTOFF — force stop any that don't shut down within timeout
3. Record pre-shutdown snapshot counts per VM (`vm_snapshot_info`)
4. **Wait for shutdown snapshot to appear** (count delta from step 3)
5. **Wait for shutdown snapshot to reach 100% replication on source** (URI poll)

**Play 3 (hosts: source)** — clone + power on

1. Get replication target VMs on source (the `-DR` suffix stubs)
2. **Clone each to original name** (`vm_clone`, `preserve_mac_address: true`, clear tags)
3. Power on cloned VMs (`vm_params: power_state: start`)
4. Display which VMs are running
5. **Operator health check pause** — verify applications before continuing

**Play 4 (hosts: source)** — restore forward replication

1. Resolve DR cluster name as seen from source
2. Enable replication from restored source VMs back to DR (`vm_replication: state: enabled`)
3. Report replication configuration status

**Play 5 (hosts: dr)** — summary

- List all outcomes
- Remind operator: run cleanup only after confirming health

---

## Phase 5: `automated_dr_failback_cleanup.yml`

### Purpose
Permanent deletion. Separate, manual-only. Run only after operator confirms source
applications are healthy.

### Variables

```yaml
require_operator_confirmation: true  # require typing "DELETE" before any deletion
```

### Play structure

**Play 1 (hosts: dr)** — safety verification

1. Get DR failover VMs to be deleted
2. **Verify original-named source VMs exist and are RUNNING** — abort if not
3. **Operator confirmation** — must type "DELETE"

**Play 2 (hosts: dr)** — delete DR failover VMs

1. Force stop any still-running DR VMs (should not be running post-cutover)
2. Disable reverse replication (`vm_replication: state: absent`)
3. Delete DR failover VMs (`vm: state: absent`)

**Play 3 (hosts: source)** — delete replication stub VMs

1. Find VMs on source matching `-DR` suffix pattern (the reverse-replication stubs)
2. Display stubs to be deleted — confirm they are NOT the restored original-named VMs
3. Delete stubs (`vm: state: absent`)
4. Final summary: failback complete

---

## `tests/test_failback_live.yml`

### Test VM Setup

Two test VMs on source (matching `test_failover_live.yml` pattern):
- `dr-failback-test-keep` — goes through full failback lifecycle
- `dr-failback-test-conflict` — set to RUNNING on source to exercise the running-conflict path

Both: minimal (256 MB RAM, 1 vCPU, 1 GB virtio disk, VLAN 0), running.

### Play structure

```
Pre-flight:   pre-test cleanup on source and DR (reuse test_failover_live.yml pattern)
Setup:        create test VMs on source, enable replication to DR
              wait for initial replication sync to 100%
Simulated failover:
              run automated_dr_failover.yml via ansible.builtin.command
              (same pattern as test_failover_live.yml — auto-disconnect if available)
              reconnect source before failback phases

Phase 1 test: run prepare with conflict_action=rename_and_proceed
              assert dr-failback-test-conflict was renamed on source
              assert no conflicts remain

Phase 2/3 test: reverse replication configured
              assert replication target VMs appear on source with -DR suffix names
              wait for 100% (with test VMs this should be fast)

Phase 4 test: run cutover
              assert DR VMs are SHUTOFF
              assert source has VMs with original names in RUNNING state
              assert source VMs have no DR-failover tag
              assert forward replication restored

Phase 5 test: run cleanup
              assert no DR-failover tagged VMs remain on DR
              assert no -DR suffix VMs remain on source
              assert original-named VMs still RUNNING

Summary:      per-phase pass/fail table (matching test_failover_live.yml pattern)
Teardown:     delete all test VMs, remove replication
```

### Result tracking

Same `set_fact` + final summary play pattern as `test_failover_live.yml`:
```yaml
result_conflict_check: NOT RUN / PASS / FAIL
result_reverse_replication: NOT RUN / PASS / FAIL
result_cutover: NOT RUN / PASS / FAIL
result_cleanup: NOT RUN / PASS / FAIL
```

---

## Error Handling Map

| Phase | Failure Mode | Handling |
|---|---|---|
| Conflict check | Running source VM, `conflict_action=abort` | Hard fail, list VMs |
| Conflict rename | REST PATCH fails | block/rescue, re-raise |
| Reverse replication setup | `vm_replication` fails | block/rescue per VM |
| Replication progress poll | Never reaches 100% (timeout) | Fail with per-VM progress |
| Cutover pre-flight | Replication not at 100% | Hard fail — run prepare first |
| Graceful shutdown | VM doesn't shut down in time | Force stop, log warning, continue |
| Shutdown snapshot wait | Snapshot never appears | Hard fail — do not clone |
| Final replication wait | Timeout | Hard fail — do not clone |
| Clone on source | `vm_clone` fails | block/rescue, list successes/failures |
| Power on | `vm_params` fails | Log warning, continue (operator powers on manually) |
| Replication restore | `vm_replication` fails | `ignore_errors: true`, log warning |
| Cleanup: delete DR VM | `vm: state: absent` fails | block/rescue, list failures |
| Cleanup: delete source stub | `vm: state: absent` fails | block/rescue, list failures |

---

## Implementation Order

1. **Unit tests first** — add name-stripping regex tests to `tests/test_failback_logic.yml` (no cluster needed)
2. **prepare.yml Phase 1** (conflict check) — test in isolation on live source, should be a no-op if no conflicts
3. **prepare.yml Phase 2+3** (reverse replication + poll) — requires live DR failover state
4. **cutover.yml** — only after replication at 100%; use short test VMs to minimize wait
5. **cleanup.yml** — last and irreversible
6. **test_failback_live.yml** — built alongside, one assertion block per phase
