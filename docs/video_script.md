# Video Script — Automated DR Failover with HyperCore and Ansible
## Getting Started Guide

**Format:** AI presenter + screen recording overlays + slide text
**Recommended tools:** Synthesia or HeyGen (AI presenter), or ElevenLabs (voiceover) + Canva/PowerPoint (slides)
**Total estimated runtime:** ~8 minutes
**Audience:** HyperCore administrators with no prior Ansible experience

---

### Production notes

- Use a professional/neutral AI avatar or voiceover
- Slides: dark background, Scale Computing brand colors if available
- Terminal recordings: use a large font (18pt+), light-on-dark theme
- Each scene has: [SLIDE TEXT] for on-screen text, [VISUAL] for what to show,
  and NARRATOR: for the spoken words
- Pause 1 second between sentences when recording voiceover

---

## SCENE 1 — Title (0:00–0:20)

**[SLIDE TEXT]**
```
Automated DR Failover
for Scale Computing HyperCore

Getting Started with Ansible
```

**[VISUAL]** Title slide with Scale Computing logo. Subtle animation of a server rack icon.

**NARRATOR:**
Your HyperCore replication is running. Your VMs are protected. But if the primary site goes
down tonight, someone still has to manually clone those VMs and power them on.

In this video, we're going to automate that entire process — so failover happens
automatically, whether it's 2pm or 2am.

---

## SCENE 2 — What we're building (0:20–1:00)

**[SLIDE TEXT]**
```
What this does

Every 5 minutes, automatically:
  ✓ Check if primary cluster is reachable
  ✓ If not → clone all replicated VMs on DR cluster
  ✓ Power them on, preserving MAC addresses
  ✓ If primary is fine → exit silently, do nothing
```

**[VISUAL]** Simple diagram: two HyperCore clusters (Primary → DR arrow for replication),
with a laptop icon labeled "Ansible controller" pointing at the DR cluster.

**NARRATOR:**
Here's what we're building. An Ansible playbook that runs every 5 minutes on a schedule.
Each run checks whether the primary cluster is reachable. If it is — and it will be most of
the time — the playbook exits in seconds without touching anything.

If the primary is genuinely unreachable — confirmed by checking the replication connection
status AND pinging each node IP — the playbook clones all your replicated VMs on the DR
cluster and powers them on automatically.

No manual steps. No waiting for someone to notice the outage.

---

## SCENE 3 — What you need (1:00–1:35)

**[SLIDE TEXT]**
```
Prerequisites

□ A computer to run Ansible from (Windows/Mac/Linux)
□ Python 3.10 or newer
□ Network access to your DR cluster (port 443)
□ Ability to ping your primary cluster's node IPs
□ HyperCore admin credentials for the DR cluster
□ Replication already configured in HyperCore
```

**[VISUAL]** Checklist slide, items appear one at a time.

**NARRATOR:**
Before we start, let's check what you need.

You need a computer to run Ansible from — this is called the Ansible controller. It can be
Windows with WSL2, a Mac, or any Linux machine. You need Python 3.10 or newer, which we'll
check in a moment.

The controller needs network access to your DR cluster on port 443 — the same port you use
for the HyperCore web interface. And it needs to be able to ping the node IPs of your primary
cluster, so it can verify the primary is actually down.

Finally, you need HyperCore admin credentials and replication already set up between your
primary and DR clusters.

---

## SCENE 4 — Installing Ansible (1:35–3:10)

**[SLIDE TEXT]**
```
Step 1: Install Ansible

# Check Python version
python3 --version

# Download the project
git clone https://github.com/ScaleComputing/hypercore-ansible-dr-failover.git
cd hypercore-ansible-dr-failover

# Create a virtual environment
python3 -m venv .venv
source .venv/bin/activate    ← run this every time you open a new terminal

# Install Ansible
pip install -r requirements.txt

# Install the HyperCore collection
ansible-galaxy collection install -r requirements.yml
```

**[VISUAL]** Terminal window showing each command being typed and its output.
Highlight the `(.venv)` prompt indicator after activation.

**NARRATOR:**
Let's get Ansible installed. Open your terminal and check your Python version first.
You should see 3.10 or higher.

Next, clone the project from GitHub and change into the project directory.

Now we'll create a virtual environment. Think of this as a clean sandbox that keeps Ansible
isolated from the rest of your system. Run the `source` command to activate it — your prompt
will change to show `.venv` in parentheses. You'll need to run this activation command each
time you open a new terminal.

Install Ansible with pip using the requirements file. This takes about a minute.

Finally, install the Scale Computing HyperCore Ansible collection. This is the package that
gives Ansible the ability to talk to HyperCore clusters directly.

When both finish, run `ansible --version` to confirm Ansible is ready.

---

## SCENE 5 — Configure your inventory (3:10–4:15)

**[SLIDE TEXT]**
```
Step 2: Configure your inventory

# Copy the example
cp inventory/inventory.example.yml inventory/inventory.yml

# Edit with your DR cluster details:
all:
  hosts:
    10.0.0.100:              ← your DR cluster IP
      scale_user: admin      ← your HyperCore username
      scale_pass: "admin"    ← your HyperCore password
```

**[VISUAL]** Terminal showing the copy command, then a text editor (VS Code or nano) opening
the file. Highlight the three lines that need to be changed.

**NARRATOR:**
Now configure your inventory. This is the file that tells Ansible which HyperCore cluster
to connect to.

Copy the example file and open it in a text editor. You only need to change three things:
the DR cluster's management IP address, your HyperCore username, and your password.

Note that this file is excluded from git by default — your credentials won't accidentally
end up in source control.

Once you've saved the file, let's test the connection:

---

## SCENE 6 — Test connectivity (4:15–4:50)

**[SLIDE TEXT]**
```
Step 3: Test connectivity

ansible -i inventory/inventory.yml all \
  -m scale_computing.hypercore.cluster_info
```

**Expected output:**
```
  record:
    name: my-dr-cluster
    icos_version: 9.4.1.xxxxx
    uuid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

**[VISUAL]** Terminal showing the command and successful output. Highlight the cluster name
and ICOS version in the response.

**NARRATOR:**
Run this command to verify Ansible can reach your DR cluster. You should see your cluster's
name and ICOS version in the output.

If you see a connection error, double-check the IP address and try opening
`https://your-cluster-ip` in a web browser from the same machine. If the browser can reach
it, Ansible should be able to as well.

---

## SCENE 7 — First dry run (4:50–5:35)

**[SLIDE TEXT]**
```
Step 4: First run (primary cluster is online)

ansible-playbook -i inventory/inventory.yml \
  automated_dr_failover.yml -v

Expected:
  [1/6] Exit — all monitored clusters are ESTABLISHED
  ↳ Primary is up. Nothing to do. ✓
```

**[VISUAL]** Terminal showing the playbook run. Highlight the ESTABLISHED exit task in green.
Show the PLAY RECAP at the bottom: `ok=4 changed=0 failed=0`.

**NARRATOR:**
Run the playbook for the first time while your primary cluster is still online.

The `-v` flag means verbose, which shows more detail about what each step is doing.

With the primary healthy, the playbook should exit almost immediately at Phase 1 with the
message "all monitored clusters are ESTABLISHED." No VMs are touched.

This is the normal steady-state behavior. Every 5 minutes: run, check, exit. Quiet and
harmless until you actually need it.

---

## SCENE 8 — Schedule it (5:35–6:30)

**[SLIDE TEXT]**
```
Step 5: Schedule the playbook

# Open crontab editor
crontab -e

# Add this line (adjust path for your system):
*/5 * * * * cd /home/user/hypercore-ansible-dr-failover && \
  .venv/bin/ansible-playbook \
  -i inventory/inventory.yml \
  automated_dr_failover.yml \
  >> /var/log/dr-failover.log 2>&1

# Verify it's running:
tail -f /var/log/dr-failover.log
```

**[VISUAL]** Terminal showing crontab -e, then the cron entry being added.
Then show `tail -f` with a few log lines scrolling by.

**NARRATOR:**
The last step is putting the playbook on a schedule. We'll use cron — the built-in Linux
task scheduler.

Run `crontab -e` to open the cron editor. Add the line shown here, adjusting the path to
match where you cloned the project.

The `*/5 * * * *` at the beginning means "every 5 minutes." The double-arrow at the end
redirects output to a log file so you can see what happened on each run.

Save and close the editor. After 5 minutes, use `tail -f` to watch the log file. You should
see the playbook running and exiting cleanly each time.

That's it — your automated DR is live.

---

## SCENE 9 — What a real failover looks like (6:30–7:20)

**[SLIDE TEXT]**
```
What failover looks like

[1/6] Disconnected cluster(s) detected — continuing checks
[2/6] All primary node IPs unreachable after 3 retries
[3/6] DR cluster nodes healthy ✓
[4/6] Found 3 replicated VM(s) to fail over
[5/6] Cloning: web-01, db-01, app-server
[6/6] Powering on cloned VMs
      ✓ DR FAILOVER COMPLETE
```

**[VISUAL]** Animated terminal output showing each phase completing in sequence.
Use green checkmarks for passed phases, and highlight the "DR FAILOVER COMPLETE" summary.

**NARRATOR:**
When a real failover occurs, here's what you'll see in the log.

Phase 1 detects the primary cluster's replication connection is no longer established.
Phase 2 confirms this by pinging each primary node IP and getting no response after retries.
Phase 3 verifies your DR cluster is healthy before proceeding.
Phase 4 finds your replicated VMs — in this example, three of them.
Phase 5 clones each one, preserving their MAC addresses for network continuity.
Phase 6 powers them on.

The whole process typically takes under two minutes from detection to VMs running.

---

## SCENE 10 — Key settings to know (7:20–7:50)

**[SLIDE TEXT]**
```
Key settings (in automated_dr_failover.yml)

ping_retries: 3       # how many times to retry pinging primary nodes
ping_delay: 5         # seconds between retries
dr_exclude_tag: ""    # set to skip infrastructure VMs (e.g. "dr-exclude")
http_check_host: ""   # optional app-alive check — if it responds, abort failover
source_cluster_name: ""  # monitor a specific named cluster, or all (default)
```

**[VISUAL]** Code view of the vars section in the playbook, each variable highlighted in turn.

**NARRATOR:**
A few settings worth knowing.

`ping_retries` and `ping_delay` control how patient the playbook is before deciding the
primary is down. More retries means higher confidence, but slightly slower detection.

`dr_exclude_tag` lets you skip specific VMs — useful for infrastructure VMs like DNS servers
that run independently at both sites and shouldn't be cloned during failover.

`http_check_host` adds an extra signal: if a specific application endpoint at your primary
site is still responding, failover is aborted even if the replication link looks broken.

---

## SCENE 11 — Wrap up (7:50–8:10)

**[SLIDE TEXT]**
```
You're set up. What's next?

→ MANUAL_TEST_PLAN.md  — validate with a controlled failover test
→ README.md            — full variable reference
→ IMPROVEMENTS.md      — planned enhancements

github.com/ScaleComputing/hypercore-ansible-dr-failover
```

**[VISUAL]** Final slide with project GitHub URL and Scale Computing logo.

**NARRATOR:**
You now have automated DR failover running on a schedule. The primary stays healthy, nothing
happens. The primary goes down, your VMs come up on DR — automatically.

When you're ready to validate the setup, follow the manual test plan in `MANUAL_TEST_PLAN.md`
for a step-by-step controlled failover test.

The full README has a complete variable reference. And check `IMPROVEMENTS.md` for planned
features including startup ordering, health checks, and failback automation.

Thanks for watching.

---

## Production checklist

Before recording or generating the video:

- [ ] Replace placeholder cluster IPs (`10.0.0.100`) with realistic-looking demo values
- [ ] Record or simulate terminal output for scenes 4–8 (use `asciinema` or a screen recorder)
- [ ] Verify all commands match the current version of the playbook
- [ ] Add company intro/outro bumpers if publishing on a branded channel
- [ ] Add captions/subtitles (auto-generate in YouTube or Synthesia, then review)
- [ ] Review runtime — target 8 minutes; trim narration if over 10 minutes

## Recommended AI video tools

| Tool | Best for | Notes |
|---|---|---|
| [Synthesia](https://synthesia.io) | AI avatar presenter + slides | Upload slide images, paste narration per scene |
| [HeyGen](https://heygen.com) | AI avatar with screen recording | Good for mixing presenter with terminal demos |
| [ElevenLabs](https://elevenlabs.io) | Voiceover audio only | Pair with Canva or PowerPoint for slides |
| [Descript](https://descript.com) | Edit video by editing transcript | Good if you have a real screen recording to edit |
