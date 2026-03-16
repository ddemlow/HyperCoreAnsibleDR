# Manual Test Plan

Validates the DR failover playbook against a real HyperCore environment.
Requires two clusters with replication configured.

## Test environment prerequisites

- Two HyperCore clusters (source and DR), ICOS v9.4+
- At least one VM replicating from source → DR
- Ansible controller that can reach both cluster APIs and ping all node IPs
- Playbook installed and inventory configured per README

Tag two VMs on the DR cluster before starting:
- One VM: add tag `dr-exclude` (used in TC-06)
- The replicated VM(s): leave untagged

---

## TC-01 — Clean run: primary ESTABLISHED, no failover

**Scenario:** Primary cluster is fully operational.

**Steps:**
1. Confirm the source cluster is online and replication shows `ESTABLISHED`
2. Run the playbook:
   ```
   ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml -v
   ```

**Expected:**
- Play exits at Phase 1 with message: `Exit — all monitored clusters are ESTABLISHED`
- No VMs are cloned
- Exit code 0
- Runtime under 10 seconds

---

## TC-02 — Primary ping succeeds: no failover even if replication flaps

**Scenario:** Replication link shows `DISCONNECTED` but primary nodes are still pingable
(simulates a replication config issue, not a site failure).

**Steps:**
1. On the DR cluster, temporarily break the replication connection (remove and re-add, or
   use the HyperCore UI to disconnect — do NOT power off the source cluster)
2. Confirm `remote_cluster_info` shows a non-ESTABLISHED status
3. Run the playbook

**Expected:**
- Phase 1 passes (not ESTABLISHED)
- Phase 2: ping succeeds for at least one primary node
- Play exits: `Exit — a primary node responded (split-brain risk, no failover)`
- No VMs are cloned

**Cleanup:** Restore the replication connection.

---

## TC-03 — Full failover: primary completely unreachable

**Scenario:** Source cluster is offline. This is the primary happy path.

**Steps:**
1. Power off the source cluster (or disconnect it from the network at the switch level)
2. Wait 2–3 minutes for the replication connection to show `DISCONNECTED`
3. Run the playbook with verbose output:
   ```
   ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml -v
   ```

**Expected:**
- Phase 1: detects DISCONNECTED cluster(s)
- Phase 2: all node pings fail after retries
- Phase 3: DR cluster nodes all respond
- Phase 4: finds replicated VMs, no existing DR-tagged VMs
- Phase 5: clones each replicated VM with `-YYYY-MM-DD_HHMM-DR` suffix
- Phase 6: powers on each cloned VM
- Summary block shows correct counts
- Exit code 0

**Verify after:**
- DR cluster shows cloned VMs running
- Cloned VMs have the `DR-failover` tag
- MAC addresses match originals (check VM NIC details)

**Cleanup:** Power on source cluster, delete cloned DR VMs, confirm replication resumes.

---

## TC-04 — Idempotency: re-run after failover does nothing

**Scenario:** Failover has already occurred. Re-run must not clone again.

**Pre-condition:** TC-03 has been run and DR VMs exist with `DR-failover` tag.

**Steps:**
1. With source cluster still offline, run the playbook again

**Expected:**
- Phase 4 detects existing DR-tagged VMs
- Play exits: `Failover already performed — N DR VM(s) exist, exiting`
- No new VMs are cloned
- Exit code 0

---

## TC-05 — DR cluster node unreachable: abort

**Scenario:** A DR cluster node is not responding. Failover should be aborted.

**Steps:**
1. Power off the source cluster (so Phase 1 and 2 would normally proceed)
2. Temporarily block ICMP to one DR cluster node IP (firewall rule or pull a cable)
3. Run the playbook

**Expected:**
- Phases 1–2 pass (primary is down)
- Phase 3: ping fails for the unreachable DR node
- Play fails with: `DR cluster node X.X.X.X did not respond to ping`
- No VMs are cloned
- Exit code non-zero

**Cleanup:** Restore DR node connectivity, restore source cluster.

---

## TC-06 — VM exclusion: dr-exclude tagged VMs are skipped

**Pre-condition:** At least one replicated VM on DR cluster has the tag `dr-exclude`.

**Steps:**
1. Power off source cluster
2. Run with exclusion tag set:
   ```
   ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
     -e "dr_exclude_tag=dr-exclude"
   ```

**Expected:**
- Phase 4 reports excluded VM(s) by name
- Excluded VMs are NOT cloned
- Other replicated VMs ARE cloned and powered on
- Summary shows correct excluded count

---

## TC-07 — source_cluster_name filter: only named cluster is monitored

**Pre-condition:** DR cluster has connections to two remote clusters.

**Steps:**
1. Power off only source cluster A (leave source cluster B running)
2. Run with `source_cluster_name` set to cluster A's name:
   ```
   ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
     -e "source_cluster_name=cluster-a"
   ```
3. Then run again with `source_cluster_name` set to cluster B's name

**Expected (run 1 — watching cluster A which is down):**
- Failover proceeds for all replicated VMs

**Expected (run 2 — watching cluster B which is still up):**
- Phase 1: cluster B is ESTABLISHED → exit immediately, no failover

---

## TC-08 — http_check_host prevents failover when app is alive

**Pre-condition:** Source cluster is down OR replication is disconnected.
An application endpoint at the primary site is still reachable (e.g., via secondary path).

**Steps:**
1. Configure `http_check_host` to a host that IS reachable and `http_check_port` to an open port
2. Take down the source cluster (or disconnect replication)
3. Run the playbook:
   ```
   ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
     -e "http_check_host=10.0.0.1 http_check_port=443"
   ```

**Expected:**
- Phases 1–2 indicate primary is down
- Phase 2 http check: TCP connection to `http_check_host:http_check_port` succeeds
- Play exits: `application at X.X.X.X is still responding (no failover needed)`
- No VMs are cloned

---

## TC-09 — Invalid source_cluster_name: clear error message

**Steps:**
1. Run with a cluster name that does not exist:
   ```
   ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
     -e "source_cluster_name=does-not-exist"
   ```

**Expected:**
- Phase 1 fails with a clear message listing available cluster names
- Exit code non-zero
- No VMs are touched

---

## TC-10 — Multiple DR clusters in inventory run independently

**Pre-condition:** Inventory has two DR cluster hosts. Source cluster for DR-cluster-1 is down;
source cluster for DR-cluster-2 is up.

**Steps:**
1. Run playbook against full inventory (both hosts)

**Expected:**
- DR-cluster-1: completes full failover
- DR-cluster-2: exits at Phase 1 (ESTABLISHED)
- Both complete — neither cancels the other (`end_host` not `end_play`)
- Exit code 0

---

## Regression checklist (run after any code change)

- [ ] TC-01: clean run exits immediately
- [ ] TC-03: full failover succeeds end-to-end
- [ ] TC-04: idempotent re-run is a no-op
- [ ] TC-06: excluded VMs are not cloned
