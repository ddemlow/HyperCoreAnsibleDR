# Never Wake Up to a Dead Site: Automated DR Failover for HyperCore with Ansible

Disaster recovery has a reputation problem. Every organization has a DR plan. Far fewer have
a DR plan that actually works at 2am when the on-call engineer is half asleep and the primary
site is completely dark.

The gap between "we have replication set up" and "our applications failed over automatically
while everyone slept" is wider than most people realize. This post walks through an Ansible
playbook that closes that gap for Scale Computing HyperCore environments.

## The Problem with Manual DR

HyperCore's built-in replication is excellent. Snapshots of your VMs replicate continuously
to a DR cluster at another site. If your primary site burns down, the data is safe.

But data being safe and applications being available are two different things. After a failure,
someone still has to:

1. Notice the primary is down (not always immediate)
2. Confirm it's a real outage and not a blip
3. Log in to the DR cluster
4. Find the replicated VMs
5. Clone each one manually
6. Power them on in the right order
7. Verify applications are responding

In a real outage, under pressure, at an odd hour, with adrenaline running — this process
takes time. Every minute of delay is downtime your users feel.

## A Better Approach: Let Ansible Do It

The `automated_dr_failover` playbook runs on a schedule (every 5 minutes is typical) against
your DR cluster. On each run it asks a simple question: *should I fail over right now?*

Most of the time the answer is no, and the playbook exits in seconds without touching anything.
When the answer is yes — primary is genuinely unreachable, DR cluster is healthy, no failover
has already happened — it acts.

The whole decision process takes under a minute. VMs are cloned and running before most people
would have even finished their first cup of coffee.

## How the Detection Logic Works

Getting DR automation right means being careful about *false positives* — triggering a failover
when the primary is actually fine. The playbook uses multiple independent signals before acting:

**Signal 1: Replication connection status**

HyperCore tracks the health of its replication link between clusters. The playbook checks this
first via the `remote_cluster_info` module. If the connection shows `ESTABLISHED`, the primary
is still reachable and the playbook exits immediately — no further checks needed.

**Signal 2: Node IP ping**

The replication connection record includes the IP addresses of every node in the primary cluster.
The playbook pings each one with configurable retries (default: 3 attempts, 5 second delay).
If *any* node responds, the primary is still up. This catches scenarios where the replication
link is flapping but the cluster is actually healthy.

**Signal 3: Application-alive check (optional)**

For an extra layer of confidence, you can configure an application endpoint at the primary site.
If that TCP port responds, the failover is aborted — even if cluster connectivity looks broken.
This is useful when you have a known-healthy resource at the primary site (a load balancer health
endpoint, a management VM) that gives you high confidence the site is actually down.

Only when all configured signals agree that the primary is unreachable does the playbook proceed.

**Signal 4: DR cluster health**

Before touching anything, the playbook verifies that the DR cluster itself is healthy by pinging
all local node IPs via the `node_info` module. A half-degraded DR cluster is not the right
place to start a failover.

## Idempotency: Safe to Run Repeatedly

A scheduled playbook that modifies infrastructure needs to be safe to run over and over. The
playbook checks for VMs already tagged with `DR-failover` before doing anything. If they exist,
it reports the situation and exits cleanly. Re-runs after a failover are no-ops.

This makes it safe to leave the schedule running indefinitely — before, during, and after a
real event. When the primary comes back and replication re-establishes, the schedule keeps
running harmlessly.

## The Failover

Once the decision to fail over is made, the playbook:

1. Queries all VMs on the DR cluster for `replication_source_vm_uuid` — a non-empty UUID
   identifies a VM that is a replication target from the primary
2. Filters out any VMs tagged with `dr-exclude` (useful for infrastructure VMs that run
   independently at both sites)
3. Clones each replicated VM, preserving its MAC address so network identity is maintained
4. Powers on each cloned VM

MAC address preservation is important. It means the cloned VMs come up with the same network
identity as the originals — DHCP leases, ARP caches, and anything else that keys off MAC
addresses will work without reconfiguration.

## Setting It Up

### 1. Install dependencies

```
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure inventory

Create `inventory/inventory.yml` from the provided example. The DR cluster IP or FQDN is
the host — the playbook uses it as the API endpoint:

```yaml
all:
  hosts:
    10.0.0.100:
      scale_user: admin
      scale_pass: "{{ vault_scale_pass }}"
```

### 3. Schedule it

**Cron (every 5 minutes):**

```
*/5 * * * * cd /opt/hypercore-ansible-dr && \
  .venv/bin/ansible-playbook -i inventory/inventory.yml \
  automated_dr_failover.yml >> /var/log/dr-failover.log 2>&1
```

**AWX / Ansible Automation Platform:**

Create a Job Template pointing to this playbook and set the schedule to run every 5 minutes.
The playbook is idempotent so there's no risk in running it frequently.

## Tuning for Your Environment

All detection parameters are variables with sensible defaults:

```yaml
ping_retries: 3          # more retries = higher confidence, longer detection time
ping_delay: 5            # seconds between retries
dr_vm_tag: "DR-failover" # change if you have existing VMs with this tag
dr_exclude_tag: ""       # set to skip infrastructure VMs (e.g., "dr-exclude")
http_check_host: ""      # set to an app endpoint at primary for extra confirmation
source_cluster_name: ""  # set to monitor a specific named cluster; default monitors all
```

For a conservative environment where false positives are very costly, increase `ping_retries`
to 5 or 6. For environments where RTO (recovery time objective) is tight, reduce to 2.

## Multiple Source Clusters

The playbook supports environments where multiple primary clusters replicate to the same DR
cluster. It checks the replication connection status for all remote clusters (or a named one
if `source_cluster_name` is set), pings all disconnected nodes, and proceeds only if all
checked nodes are unreachable.

One known limitation: the HyperCore API on the target cluster doesn't expose which source
cluster each replicated VM came from. If you have two source clusters and only one goes down,
all replicated VMs will be failed over. The workaround is to tag source VMs with their
cluster name before setting up replication, then use `dr_exclude_tag` to limit scope.

## What's Next

The current playbook handles the most common scenario well: detect failure, clone, power on.
The next planned enhancements focus on making the recovered environment production-ready faster:

- **Startup ordering** — power on database VMs before app servers before web tier, with
  configurable delays between groups
- **Post-failover health checks** — TCP verify each VM's application port before declaring
  the failover complete
- **Notifications** — Slack, Teams, or email alert the moment failover fires (or is aborted)
- **Replication lag monitoring** — surface RPO (recovery point objective) before committing
  to failover so operators know the data loss window
- **Failback automation** — return VMs to primary once it's restored

## The Bigger Picture

Replication gives you the data. Automation gives you the recovery. Together they turn a DR
plan from a document that lives in a drawer into a system that actually works when you need it.

The playbook is open source under the GPL-3.0 license and available at
[github.com/ScaleComputing/hypercore-ansible-dr-failover](https://github.com/ScaleComputing/hypercore-ansible-dr-failover).
Contributions, bug reports, and feature requests are welcome via GitHub Issues.

---

*Scale Computing HyperCore Ansible Collection documentation:
[github.com/ScaleComputing/HyperCoreAnsibleCollection](https://github.com/ScaleComputing/HyperCoreAnsibleCollection)*
