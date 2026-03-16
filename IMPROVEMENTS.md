# Improvement Backlog — Prioritized

Ranked by production impact vs. implementation effort.

---

## 1. Failover notifications (HIGH impact / LOW effort)

**Why first:** Without alerting, a failover can trigger silently at 3am and no one knows.
The playbook already has a clean summary block — plugging in a notification is one task.

Add a `notify_*` block at the end (and in the rescue block) that sends to one or more targets:

- **Slack / Teams webhook** — `ansible.builtin.uri` POST to an incoming webhook URL
- **Email** — `community.general.mail`
- **PagerDuty** — `community.general.pagerduty_alert`

Variable-driven: set `notify_slack_webhook: ""` and skip if empty. Same pattern as `http_check_host`.

Also notify on **abort** (primary still reachable, idempotency skip) so the operator has a full
audit trail from the scheduled runs, not just when something fires.

---

## 2. VM startup ordering (HIGH impact / MEDIUM effort)

**Why second:** Database must be up before app server. App server before web tier. Without ordering,
random startup causes cascading failures that look like a broken failover.

Add a `vm_startup_groups` variable — an ordered list of lists of VM name patterns:

```yaml
vm_startup_groups:
  - ["db-*"]              # group 1: start first
  - ["app-*"]             # group 2: start after group 1 is up
  - ["web-*", "proxy-*"]  # group 3: start last
  delay_between_groups: 30  # seconds to wait between groups
```

VMs not matched by any group start last. Groups processed sequentially; within a group, VMs
start in parallel. Pairs well with improvement #3 (health checks per group before advancing).

---

## 3. Post-failover health checks (HIGH impact / MEDIUM effort)

**Why third:** Knowing VMs powered on is not the same as knowing the application is ready.
A VM can boot and still fail due to missing network, bad config, or a service crash.

Add optional per-VM health checks after power-on:

```yaml
vm_health_checks:
  - vm_name: "db-primary"
    host: "10.0.0.50"
    port: 5432
    timeout: 120
  - vm_name: "web-01"
    host: "10.0.0.60"
    port: 443
    timeout: 60
```

Use `ansible.builtin.wait_for` per entry. Failed checks trigger a notification (improvement #1)
but do not roll back — report and let the operator decide. Pairs naturally with startup ordering
(#2): check group 1 is healthy before starting group 2.

---

## 4. Replication lag / RPO check before failover (MEDIUM impact / LOW effort)

**Why fourth:** Before committing to failover, the operator should know how stale the replicated
snapshots are. If replication has been broken for 6 hours, the data loss window is 6 hours.

Use `vm_replication_info` to check replication state and last-sync time for each VM before
cloning. Emit a clear warning (and optionally abort if `max_rpo_hours` is exceeded):

```yaml
max_rpo_hours: 1   # abort failover if any VM's last snapshot is older than this; 0 = no limit
```

Also useful as a standalone monitoring playbook run on a schedule to detect replication issues
before a real disaster occurs.

---

## 5. Graceful failover mode (MEDIUM impact / HIGH effort)

**Why fifth:** The current playbook clones from the most recent replication snapshot, which may
be minutes or hours old. A graceful failover minimizes data loss when the primary is still
partially accessible:

1. Quiesce / shut down source VMs on primary
2. Wait for the shutdown snapshot to complete replication to DR
3. Verify replication is current
4. Clone and start on DR

This requires connectivity to both clusters and a configurable `graceful_failover: false` flag.
When `true`, the playbook attempts the graceful path first; if primary is unreachable, it falls
back to clone-from-last-snapshot automatically.

---

## 6. Failback automation (MEDIUM impact / HIGH effort)

**Why sixth:** DR is only complete if you can return to the primary site. Currently failback is
manual. A `automated_dr_failback.yml` companion playbook would:

1. Verify primary cluster is back online and replication is re-established
2. Shut down DR failover VMs gracefully
3. Wait for reverse-replication snapshot to land on primary
4. Power on VMs on primary
5. Clean up DR failover VMs and remove `dr_vm_tag`

Pairs with improvement #1 (notify when failback completes).

---

## 7. Per-VM configuration via tags (MEDIUM impact / LOW effort)

**Why seventh:** The current model is all-or-nothing. Adding tag-driven per-VM metadata allows
fine-grained control without editing the playbook:

| Tag on source VM | Effect |
|---|---|
| `dr-exclude` | Skip this VM during failover |
| `dr-group-1` | Start in group 1 (see #2) |
| `dr-check-port-443` | Health check port 443 after start (see #3) |
| `dr-delay-60` | Wait 60s after power-on before advancing |

Tags are already preserved through HyperCore replication, making this zero-extra-config for
the playbook operator once source VMs are tagged.

---

## 8. Dry-run / simulation mode (LOW impact / LOW effort)

**Why eighth:** `--check` mode doesn't fully work with HyperCore modules (most don't support it).
A `simulate: false` variable would run all detection phases (replication check, pings, idempotency)
and print exactly what would happen, but skip the clone and power-on tasks:

```
ansible-playbook automated_dr_failover.yml -e "simulate=true"
```

Useful for validating inventory, credentials, and detection logic without risk. Good for onboarding
new operators and for scheduled testing without a real outage.

---

## 9. Structured failover event log (LOW impact / LOW effort)

**Why ninth:** AWX/cron runs are ephemeral. A persistent log of every failover event (what
triggered it, which VMs were cloned, timestamp) is valuable for post-incident review and
auditing.

Append a JSON or CSV record to a local log file (or ship to a syslog/SIEM endpoint) at the
end of every run — both when failover fires and when it exits cleanly. Lightweight: one
`ansible.builtin.lineinfile` or `ansible.builtin.uri` task.

---

## 10. Cascading / tertiary site support (LOW impact / HIGH effort)

**Why last:** If the DR cluster itself fails, a tertiary site could take over. This requires
chaining playbook runs across three clusters and handling significantly more state. Edge case
for most deployments, but documented here for completeness.

The current architecture (one playbook per DR cluster, each independently scheduled) already
supports two-tier DR. Extending to three tiers would require a coordinating layer to prevent
split-brain across all three sites simultaneously.
