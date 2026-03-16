---
marp: true
theme: default
paginate: true
backgroundColor: #1a1a2e
color: #eaeaea
style: |
  section {
    font-family: 'Segoe UI', Arial, sans-serif;
    padding: 48px 64px;
  }
  h1 {
    color: #00aaff;
    font-size: 2em;
    border-bottom: 3px solid #00aaff;
    padding-bottom: 12px;
  }
  h2 {
    color: #00aaff;
    font-size: 1.5em;
  }
  h3 {
    color: #88ccff;
    font-size: 1.1em;
    margin-bottom: 4px;
  }
  code {
    background: #0d1117;
    color: #58d68d;
    padding: 2px 8px;
    border-radius: 4px;
    font-size: 0.85em;
  }
  pre {
    background: #0d1117;
    border-left: 4px solid #00aaff;
    padding: 20px;
    border-radius: 6px;
  }
  pre code {
    background: transparent;
    color: #58d68d;
    font-size: 0.82em;
    padding: 0;
  }
  ul {
    line-height: 1.9;
  }
  li {
    margin-bottom: 4px;
  }
  .columns {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 40px;
  }
  table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.85em;
  }
  th {
    background: #00aaff;
    color: #000;
    padding: 8px 12px;
  }
  td {
    border-bottom: 1px solid #333;
    padding: 8px 12px;
  }
  .highlight {
    color: #00aaff;
    font-weight: bold;
  }
  .green { color: #58d68d; }
  .yellow { color: #f5b942; }
  .red { color: #e74c3c; }
  section.title {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    text-align: center;
  }
  section.title h1 {
    font-size: 2.6em;
    border: none;
  }
  section.title h2 {
    font-size: 1.3em;
    color: #aaa;
    font-weight: normal;
  }
---

<!-- _class: title -->

# Automated DR Failover
# for HyperCore

## Getting Started with Ansible
### Scale Computing

---

# The Problem

Manual DR failover is slow and error-prone

<br>

When the primary site goes down, someone must:

- **Notice** the outage (not always immediate)
- **Confirm** it's real, not a blip
- **Log in** to the DR cluster
- **Find** all the replicated VMs
- **Clone** each one manually
- **Power them on** in the right order
- **Verify** applications are running

<br>

> Every minute of delay is downtime your users feel

---

# The Solution

An Ansible playbook that runs every 5 minutes — automatically

<br>

```
Primary up?   → exit silently, do nothing         (most of the time)

Primary down? → clone all replicated VMs
               → power them on
               → done                             (under 2 minutes)
```

<br>

**No manual steps. No on-call engineer needed at 2am.**

---

# How It Decides to Fail Over

Multiple independent signals — never acts on a single check

<br>

| Phase | Check | If passes → |
|---|---|---|
| 1 | Replication link `ESTABLISHED`? | Exit — primary is up |
| 2 | Primary node IPs respond to ping? | Exit — primary is up |
| 2b | Application TCP port alive? *(optional)* | Exit — app is up |
| 3 | DR cluster nodes healthy? | Abort — DR not ready |
| 4 | Failover VMs already exist? | Exit — already done |
| 5–6 | **Clone + power on** | ✓ Failover complete |

<br>

> If primary node IPs still respond after retries → **abort** (could be split-brain)

---

# Prerequisites

Before you start, confirm you have:

<br>

- ✅ **A controller machine** — Windows (WSL2), macOS, or Linux
- ✅ **Python 3.10+** — usually pre-installed on Mac/Linux
- ✅ **Network access to DR cluster** on port 443 (HTTPS)
- ✅ **Ability to ping** primary cluster node IPs from the controller
- ✅ **HyperCore admin credentials** for the DR cluster
- ✅ **Replication configured** — VMs replicating from primary → DR

<br>

> Windows users: install WSL2 first (`wsl --install` in PowerShell as Admin)

---

# Step 1 — Install Ansible

```bash
# 1. Check Python version (need 3.10+)
python3 --version

# 2. Clone the project
git clone https://github.com/ScaleComputing/hypercore-ansible-dr-failover.git
cd hypercore-ansible-dr-failover

# 3. Create a virtual environment (isolated sandbox)
python3 -m venv .venv

# 4. Activate it  ← run this every time you open a new terminal
source .venv/bin/activate
#   Your prompt will show (.venv) when active

# 5. Install Ansible
pip install -r requirements.txt

# 6. Install the HyperCore collection
ansible-galaxy collection install -r requirements.yml
```

---

# Step 2 — Configure Your Inventory

The inventory tells Ansible which cluster to connect to

<br>

```bash
# Copy the example file
cp inventory/inventory.example.yml inventory/inventory.yml
```

<br>

Edit `inventory/inventory.yml` — change these three values:

```yaml
all:
  hosts:
    10.0.0.100:              # ← your DR cluster management IP
      scale_user: admin      # ← your HyperCore username
      scale_pass: "admin"    # ← your HyperCore password
```

<br>

> `inventory/inventory.yml` is excluded from git — your credentials stay local

---

# Step 3 — Test Connectivity

Verify Ansible can reach your DR cluster

<br>

```bash
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.cluster_info
```

<br>

**Expected output:**
```yaml
record:
  name: my-dr-cluster
  icos_version: 9.4.1.210763
  uuid: a5d9148c-37f7-4b43-843c-196751d3c050
```

<br>

> If you see an error: check the IP, try opening `https://your-cluster-ip` in a browser

---

# Step 4 — First Run (Primary Online)

Run the playbook while primary is healthy — it should exit immediately

<br>

```bash
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml -v
```

<br>

**Expected:**
```
TASK [[1/6] Exit — all monitored clusters are ESTABLISHED]
```

<br>

**PLAY RECAP:**
```
10.0.0.100 : ok=4  changed=0  unreachable=0  failed=0
```

<br>

> `-v` = verbose mode. Use `-vv` or `-vvv` for even more detail when troubleshooting

---

# Step 5 — Run the Automated Tests

Validates decision logic without needing a live failover

<br>

```bash
ansible-playbook tests/test_failover_logic.yml
```

<br>

All 15 tasks should show **PASS**:

```
TASK [[1.1] ESTABLISHED cluster produces empty disconnected list]
ok: [localhost] => msg: 'PASS: ESTABLISHED cluster correctly produces no disconnected clusters'

TASK [[4.3] dr_exclude_tag correctly excludes tagged VMs]
ok: [localhost] => msg: 'PASS: dr_exclude_tag correctly excludes tagged VM'

PLAY RECAP
localhost : ok=15  changed=0  failed=0
```

---

# Step 6 — Schedule It

### Option A: Cron (simple)

```bash
crontab -e
```
Add this line:
```
*/5 * * * * cd /home/user/hypercore-ansible-dr-failover && \
  .venv/bin/ansible-playbook -i inventory/inventory.yml \
  automated_dr_failover.yml >> /var/log/dr-failover.log 2>&1
```

<br>

### Option B: AWX / Ansible Automation Platform (recommended)

1. Add repo as a **Project**
2. Create an **Inventory** with your DR cluster host
3. Create a **Job Template** → `automated_dr_failover.yml`
4. Add a **Schedule**: every 5 minutes

---

# Key Variables

All have sensible defaults — override only what you need

<br>

| Variable | Default | When to change |
|---|---|---|
| `source_cluster_name` | `""` (all) | Multiple source clusters — watch only one |
| `ping_retries` | `3` | Increase for flaky networks; decrease for faster RTO |
| `ping_delay` | `5` | Seconds between ping attempts |
| `dr_vm_tag` | `DR-failover` | Change if you have existing VMs with this tag |
| `dr_exclude_tag` | `""` | Set to skip infra VMs (DNS, NTP, monitoring) |
| `http_check_host` | `""` | Add an app-alive endpoint as extra abort signal |

<br>

Override at runtime: `-e "ping_retries=5 dr_exclude_tag=dr-exclude"`

---

# What a Real Failover Looks Like

```
TASK [[1/6] Disconnected cluster(s) detected — continuing checks]
  msg: 1 disconnected cluster(s): ['primary-site'] → DISCONNECTED

TASK [[2/6] Ping all disconnected cluster nodes (3 retries, 5s delay)]
  → 10.0.0.1 ... FAILED (no response after 3 retries)
  → 10.0.0.2 ... FAILED (no response after 3 retries)

TASK [[2/6] All primary node checks failed — primary appears to be offline]

TASK [[3/6] DR cluster nodes healthy ✓]

TASK [[4/6] Found 3 replicated VM(s) to fail over: ['web-01','db-01','app-01']]

TASK [[5/6] Clone replicated VMs for DR failover]
  → web-01-2024-06-15_1437-DR ... changed
  → db-01-2024-06-15_1437-DR  ... changed

TASK [[6/6] Power on cloned DR VMs]
  → all VMs started ✓

DR FAILOVER COMPLETE ✓  VMs cloned: 3/3  Powered on: 3
```

---

# Excluding VMs from Failover

Some VMs should NOT be auto-cloned (DNS, NTP, monitoring)

<br>

**On the DR cluster**, add the exclusion tag to those VMs via the HyperCore UI.

Then run with:

```bash
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  -e "dr_exclude_tag=dr-exclude"
```

<br>

The playbook will report what was skipped:

```
TASK [[4/6] Report excluded VMs]
  msg: Skipping 1 VM(s) excluded by tag 'dr-exclude': ['infra-dns']
```

---

# Multiple Source Clusters

<br>

<div class="columns">

<div>

**Default behavior** (`source_cluster_name: ""`):
- Monitors **all** remote clusters
- If any cluster is DISCONNECTED and all its nodes unreachable → failover
- All replicated VMs are cloned

</div>

<div>

**Named cluster** (`source_cluster_name: "site-a"`):
- Monitors **only** that cluster
- Other clusters are ignored

</div>

</div>

<br>

> ⚠️ **Known limitation:** The HyperCore API does not expose which source cluster
> a replicated VM came from. All replicated VMs are cloned when any
> confirmed-down cluster is detected.

---

# Understanding Output

| Status | Meaning |
|---|---|
| <span class="green">ok</span> | Task ran, nothing changed |
| <span class="yellow">changed</span> | Task ran and made a change (VM cloned, powered on) |
| <span class="yellow">skipped</span> | Task skipped — `when` condition was false |
| <span class="red">failed</span> | Task failed — read the error message |

<br>

**Healthy run (primary up):**
```
ok=4  changed=0  unreachable=0  failed=0
```

**Successful failover:**
```
ok=14  changed=6  unreachable=0  failed=0
```

---

# Planned Enhancements

| # | Feature | Value |
|---|---|---|
| 1 | **Notifications** (Slack/Teams/email) | Know the moment failover fires |
| 2 | **VM startup ordering** | DB before app before web tier |
| 3 | **Post-failover health checks** | Confirm apps respond after boot |
| 4 | **Replication lag / RPO check** | Know your data loss window |
| 5 | **Graceful failover** | Quiesce source → zero data loss |
| 6 | **Failback automation** | Return to primary after recovery |
| 7 | **Per-VM config via tags** | Fine-grained control without editing |
| 8 | **Dry-run mode** | Simulate without touching anything |

<br>

See `IMPROVEMENTS.md` for full details and implementation notes

---

<!-- _class: title -->

# You're Ready

<br>

**Validate:** Follow `MANUAL_TEST_PLAN.md` for a controlled failover test

**Reference:** `README.md` — full variable documentation

**Contribute:** GitHub Issues and Pull Requests welcome

<br>

### github.com/ScaleComputing/hypercore-ansible-dr-failover

<br>

*Scale Computing HyperCore — Ansible Collection*
*github.com/ScaleComputing/HyperCoreAnsibleCollection*
