# Troubleshooting Guide

Covers every common failure mode from installation through scheduled operation.
Use the section headers to jump to your specific problem.

---

## Contents

1. [Installation problems](#1-installation-problems)
2. [Connectivity problems](#2-connectivity-problems)
3. [Replication detection problems](#3-replication-detection-problems)
4. [Failover not triggering when it should](#4-failover-not-triggering-when-it-should)
5. [Failover triggering when it should not](#5-failover-triggering-when-it-should-not)
6. [Clone failures](#6-clone-failures)
7. [Power-on failures](#7-power-on-failures)
8. [Idempotency problems](#8-idempotency-problems)
9. [Multi-cluster problems](#9-multi-cluster-problems)
10. [Scheduling problems (cron / AWX)](#10-scheduling-problems-cron--awx)
11. [Ansible Vault problems](#11-ansible-vault-problems)
12. [Collecting diagnostic information](#12-collecting-diagnostic-information)

---

## 1. Installation problems

### `python3: command not found`

Python is not installed or not in PATH.

- **macOS:** `brew install python3` (install Homebrew first from brew.sh if needed)
- **Ubuntu/Debian:** `sudo apt update && sudo apt install python3 python3-pip python3-venv`
- **Windows:** Install WSL2 first (`wsl --install` in PowerShell as Admin), then use Ubuntu terminal

### `python3 --version` shows 3.8 or older

Ansible-core 2.16 requires Python 3.10+.

```bash
# macOS: install a newer version
brew install python@3.12

# Ubuntu: add deadsnakes PPA
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt install python3.12 python3.12-venv

# Then create venv with the specific version:
python3.12 -m venv .venv
```

### `pip install -r requirements.txt` fails with `error: externally-managed-environment`

You are trying to install into the system Python. Always activate the virtual environment first:

```bash
source .venv/bin/activate
# Prompt should show (.venv)
pip install -r requirements.txt
```

### `ansible-galaxy collection install` fails with SSL error

Corporate networks sometimes intercept TLS. Try:

```bash
ansible-galaxy collection install -r requirements.yml --ignore-certs
```

Or configure your corporate CA certificate:

```bash
export SSL_CERT_FILE=/path/to/corporate-ca.crt
ansible-galaxy collection install -r requirements.yml
```

### `ansible: command not found` after installation

The virtual environment is not activated. Run:

```bash
source .venv/bin/activate
```

If you are running from a script or cron, use the full path to the venv's ansible binary:

```bash
/path/to/project/.venv/bin/ansible-playbook
```

---

## 2. Connectivity problems

### `UNREACHABLE: Failed to connect to the host`

Ansible cannot reach the DR cluster API.

**Step 1:** Confirm the host in inventory is correct:
```bash
cat inventory/inventory.yml
```

**Step 2:** Test basic network access from the controller:
```bash
curl -k https://YOUR_CLUSTER_IP/rest/v1/Cluster
```
If curl fails, the problem is network/firewall, not Ansible.

**Step 3:** Check that port 443 is open:
```bash
nc -zv YOUR_CLUSTER_IP 443
```

**Step 4:** If using a hostname instead of IP, verify DNS resolves:
```bash
nslookup your-cluster-hostname
```

### `Authentication failed` / `401 Unauthorized`

The username or password is wrong, or the account is locked.

```bash
# Test credentials directly against the API:
curl -k -u admin:yourpassword https://YOUR_CLUSTER_IP/rest/v1/Cluster
```

Log into the HyperCore web UI with the same credentials to confirm they work.
If the account is locked, unlock it via another admin account in the HyperCore UI.

### `SSL: CERTIFICATE_VERIFY_FAILED`

HyperCore uses a self-signed certificate by default. The playbook sets `SC_HOST` with
`https://` which may trigger certificate verification in some environments.

Set the environment variable to disable verification (already handled by the HyperCore
collection internally, but if you see this error check collection version):

```bash
ansible-galaxy collection list scale_computing.hypercore
# Ensure version >= 1.3.0
ansible-galaxy collection install scale_computing.hypercore --upgrade
```

### Connectivity test succeeds in terminal but fails in cron

The virtual environment is not activated in the cron job. Use the full path:

```
*/5 * * * * /home/user/hypercore-ansible-dr-failover/.venv/bin/ansible-playbook \
  -i /home/user/hypercore-ansible-dr-failover/inventory/inventory.yml \
  /home/user/hypercore-ansible-dr-failover/automated_dr_failover.yml
```

Always use **absolute paths** in cron — relative paths depend on the working directory,
which cron does not set to your project folder by default.

---

## 3. Replication detection problems

### `No remote cluster connections found`

The DR cluster has no remote cluster connections configured.

**Verify in HyperCore UI:** *Data Protection → Remote Clusters*

A connection must exist and show the primary cluster. If it's missing, replication has not
been configured from the primary side. Set it up in the HyperCore UI on the **primary**
cluster under *Data Protection → Remote Clusters → Add*.

**Verify via Ansible:**
```bash
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.remote_cluster_info \
  -v
```

### `source_cluster_name 'X' not found`

The name you set in `source_cluster_name` does not match any remote cluster record.
The error message lists the available names — use one of those exactly (case-sensitive).

```bash
# See all available cluster names:
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.remote_cluster_info \
  | grep '"name"'
```

### Remote cluster shows `ESTABLISHED` but primary is actually offline

The replication connection status may take several minutes to update after a failure.
The HyperCore cluster reports this status based on its own heartbeat interval.

The playbook handles this automatically: once pings confirm the source nodes are
unreachable, it polls `remote_cluster_info` until the status changes before proceeding.
You can monitor the status change manually if needed:
```bash
watch -n 30 'ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.remote_cluster_info 2>/dev/null \
  | grep connection_status'
```

### Remote cluster shows `DISCONNECTED` but primary is online

The replication link is broken even though the primary cluster is healthy. This can happen
if the replication configuration is misconfigured or the replication network path is blocked
even though the management network path is up.

The playbook correctly proceeds to ping the primary node IPs in this case. If the IPs
respond, failover is aborted. The replication link should be investigated and repaired
separately in the HyperCore UI.

---

## 4. Failover not triggering when it should

Work through this checklist when the primary is genuinely down but no failover occurs.

### Check 1: Connection status has not updated yet

The playbook polls for the status change automatically after pings fail, so this should
not block failover under normal circumstances. If failover still did not trigger, check
the current status manually:

```bash
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.remote_cluster_info \
  | grep connection_status
```

If it still shows `ESTABLISHED` after the playbook's disconnect-detection polling
exhausted all retries, the DR cluster may still have a path to the primary — replication
may be on a separate VLAN that is still up even though the management network is down.
Check routing and adjust `disconnect_detect_retries`/`disconnect_detect_delay` if needed.

### Check 2: Primary node IPs are still responding to ping

```bash
# See what IPs the DR cluster knows about for the primary:
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.remote_cluster_info \
  | grep -A5 remote_node_ips

# Try pinging each one from the controller:
ping -c 3 10.0.0.1
```

If pings succeed, the playbook correctly aborts — the primary nodes are still reachable.
This is a split-brain situation: the replication link is broken but the nodes are up.
Do not force a failover in this case without manually confirming VMs are not running on
the primary.

### Check 3: `http_check_host` is responding

If you have configured `http_check_host`, verify whether that endpoint is actually down:

```bash
nc -zv YOUR_HTTP_CHECK_HOST YOUR_HTTP_CHECK_PORT
```

If it responds, the playbook aborts correctly. If the application is down but the host
still responds (e.g., the web server returns 503), TCP-level checks still count as "alive."

### Check 4: No replicated VMs exist on the DR cluster

```bash
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.vm_info \
  | grep replication_source_vm_uuid
```

Any VM with a non-empty UUID is a replication target. If all are empty strings, replication
has not been configured from the primary side, or the primary has not yet sent a snapshot.

---

## 5. Failover triggering when it should not

### Primary was briefly unreachable (network blip)

Increase `ping_retries` and `ping_delay` to require more sustained unreachability:

```bash
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  -e "ping_retries=6 ping_delay=10"
```

With `ping_retries=6` and `ping_delay=10`, the primary must be unreachable for 60+ seconds
across all node IPs before failover proceeds.

### Failover ran while the primary was in maintenance

Add an `http_check_host` that points to a resource that is only alive when the primary is
genuinely operational (not just a node IP, but a management endpoint or application port
that would be intentionally disabled during planned maintenance).

Alternatively, for planned maintenance windows, stop the cron job or disable the AWX
schedule temporarily:
```bash
# Disable cron temporarily:
crontab -l | grep -v 'automated_dr_failover' | crontab -
```

### The controller itself lost network connectivity

If the Ansible controller lost connectivity to both the DR cluster AND the primary node IPs
at the same time (e.g., the controller's network link went down), the playbook would see:
- DR cluster API unreachable → `UNREACHABLE` error → play fails (does not clone)

This is safe behavior — the play fails rather than proceeding with a clone, because it
cannot even reach the DR cluster API.

---

## 6. Clone failures

### `VM name already exists`

A VM with the generated clone name already exists on the DR cluster. This can happen if:

- A previous partial run created some clones but not others
- The minute changed between loop iterations (e.g., `web-01-2024-06-15_1437-DR` exists but
  you re-ran at `14:38` creating `web-01-2024-06-15_1438-DR` — this would actually succeed)

**Identify existing partial clones:**
```bash
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.vm_info \
  | grep -E 'vm_name.*DR'
```

Delete partial clones via the HyperCore UI, then re-run the playbook.

### `Insufficient storage on DR cluster`

The DR cluster doesn't have enough free storage to clone all replicated VMs.

Check available storage in the HyperCore UI under *System → Storage*. The clones require
at least as much space as the current snapshot of each source VM.

If storage is the constraint, use `dr_exclude_tag` to temporarily skip the largest VMs
and clone critical ones first.

### Clone task runs but `cloned_vms` shows no `changed` items

This can happen if the vm_clone module returns success but reports `changed=false` for some
reason. Check the module output directly with `-vv`:

```bash
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml -vv \
  | grep -A 20 "Clone replicated"
```

### One VM clone fails, others succeed (partial failover)

The `block/rescue` structure catches this and reports it clearly. The rescue output will
show which task failed and why. VMs that were successfully cloned but not yet powered on
will need to be started manually via the HyperCore UI.

After resolving the cause of the clone failure, re-run the playbook — the idempotency check
will detect existing DR-tagged VMs and exit. To retry, either delete the partially cloned
VMs or change `dr_vm_tag` to a new value.

---

## 7. Power-on failures

### VM powers on but never becomes reachable

The VM booted but networking may not be configured correctly. Common causes:

- **MAC address conflict:** Another VM on the DR cluster has the same MAC. Check for
  duplicate NICs in the HyperCore UI.
- **IP conflict:** Same IP is in use elsewhere on the DR network. Check DHCP leases.
- **Wrong VLAN:** The DR cluster may use different VLAN IDs than the primary. Verify
  network configuration on the cloned VMs.

### `VM is already powered on`

The replicated source VM on the DR cluster is already running (some HyperCore versions
allow replicated VMs to be powered on). The power-on step will skip it (`changed=false`).
This is usually harmless — the VM is already running.

### Power-on succeeds but VM stops immediately

The VM may have a startup error (guest OS issue, not an Ansible issue). Check the VM's
console in the HyperCore UI under *VMs → [VM name] → Console*.

---

## 8. Idempotency problems

### Playbook keeps cloning on every run after failover

The `dr_vm_tag` on the cloned VMs does not match the `dr_vm_tag` variable.

**Check what tags the cloned VMs actually have:**
```bash
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.vm_info \
  | grep -B2 -A2 '"tags"'
```

**Check what tag the playbook is looking for:**
```bash
grep 'dr_vm_tag' automated_dr_failover.yml
```

If they don't match (e.g., tag was changed after the initial failover), either update the
variable to match the existing tag, or manually add the correct tag to the existing DR VMs
in the HyperCore UI.

### Need to re-run failover after cleaning up

After deleting the DR VMs or if you want to re-trigger:

1. Delete the DR failover VMs in HyperCore UI (or power off and delete via Ansible)
2. Confirm no VMs with `dr_vm_tag` remain:
   ```bash
   ansible -i inventory/inventory.yml all \
     -m scale_computing.hypercore.vm_info \
     | grep "DR-failover"
   ```
3. Re-run the playbook

Or use a different tag:
```bash
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  -e "dr_vm_tag=DR-failover-2"
```

---

## 9. Multi-cluster problems

### Only one cluster fails over when two are expected

Check that both remote cluster connections appear in `remote_cluster_info`:
```bash
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.remote_cluster_info \
  | grep -E 'name|connection_status'
```

If `source_cluster_name` is set, it filters to only that cluster. Leave it empty (`""`) to
monitor all clusters.

### Failover fires for ALL VMs when only one source cluster is down

This is a known limitation. The HyperCore API on the target cluster does not expose which
source cluster a replicated VM came from. When any confirmed-down cluster is detected, all
replicated VMs are cloned.

**Workaround:** Tag VMs on the primary cluster with their source cluster name (e.g.,
`source-site-a`) before setting up replication. Then use `dr_exclude_tag` per run, or
create separate inventories per source cluster.

### Two DR clusters in inventory — one exits early and kills the other

This was a bug in earlier versions (`meta: end_play` instead of `meta: end_host`). Ensure
you are using the current version of the playbook:
```bash
git log --oneline -5
```

`meta: end_host` exits only the current host's play, allowing other hosts to continue.
If you see `end_play` anywhere in the playbook, upgrade to the latest version.

---

## 10. Scheduling problems (cron / AWX)

### Cron job is not running

Check that cron is running on the controller:
```bash
systemctl status cron     # Debian/Ubuntu
systemctl status crond    # RHEL/CentOS
```

Verify your crontab entry is saved:
```bash
crontab -l
```

Test the exact command from your crontab manually in a shell to confirm it works outside cron.

### Cron runs but produces no output / empty log file

The path to the log file may not be writable, or the path doesn't exist:
```bash
touch /var/log/dr-failover.log && ls -la /var/log/dr-failover.log
```

If `/var/log/` is not writable by your user:
```bash
# Use a path in your home directory instead:
>> /home/youruser/dr-failover.log 2>&1
```

### AWX job fails with `No module named 'scale_computing'`

The HyperCore collection is not installed in the AWX Python environment. In AWX:

1. Go to *Execution Environments* and create a custom EE that includes the collection, OR
2. Use a *Requirements file* in the project to auto-install at sync time, OR
3. Install the collection manually on the AWX controller:
   ```bash
   ansible-galaxy collection install scale_computing.hypercore
   ```

### AWX schedule runs but shows `ESTABLISHED` even during an outage

AWX may be caching inventory or using a stale connection. Ensure the inventory source is
set to re-sync on each job run, and that `SC_HOST` environment variables are set at the
job template level, not cached from a previous run.

---

## 11. Ansible Vault problems

### `Decryption failed` when running with vault-encrypted password

The vault password you provided does not match the one used when encrypting.

```bash
# Test vault decryption:
ansible-vault decrypt --ask-vault-pass inventory/inventory.yml
```

If you have forgotten the vault password, re-encrypt with a new one:
```bash
# Re-enter the plain-text password:
ansible-vault encrypt_string 'YourNewPassword' --name 'scale_pass'
# Copy the output back into inventory/inventory.yml
```

### `vault_pass.txt` not found in cron

Use an absolute path:
```
*/5 * * * * .venv/bin/ansible-playbook \
  --vault-password-file /home/user/hypercore-ansible-dr-failover/vault_pass.txt \
  -i /home/user/hypercore-ansible-dr-failover/inventory/inventory.yml \
  /home/user/hypercore-ansible-dr-failover/automated_dr_failover.yml
```

Ensure `vault_pass.txt` has restrictive permissions:
```bash
chmod 600 vault_pass.txt
```

---

## 12. Collecting diagnostic information

When opening a GitHub issue or asking for help, include the output of these commands
(sanitize IP addresses and passwords before sharing):

```bash
# 1. Versions
ansible --version
ansible-galaxy collection list scale_computing.hypercore

# 2. Inventory structure (no passwords)
ansible-inventory -i inventory/inventory.yml --list | python3 -m json.tool \
  | sed 's/"scale_pass": "[^"]*"/"scale_pass": "REDACTED"/g'

# 3. Remote cluster status
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.remote_cluster_info -v 2>&1 \
  | sed 's/10\.[0-9]*\.[0-9]*\.[0-9]*/X.X.X.X/g'

# 4. VM replication status
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.vm_replication_info -v 2>&1

# 5. Full playbook run with maximum verbosity
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml -vvv 2>&1 \
  | sed 's/10\.[0-9]*\.[0-9]*\.[0-9]*/X.X.X.X/g' \
  | sed 's/scale_pass: "[^"]*"/scale_pass: "REDACTED"/g'
```

Paste the output into the GitHub Issue using a code block (wrap in triple backticks).

---

## Quick reference: exit codes

| Exit code | Meaning |
|---|---|
| `0` | All tasks succeeded (including a clean no-failover exit) |
| `1` | Fatal Ansible error (syntax, import, etc.) |
| `2` | One or more tasks failed (failover block failure, assertion failure) |
| `3` | Host unreachable (controller cannot reach DR cluster API) |
| `4` | Host parse error (inventory syntax problem) |

Check exit code in scripts with `$?` after running the playbook:
```bash
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml
if [ $? -ne 0 ]; then
  echo "Playbook failed - investigate log" | mail -s "DR FAILOVER ERROR" ops@company.com
fi
```
