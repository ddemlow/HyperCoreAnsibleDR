# Getting Started — No Ansible Experience Required

This guide walks you through everything from a blank laptop to a working automated DR
failover system. No prior Ansible experience needed.

---

## What is Ansible and why do we use it here?

**Ansible** is an automation tool. Instead of logging into a system and clicking buttons or
typing commands manually, you write a "playbook" — a YAML text file that describes what
you want to happen. Ansible reads the playbook and does the work for you.

For this DR failover system, the playbook does one job: every few minutes it checks whether
your primary HyperCore cluster is reachable. If it isn't, Ansible automatically clones your
replicated VMs on the DR cluster and powers them on.

The key advantage over a script: Ansible's HyperCore modules speak directly to the HyperCore
API using the same calls as the HyperCore UI. You don't need SSH access to VMs or special
permissions beyond a HyperCore admin account.

---

## What you need before starting

| Requirement | Notes |
|---|---|
| A computer to run Ansible from | Windows (WSL2), macOS, or Linux all work. This is your "controller". |
| Python 3.10 or newer | Usually pre-installed on macOS/Linux. Windows users need WSL2. |
| Network access to your DR cluster | The controller must be able to reach the DR cluster's management IP on port 443 (HTTPS). |
| Network access to ping your primary cluster's node IPs | The controller needs ICMP (ping) access to the primary cluster nodes. |
| A HyperCore admin username and password | For the DR cluster. |
| Replication already configured | VMs must already be replicating from primary → DR. |

---

## Step 1 — Set up Python (Windows users only)

If you are on Windows, install WSL2 first (Windows Subsystem for Linux). Open PowerShell
as Administrator and run:

```
wsl --install
```

Restart when prompted, then open the Ubuntu app from the Start menu. All remaining steps
run inside the WSL2 Ubuntu terminal.

macOS and Linux users: open your Terminal app and continue to Step 2.

---

## Step 2 — Check your Python version

```
python3 --version
```

You should see `Python 3.10.x` or higher. If you see an error or a version below 3.10:

- **macOS:** Install via [brew.sh](https://brew.sh): `brew install python3`
- **Ubuntu/Debian:** `sudo apt update && sudo apt install python3 python3-pip python3-venv`

---

## Step 3 — Download this project

If you have git installed:

```
git clone https://github.com/ScaleComputing/hypercore-ansible-dr-failover.git
cd hypercore-ansible-dr-failover
```

Or download the ZIP from GitHub (green **Code** button → **Download ZIP**), unzip it, and
open a terminal in the resulting folder.

---

## Step 4 — Create a Python virtual environment

A virtual environment keeps Ansible and its dependencies isolated from the rest of your
system. Think of it as a clean sandbox just for this project.

```
python3 -m venv .venv
```

Now activate it. **You need to do this every time you open a new terminal:**

```
# macOS / Linux / WSL2:
source .venv/bin/activate
```

Your terminal prompt will change to show `(.venv)` at the start, confirming it's active.

---

## Step 5 — Install Ansible

With the virtual environment active:

```
pip install -r requirements.txt
```

This installs `ansible-core`. It will take about a minute. When it finishes, verify:

```
ansible --version
```

You should see something like `ansible [core 2.16.x]`.

---

## Step 6 — Install the HyperCore Ansible collection

Ansible's functionality is extended by "collections" — packages of modules written for
specific platforms. The Scale Computing HyperCore collection gives Ansible the ability to
talk to HyperCore clusters.

```
ansible-galaxy collection install -r requirements.yml
```

You should see `scale_computing.hypercore` installed successfully.

---

## Step 7 — Create your inventory file

The inventory tells Ansible which HyperCore cluster to connect to. Copy the example:

```
cp inventory/inventory.example.yml inventory/inventory.yml
```

Open `inventory/inventory.yml` in a text editor and replace the example values:

```yaml
all:
  hosts:
    10.0.0.100:            # ← replace with your DR cluster's management IP
      scale_user: admin    # ← replace with your HyperCore username
      scale_pass: "admin"  # ← replace with your HyperCore password
```

**Important:** `inventory/inventory.yml` is excluded from git (via `.gitignore`) so your
password will not accidentally be committed to source control.

> **Tip — keeping passwords secure:**
> For a production setup, store the password in Ansible Vault instead of plain text.
> See [Ansible Vault basics](#ansible-vault-basics-optional) at the end of this guide.

---

## Step 8 — Test connectivity to your DR cluster

Before running the full playbook, verify that Ansible can reach your DR cluster:

```
ansible -i inventory/inventory.yml all -m scale_computing.hypercore.cluster_info
```

You should see your cluster's name and ICOS version in the output. If you see a connection
error, check:

- Is the DR cluster IP correct in your inventory?
- Can you reach `https://<DR cluster IP>` from your controller? Try opening it in a browser.
- Are the username and password correct?

---

## Step 9 — Understand the key variables

Open `automated_dr_failover.yml` in a text editor and find the `vars:` section near the top.
These are the settings you are most likely to want to change:

```yaml
vars:
  source_cluster_name: ""   # Leave empty to monitor all remote clusters,
                             # or set to e.g. "primary-site" to monitor only that one.

  ping_retries: 3            # How many times to retry pinging a primary node.
                             # More retries = more confidence, but slower detection.
                             # 3 is a good starting point.

  ping_delay: 5              # Seconds between ping retries.

  dr_vm_tag: "DR-failover"  # Tag added to cloned VMs. Also used to detect if
                             # failover has already happened (idempotency).

  dr_exclude_tag: ""         # Any VM on the DR cluster with this tag will be SKIPPED
                             # during failover. Useful for infrastructure VMs like DNS
                             # or NTP servers that you don't want auto-cloned.

  http_check_host: ""        # Optional. If set, the playbook checks this IP/hostname
                             # on http_check_port. If it responds, failover is aborted
                             # (the application is still alive at the primary site).

  http_check_port: 80        # TCP port for the above check.
```

You can also override any variable at runtime without editing the file:

```
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  -e "ping_retries=5 dr_exclude_tag=dr-exclude"
```

---

## Step 10 — Do a dry run

Run the playbook while your primary cluster is **online**. It should detect the primary is
reachable and exit without doing anything:

```
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml -v
```

Expected output:
```
TASK [[1/6] Exit — all monitored clusters are ESTABLISHED (no failover needed)]
```

The `-v` flag means "verbose" — it shows more detail about what each step is doing.
Use `-vv` or `-vvv` for even more detail when troubleshooting.

---

## Step 11 — Run the automated tests

The project includes automated tests that verify the decision logic using mock data. They
don't need a live cluster and are a good way to confirm everything is installed correctly:

```
ansible-playbook tests/test_failover_logic.yml
```

All 15 tasks should show `PASS` and the play should complete with `failed=0`.

---

## Step 12 — Schedule the playbook

For the playbook to be useful as a DR automation tool, it needs to run automatically on a
schedule. Every 5 minutes is a typical interval — this means your maximum detection-to-
failover time is around 5–6 minutes.

### Option A: cron (simple, runs on your controller machine)

Open the crontab editor:

```
crontab -e
```

Add this line (adjust the path to match where you cloned the project):

```
*/5 * * * * cd /home/youruser/hypercore-ansible-dr-failover && \
  .venv/bin/ansible-playbook -i inventory/inventory.yml \
  automated_dr_failover.yml >> /var/log/dr-failover.log 2>&1
```

This runs the playbook every 5 minutes and appends output to a log file.

Check it is working after a few minutes:

```
tail -f /var/log/dr-failover.log
```

### Option B: AWX or Ansible Automation Platform (recommended for production)

AWX is the open-source web interface for Ansible. It provides scheduling, logging, role-based
access control, and notifications. If your organization already has AWX or Ansible Automation
Platform:

1. Add this repository as a **Project** (point it at your git URL)
2. Create an **Inventory** with your DR cluster host
3. Create a **Job Template** pointing to `automated_dr_failover.yml`
4. Add a **Schedule** to the Job Template: every 5 minutes

AWX stores logs for every run, making it easy to see exactly what happened and when.

---

## Understanding the output

When the playbook runs, each task shows one of these statuses:

| Status | Color | Meaning |
|---|---|---|
| `ok` | Green | Task ran successfully, nothing changed |
| `changed` | Yellow/Orange | Task ran and made a change (e.g., cloned a VM) |
| `skipped` | Cyan | Task was skipped because its `when` condition was false |
| `failed` | Red | Task failed — look at the error message for details |

At the end of the run, the **PLAY RECAP** line summarizes the counts.

A normal healthy run (primary is up) looks like:

```
PLAY RECAP *****************************************************
10.0.0.100 : ok=4  changed=0  unreachable=0  failed=0  skipped=0
```

A successful failover run looks like:

```
PLAY RECAP *****************************************************
10.0.0.100 : ok=14 changed=6  unreachable=0  failed=0  skipped=2
```

(`changed=6` means 6 VMs were either cloned or powered on.)

---

## Troubleshooting common problems

**`MODULE FAILURE` or `Connection refused` when testing connectivity**

The controller cannot reach the DR cluster API. Check:
- The IP address in `inventory/inventory.yml` is correct
- You can open `https://<IP>` in a browser from the same machine
- No firewall is blocking port 443

**`Authentication failed`**

The username or password in `inventory/inventory.yml` is wrong. Test by logging into the
HyperCore UI with the same credentials.

**`No remote cluster connections found`**

The DR cluster has no replication connections configured. Log into the HyperCore UI on the
DR cluster and verify that a remote cluster connection exists under
*Data Protection → Remote Clusters*.

**Primary cluster shows `ESTABLISHED` but you know it's offline**

The replication connection status may take a few minutes to update after a failure. Wait
2–3 minutes and re-run. You can also check the current status:

```
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.remote_cluster_info
```

**Playbook runs but no VMs are cloned**

Check that VMs on the DR cluster have a `replication_source_vm_uuid`. Run:

```
ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.vm_info \
  | grep replication_source_vm_uuid
```

Any VM with a non-empty UUID is a replication target.

---

## Ansible Vault basics (optional)

Storing plain-text passwords in `inventory/inventory.yml` is fine for a lab, but not ideal
for production. Ansible Vault encrypts sensitive values.

Encrypt your password:

```
ansible-vault encrypt_string 'YourPasswordHere' --name 'scale_pass'
```

Enter a vault password when prompted. Copy the output into your inventory:

```yaml
all:
  hosts:
    10.0.0.100:
      scale_user: admin
      scale_pass: !vault |
        $ANSIBLE_VAULT;1.1;AES256
        38326539383731306566313930386164...
```

Run the playbook with the vault password:

```
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  --ask-vault-pass
```

Or store the vault password in a file (not committed to git):

```
echo 'YourVaultPassword' > vault_pass.txt
chmod 600 vault_pass.txt
ansible-playbook -i inventory/inventory.yml automated_dr_failover.yml \
  --vault-password-file vault_pass.txt
```

---

## Next steps

- Read `README.md` for a full reference of all variables and options
- Read `MANUAL_TEST_PLAN.md` to validate your setup with a controlled failover test
- Review `IMPROVEMENTS.md` for ideas on possible future enhancements
- Open a GitHub Issue if you run into problems or have feature requests
