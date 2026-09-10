# Slurm Mastery Plan (with AWS)

**Owner:** Shashwat
**Goal:** Go from "I integrate with Slurm" to "I can explain, tune, operate, and extend Slurm" — plus real AWS infrastructure fluency. Target: credible depth in AI-infra interviews and technical discussions.
**Started:** _(fill in)_
**Environment:** AWS Free account plan, 3–4 × t3.micro-class instances, **Rocky Linux 9** (RHEL-compatible — what real HPC clusters run)

---

## How to use this document

1. Work phases in order. Each phase has **Goal → Steps → Problems you WILL hit → Exit criteria → Interview line**.
2. After every session, update the tracker below and add notes in the Session Log at the bottom.
3. **The "Problems you WILL hit" sections are the actual curriculum.** Anyone can copy-paste a working config. The expertise is in knowing what breaks, why, and how you diagnosed it. Do not skip a problem because it didn't happen to you — cause it on purpose.
4. Rule: never move to the next phase until the exit criteria pass on a *fresh rebuild*, not just on the cluster you fumbled into working.

---

## Progress Tracker

| # | Phase | Est. time | Status | Date done |
|---|-------|-----------|--------|-----------|
| 0 | AWS account hygiene & safety rails | 1 h | ☐ Not started | |
| 1 | First VM: launch, SSH, cloud-init, AMI | 2 h | ☐ Not started | |
| 2 | Cluster foundations: hosts, users, munge, NFS | 3 h | ☐ Not started | |
| 3 | Build Slurm from source + first job | 4 h | ☐ Not started | |
| 4 | Breakage drills: diagnose like an admin | 3 h | ☐ Not started | |
| 5 | Accounting: slurmdbd, sacctmgr, sacct | 3 h | ☐ Not started | |
| 6 | Scheduling core: priority, fairshare, backfill | 6 h | ☐ Not started | |
| 7 | Resource isolation: cgroups, GRES, fake GPUs | 4 h | ☐ Not started | |
| 8 | Automation: cloud-init / Ansible / Terraform | 5 h | ☐ Not started | |
| 9 | **Elastic cloud nodes (power_save + EC2)** | 6 h | ☐ Not started | |
| 10 | slurmrestd + JWT auth | 3 h | ☐ Not started | |
| 11 | Containers & MPI: Enroot/Pyxis, PMIx | 4 h | ☐ Not started | |
| 12 | Slinky: Slurm on Kubernetes | 6 h | ☐ Not started | |
| 13 | Artifacts & interview narrative | 4 h | ☐ Not started | |

**Status values:** ☐ Not started · ◐ In progress · ☑ Done · ⚠ Blocked

**Total: ~54 hours.** At 5 h/week that's roughly 11 weeks. Phases 0–6 are the foundation and matter most; 9 is your differentiator; 12 is the bonus that connects to your Kubernetes background.

---

## Phase 0 — AWS account hygiene & safety rails

**Goal:** Make the account safe and predictable before anything else exists. On the Free account plan you can't be billed, but credits deplete and the plan ends at 6 months or credit exhaustion — whichever comes first. Protect the runway.

### Steps
1. Billing console → check plan type, credit balance, and expiry date. Write them here: `Plan: Free  Credits: $120  Expires: ~March 2027 (182 days from 2026-09-10)`
2. Create an AWS Budget at $5 with an email alert. (This is also one of the onboarding activities that earns credits.)
3. Complete the other onboarding activities to earn up to $100 more.
4. Create an IAM user with admin rights and **stop using the root account.** Enable MFA on root.
5. Pick one region and stay in it (`ap-south-1` for Mumbai — lower latency from Pune). Note it: `Region: ____`
6. Create an EC2 key pair, download the `.pem`, `chmod 400` it.
7. Install and configure the AWS CLI locally with the IAM user's credentials. `aws sts get-caller-identity` should return your user.

### Problems you WILL hit
- **"Access denied" on the new IAM user.** Attached the policy to a group but the user isn't in it, or you're still using an old cached profile. Learn `aws configure list` and `--profile`.
- **Key pair permissions.** SSH refuses a `.pem` that's world-readable. Understand *why* — SSH won't trust a key anyone can read.
- **Region confusion.** Resources created in one region are invisible in another. Half of all "my instance disappeared" reports are this.

### Exit criteria
- [ ] Budget alert configured and confirmed by email
- [ ] IAM user works via CLI, root untouched and MFA-protected
- [ ] Credit balance and expiry date recorded above

### Interview line
Not one on its own — but "I ran my lab on a hard credit budget and automated teardown" is a credible cost-awareness signal, which infra teams care about more than candidates expect.

---

## Phase 1 — First VM: launch, SSH, cloud-init, AMI

**Goal:** Understand the EC2 primitives you'll be automating later. Don't skip to four instances; get one right first.

### Steps
1. Launch one **Rocky Linux 9** t3.micro. During launch, read every screen instead of clicking Next: AMI, instance type, key pair, network settings, storage.
   - Rocky AMIs come via AWS Marketplace; if the Free plan blocks the subscription, search Community AMIs for Rocky-9, or fall back to Amazon Linux 2023 (also RHEL-derived and `dnf`-based).
2. Note what got created *implicitly*: a VPC, subnet, route table, internet gateway, security group. Find each in the console. You cannot reason about Slurm networking later without this.
3. SSH in. Explore: `lsblk`, `free -h`, `nproc`, `cat /etc/os-release`, `curl http://169.254.169.254/latest/meta-data/` (instance metadata — useful later for cloud-init scripts).
4. Install something, create a file, then **stop** the instance (not terminate). Start it again. Note what changed: the public IP, and the private DNS name.
5. Terminate it. Check the EBS volume list — confirm the root volume went with it.
6. Relaunch, this time with a cloud-init **user-data** script that installs packages and creates users automatically. This is your first taste of "the cluster should be reproducible."
7. Create an **AMI** from that configured instance. You'll use it in Phase 2 so you configure once, not four times.

### Problems you WILL hit
- **SSH connection refused / timed out.** Almost always the security group has no inbound rule for port 22 from your IP, or you're on a subnet with no internet gateway route. Learn to distinguish "refused" (something answered, nothing listening) from "timed out" (packets dropped — firewall/SG/routing).
- **Public IP changed after stop/start.** Elastic IPs exist for this, but the better lesson: never hardcode IPs. This becomes critical in `slurm.conf`.
- **Wrong username.** Rocky AMIs use `rocky`, Amazon Linux uses `ec2-user`, Ubuntu uses `ubuntu`. `Permission denied (publickey)` often just means the wrong username.
- **`dnf` not `apt`.** Different package names throughout. Enable EPEL and CRB early: `sudo dnf install -y epel-release && sudo dnf config-manager --set-enabled crb`. Most "package not found" errors on RHEL-family are a missing repo, not a missing package.
- **Instance stuck in "pending" or fails status checks.** Read the system log in the console; usually a bad user-data script.
- **User-data ran but nothing happened.** Check `/var/log/cloud-init-output.log`. Silent failure is the default; that log is where the truth lives.

### Exit criteria
- [ ] Can launch, SSH, stop, start, terminate confidently
- [ ] A cloud-init user-data script that provisions a machine unattended
- [ ] A custom AMI created and launched from
- [ ] Can explain what a VPC, subnet, security group, and IGW each do

### Interview line
"I baked golden AMIs with cloud-init so cluster nodes were reproducible from scratch rather than hand-configured."

---

## Phase 2 — Cluster foundations: hosts, users, munge, NFS

**Goal:** Everything Slurm assumes is already true before you install Slurm. Most "Slurm problems" are actually problems in this layer.

### Steps
1. Launch 3 instances from your AMI: `ctl` (controller + db), `node1`, `node2`. Consider t3.small for `ctl` if credits allow — slurmctld + slurmdbd + MariaDB on 1 GB is tight.
2. Put all three in **one security group** and add a self-referencing rule: allow all traffic where the source is the security group itself. This is the idiomatic AWS way to let cluster members talk freely without opening anything to the internet.
3. Set hostnames properly (`hostnamectl set-hostname ctl`) and make them persist across reboot (`preserve_hostname: true` in `/etc/cloud/cloud.cfg`).
4. Populate `/etc/hosts` on all three with **private IPs** (or better, use private DNS names).
5. Create `slurm` and `munge` users with **identical UID/GID on every node**. Verify with `id slurm` on each.
6. Install munge everywhere. Generate one key on `ctl`, copy to all nodes, `chown munge:munge`, `chmod 400`, start `munged`.
7. Verify: `munge -n | ssh node1 unmunge` — must succeed.
8. Set up NFS: export `/home` and `/opt/slurm` from `ctl`, mount on both nodes. Slurm needs users to see the same filesystem paths everywhere.
9. Passwordless SSH between all nodes for your user (Slurm doesn't strictly need it, but MPI and your sanity do).

### Problems you WILL hit
- **`munge: Failed to access "/etc/munge/munge.key": Permission denied`** — wrong ownership or mode. Munge is deliberately paranoid.
- **`Invalid credential` / `Rewound credential`** — the classic: **clock skew** between nodes. Munge credentials embed a timestamp and expire. Install `chrony`/NTP. This is a real production failure mode and a great interview anecdote.
- **UID mismatch.** Jobs run as the wrong user, or files land with numeric owners. The reason production clusters use LDAP/SSSD. Cause this deliberately once: change a UID on one node and watch what breaks.
- **NFS mount hangs on boot.** `_netdev` and mount options matter; a hard NFS mount to a dead server hangs the machine. Learn `soft` vs `hard`.
- **Hostname resets after reboot** (cloud-init overwrites it) — hence step 3.
- **firewalld blocks everything.** Rocky ships firewalld enabled. Slurm needs 6817 (slurmctld), 6818 (slurmd), 6819 (slurmdbd), plus NFS and MySQL. Either add the ports properly (`firewall-cmd --permanent --add-port=6817/tcp`) or put the cluster interfaces in the `trusted` zone. Do NOT just `systemctl stop firewalld` — configuring it is the skill.
- **SELinux denies NFS home directories.** `setsebool -P use_nfs_home_dirs on`. When something fails inexplicably on RHEL-family, check `ausearch -m avc -ts recent` BEFORE blaming Slurm. Learning to read AVC denials is a genuine differentiator — most engineers just run `setenforce 0` and move on.

### Exit criteria
- [ ] `munge -n | ssh node1 unmunge` succeeds from every node to every node
- [ ] `id slurm` identical on all three
- [ ] Shared `/home` visible everywhere with correct ownership
- [ ] You can explain what munge actually does (shared-key auth for daemon-to-daemon messages) and why Slurm needs it

### Interview line
"Most Slurm outages I've debugged weren't Slurm — they were munge clock skew, UID drift, or a hung NFS mount."

---

## Phase 3 — Build Slurm from source + first job

**Goal:** Compile it yourself, configure it minimally, and run a job across two nodes.

### Steps
1. On `ctl`, install build deps:
   ```bash
   sudo dnf groupinstall -y "Development Tools"
   sudo dnf install -y munge-devel mariadb-devel pam-devel json-c-devel \
     http-parser-devel libyaml-devel libjwt-devel python3 rpm-build
   ```
   (EPEL + CRB must be enabled first — see Phase 1.)
2. Download the latest Slurm tarball from SchedMD, extract.
3. `./configure --prefix=/opt/slurm --sysconfdir=/etc/slurm --with-munge --enable-pam --with-jwt`
4. **Read the configure summary output carefully.** It lists which optional plugins were found and which were skipped. This is a map of Slurm's capability surface — screenshot it and go look up three things you don't recognize.
5. `make -j$(nproc) && sudo make install`. Since `/opt/slurm` is NFS-shared, the nodes get it for free.
   - **Do this once as an RPM build too:** `rpmbuild -ta slurm-*.tar.bz2`. That's how production RHEL clusters actually deploy Slurm, and produced RPMs are a better artifact than a source install. Worth mentioning in interviews.
6. Write a minimal `slurm.conf` (see appendix) and a minimal `cgroup.conf`. Same file on every node.
7. Create spool/log/state directories owned by `slurm`, with correct permissions.
8. Install systemd unit files from the source tree's `etc/` directory. Enable and start `slurmctld` on ctl, `slurmd` on nodes.
9. Verify: `sinfo` → nodes `idle`. Then `srun -N2 hostname`, then `sbatch --wrap="sleep 60"`, then `squeue`.

### Problems you WILL hit
- **Node stuck in `DOWN` or `UNKNOWN`.** Run `scontrol show node node1` and read `Reason=`. Then run `slurmd -D -vvvv` in the foreground on that node — this is the single most useful debugging command in Slurm.
- **`Node has low real memory` / `Low socket*core*thread count`.** Your `slurm.conf` claims resources the machine doesn't have. Fix with `slurmd -C` on the node, which prints exactly what Slurm detects. Set `RealMemory` below actual RAM (the OS needs some).
- **`Zero Bytes were transmitted or received` / `Unable to contact slurm controller`.** Ports 6817 (slurmctld) and 6818 (slurmd) blocked, or `SlurmctldHost` doesn't resolve. Comes back to Phase 2.
- **Config file drift.** You edited `slurm.conf` on ctl only. Slurm requires identical configs on all nodes (until you use configless mode — look that up as a bonus).
- **Permission errors on StateSaveLocation / SlurmdSpoolDir.** Wrong ownership. Slurm refuses to start and says so in the log; learn where the logs are.
- **Job stuck in `PD` with `Reason=Resources` or `PartitionNodeLimit`.** Read the reason code — Slurm almost always tells you exactly why, and learning to trust `squeue -o "%R"` is half the skill.

### Exit criteria
- [ ] `sinfo` shows both nodes idle
- [ ] `srun -N2 hostname` returns both hostnames
- [ ] A batch job runs and writes output to shared `/home`
- [ ] You can name what slurmctld, slurmd, and slurmdbd each do, and which one holds the queue

### Interview line
"I build Slurm from source, so I know which plugins are compiled in and what `configure` flags gate features like JWT auth or NVML GPU autodetect."

---

## Phase 4 — Breakage drills: diagnose like an admin

**Goal:** This is the phase that separates you from people who "have used Slurm." Do these deliberately. Snapshot your AMI first so recovery is cheap.

### Drills
| # | Break this | Expected symptom | What you must learn |
|---|-----------|------------------|---------------------|
| 1 | Corrupt `munge.key` on node2 | Node goes DOWN, auth errors in log | How auth failures present |
| 2 | Skew node2's clock by 10 min | Credential rejected | Why NTP is mandatory |
| 3 | `kill -9 slurmctld` with a job running | Controller down, job survives | Slurm's state recovery from StateSaveLocation |
| 4 | Delete StateSaveLocation contents | Queue lost on restart | Why this dir is sacred; backup strategy |
| 5 | Fill `/` on node1 | slurmd errors, node drains | Disk pressure and node health |
| 6 | Set `MaxTime=00:01:00`, run a long job | Job killed at limit | Timeouts, `--time`, grace periods |
| 7 | `scontrol update nodename=node1 state=drain reason=test` | Node drains, jobs finish then no new ones | DRAIN vs DOWN vs DRAINED semantics; how to resume |
| 8 | Submit a job asking for more memory than exists | Job pends forever | Reason codes, and why `PD` ≠ broken |
| 9 | Stop/start the EC2 instances | Private IP changes | Why you use DNS names, not IPs |
| 10 | Change `slurm.conf` on ctl only, `scontrol reconfigure` | Inconsistent behavior | Config distribution; try configless mode as the fix |

For each drill, write down: **symptom → where you looked → root cause → fix.** That written record is your interview material.

### Exit criteria
- [ ] All 10 drills done, each with written notes
- [ ] You reach for `scontrol show node`, `slurmd -D -vvv`, and the logs reflexively
- [ ] You can restore the cluster from broken to working without notes

### Interview line
Pick your best two drills and turn them into stories. "Tell me about a time you debugged something hard" is answerable from this phase alone.

---

## Phase 5 — Accounting: slurmdbd, sacctmgr, sacct

**Goal:** Turn on the database layer. Half of Slurm's real capability (fairshare, QOS, limits, chargeback) lives here.

### Steps
1. Install MariaDB on `ctl`. Create `slurm_acct_db`, a `slurm` DB user, grant privileges.
2. Tune MySQL for Slurm: `innodb_buffer_pool_size`, `innodb_lock_wait_timeout` — SchedMD's docs give recommended values. On a t3.micro, tune *down*.
3. Write `slurmdbd.conf` — **mode 600, owned by slurm** (it contains the DB password; slurmdbd refuses to start otherwise). Start slurmdbd before slurmctld.
4. Point `slurm.conf` at it: `AccountingStorageType=accounting_storage/slurmdbd`.
5. Register the cluster: `sacctmgr add cluster lab`.
6. Build an account hierarchy — e.g. `root` → `research` → `team-a`, `team-b`; add users to accounts with shares.
7. Run jobs from different accounts (`--account=`), then explore `sacct`, `sacct -j <id> --format=...`, `sreport cluster utilization`.
8. Create a QOS with limits (`sacctmgr add qos short MaxWall=00:10:00 MaxJobsPerUser=2`) and test that it enforces.

### Problems you WILL hit
- **slurmdbd won't start:** config file permissions, or MySQL not reachable, or the `slurm` DB user lacks privileges.
- **Jobs run but `sacct` shows nothing.** Accounting wasn't enabled when the job ran, or the cluster isn't registered in the DB.
- **"Invalid account or account/partition combination"** on submit — the user isn't associated with that account. Learn what an *association* is (cluster + account + user + partition); it's the core abstraction of Slurm accounting and it confuses everyone at first.
- **DB schema upgrade takes forever** on version bumps — real production concern; note it.

### Exit criteria
- [ ] Multi-level account hierarchy exists, visible with `sacctmgr show assoc`
- [ ] `sacct` shows completed jobs with CPU-time and memory stats
- [ ] A QOS limit demonstrably blocks a job
- [ ] You can explain an "association" without hesitating

### Interview line
"I've built account hierarchies with QOS-enforced limits, which is what makes multi-tenant GPU chargeback possible."

---

## Phase 6 — Scheduling core: priority, fairshare, backfill

**Goal:** THE phase. This is what "expert in Slurm" actually means. Budget the most time here.

### Steps
1. **Multifactor priority.** Set nonzero `PriorityWeightAge`, `PriorityWeightFairshare`, `PriorityWeightQOS`, `PriorityWeightPartition`, `PriorityWeightTRES`. Submit jobs and read `sprio -l`. Then compute a job's priority **by hand** from the weights and factors and confirm it matches. Do this until it's boring.
2. **Fairshare.** Give two accounts unequal shares. Run a lot of jobs from one. Watch `sshare -l` — RawUsage, EffectvUsage, FairShare — shift. Understand `PriorityDecayHalfLife` and `PriorityUsageResetPeriod`. Compare classic fairshare vs **fair tree** (`PriorityFlags=FAIR_TREE`) and be able to say why fair tree exists.
3. **Backfill.** Fill the cluster with a long job. Submit a big job that must wait, then a small short one behind it. Watch the small one run first *without delaying* the big one's reserved start. This is the single most important scheduler behavior in Slurm.
   - Tune `SchedulerParameters`: `bf_window`, `bf_resolution`, `bf_interval`, `bf_max_job_test`, `bf_continue`.
   - Learn to read `sdiag` cold: backfill cycle times, queue depth, last cycle, jobs tested. **`sdiag` is the tool that proves you actually tune schedulers.**
   - Deliberately misconfigure `bf_max_job_test` too low and observe starvation of deep-queue jobs.
4. **Why accurate `--time` matters.** Backfill only works if users declare realistic walltimes. Demonstrate it: run the same workload with everyone requesting max walltime vs accurate walltimes, and compare throughput.
5. **Preemption.** Configure `PreemptType=preempt/qos` with a high-priority QOS. Watch a low-priority job get suspended or requeued. Understand SUSPEND vs REQUEUE vs CANCEL trade-offs (suspend holds memory!).
6. **Reservations.** `scontrol create reservation ...` for maintenance windows and for dedicated user allocations.
7. **Job arrays and dependencies.** `sbatch --array=1-100`, `--dependency=afterok:<id>`. Understand why arrays are far cheaper for the scheduler than 100 separate jobs.
8. **Node selection.** `SelectType=select/cons_tres` and `CR_Core_Memory`: understand consumable resources, and how `--exclusive`, `--mem-per-cpu`, and `--cpus-per-task` interact.

### Problems you WILL hit
- **Priorities all identical.** You left `PriorityType=priority/basic` (FIFO). Switch to `priority/multifactor`.
- **Fairshare appears to do nothing.** Half-life too long for a lab; shorten it so effects are visible in minutes.
- **Backfill never triggers.** Jobs have no time limit, so Slurm can't project when resources free up. This is the #1 real-world backfill killer.
- **Preemption doesn't fire.** Preemption requires correct partition/QOS setup *and* `PreemptMode` compatible with your select plugin.
- **Suspended jobs cause OOM.** Suspend doesn't release memory. Real gotcha.

### Exit criteria
- [ ] You can hand-compute a job's priority and match `sprio`
- [ ] You can demonstrate backfill running a job out of order and explain the reservation guarantee
- [ ] You can read `sdiag` and say whether the scheduler is healthy
- [ ] You can explain fairshare decay to a non-expert in two sentences
- [ ] Written note: "before/after" of one backfill parameter change with `sdiag` numbers

### Interview line
This is your headline. "I tuned backfill parameters on a cluster and can show the `sdiag` before/after" is a sentence very few candidates can say.

---

## Phase 7 — Resource isolation: cgroups, GRES, fake GPUs

**Goal:** How Slurm actually confines a job — and how GPUs get allocated. Directly relevant to your VDI background and to AI infra.

### Steps
1. Enable `ProctrackType=proctrack/cgroup`, `TaskPlugin=task/cgroup,task/affinity`. Configure `cgroup.conf` with `ConstrainCores`, `ConstrainRAMSpace`, `ConstrainDevices`.
2. Run a job, find its cgroup under `/sys/fs/cgroup/`, and inspect the limits. Rocky 9 defaults to cgroup v2 — confirm with `stat -fc %T /sys/fs/cgroup/` (returns `cgroup2fs`) and note how it differs from v1 (Slurm's cgroup/v2 plugin).
3. Prove isolation: request 1 CPU, run a 4-thread process, observe throttling. Request 200 MB, allocate 500 MB, watch the OOM kill.
4. **Fake GPUs.** You have no GPUs, so simulate them: create dummy device files, declare `Gres=gpu:2` in `slurm.conf`, write `gres.conf` mapping names to files. Then submit with `--gres=gpu:1` and confirm `CUDA_VISIBLE_DEVICES` is set correctly and only the allocated device is visible.
5. Read the real-hardware path even though you can't run it: `AutoDetect=nvml`, MPS, MIG, `--gpus-per-task` vs `--gpus-per-node` vs `--gres`. Know the syntax cold — it comes up constantly in AI-infra interviews.
6. `pam_slurm_adopt`: make SSH into a compute node land the user inside their job's cgroup instead of escaping it. Excellent, underrated topic.
7. Read `topology.conf` and tree topology. Understand why placing a multi-node job under one leaf switch matters for NCCL all-reduce performance.

### Problems you WILL hit
- **cgroup v1 vs v2 mismatch.** Old docs assume v1 (RHEL 7/8 era). Rocky 9 is v2. Know which plugin you're on.
- **SELinux blocks cgroup/device access** in some configurations. Diagnose with `ausearch -m avc -ts recent`, don't reflexively disable.
- **`--mem` enforcement doesn't work** without `ConstrainRAMSpace=yes`.
- **GRES declared but not allocatable.** `gres.conf` and `slurm.conf` disagree, or the count doesn't match device files. Read the slurmd log.
- **`CUDA_VISIBLE_DEVICES` unset** — job asked for `--gres=gpu:1` but the GRES plugin isn't configured with device files, so no isolation happens.

### Exit criteria
- [ ] Can show a running job's cgroup and its enforced limits
- [ ] Fake GPU allocation works and sets the right env vars
- [ ] Can explain how Slurm hides non-allocated GPUs from a job
- [ ] Can explain topology-aware scheduling and why NCCL cares

### Interview line
"Slurm's GPU isolation is a device cgroup — the job physically cannot see GPUs it wasn't allocated. That's the mechanism `CUDA_VISIBLE_DEVICES` sits on top of."

---

## Phase 8 — Automation: rebuild the whole cluster from code

**Goal:** Turn your hand-built cluster into code. Solves the credit problem (destroy nightly, rebuild in 10 min) and becomes a portfolio artifact.

### Steps
1. **Terraform**: VPC, subnet, security group with self-reference, 3 EC2 instances, key pair, outputs with private DNS names.
2. **Ansible** (or cloud-init if you prefer simpler): users with fixed UIDs, munge key distribution, NFS, Slurm install, config templating from a single inventory.
3. Template `slurm.conf` so node names/counts come from the inventory rather than being hardcoded.
4. `terraform destroy` at the end of every session; `terraform apply` at the start. Target: cluster up and running jobs within 10 minutes of `apply`.
5. Put it in a public GitHub repo with a good README.

### Problems you WILL hit
- **Munge key in git.** Don't. Generate it at provision time or use SSM Parameter Store / Secrets Manager. Realizing this on your own is the lesson.
- **Race conditions:** slurmd starts before NFS is mounted, or before slurmctld exists. Ordering and retries matter.
- **Private DNS changes** on rebuild; templating must be dynamic.
- **Terraform state** management — keep it local for a lab but know why remote state exists.

### Exit criteria
- [ ] `terraform apply` + one Ansible run = working cluster, no manual steps
- [ ] Public repo with README, diagram, and usage instructions
- [ ] Full rebuild verified from scratch at least twice

### Interview line
"My lab cluster is fully IaC — destroy and rebuild in ten minutes. Here's the repo."

---

## Phase 9 — Elastic cloud nodes (power_save + EC2) ★ flagship

**Goal:** Make Slurm itself provision EC2 instances on demand. This is where your Slurm depth and your existing multi-cloud provisioning work at Syncious intersect, and it's the most interview-valuable thing in this document.

### Steps
1. Read Slurm's cloud/power-saving docs thoroughly (`power_save.html`, `elastic_computing.html`).
2. Configure `SuspendProgram`, `ResumeProgram`, `SuspendTime`, `ResumeTimeout`, `SuspendExcNodes`, and `TreeWidth`. Declare cloud nodes with `State=CLOUD`.
3. Write `ResumeProgram` as a script that calls the EC2 API (boto3) to launch instances from your AMI, tag them with the Slurm node name, and register their addresses back into Slurm (`scontrol update nodename=X nodeaddr=Y`).
4. Write `SuspendProgram` to terminate them.
5. Test: submit a job to an empty cloud partition, watch nodes go `POWERING_UP` → `IDLE` → allocated → `POWERING_DOWN` → `IDLE~`.
6. Handle the failure paths: instance fails to boot within `ResumeTimeout` (node gets marked DOWN — why, and how to recover), API rate limits, partial launches, orphaned instances after a controller restart.
7. Compare and write up: **this approach vs Kubernetes cluster-autoscaler/Karpenter with Slinky.** Who owns elasticity, and what breaks in each model.

### Problems you WILL hit
- **Nodes marked DOWN after resume.** Boot took longer than `ResumeTimeout`, or slurmd didn't register. Read `slurmctld` log for `Power up/down` messages.
- **IP address mismatch.** Cloud nodes get new IPs each launch; you must update `NodeAddr`, or use DNS.
- **Orphaned instances** costing credits after a failed suspend. Build a reaper script — and note that this exact bug costs real companies real money.
- **IAM permissions** on the controller: it needs `ec2:RunInstances`, `ec2:TerminateInstances`, `ec2:DescribeInstances`. Use an instance profile, not access keys on disk.
- **Thundering herd:** 50 pending jobs trigger 50 launches. Learn `ResumeRate` and `SuspendRate`.

### Exit criteria
- [ ] A job submitted to an empty partition causes a real EC2 instance to boot, run it, and terminate
- [ ] Failure paths handled: timeout, orphan cleanup, rate limiting
- [ ] Written comparison of Slurm-native elasticity vs Kubernetes autoscaling

### Interview line
The strongest one you'll have: "I've implemented cloud bursting both ways — through Slurm's power-save plugin driving EC2 directly, and through Kubernetes autoscaling under Slinky. The trade-off is who owns elasticity and where the failure modes surface."

---

## Phase 10 — slurmrestd + JWT auth

**Goal:** The API surface. Plays directly to your .NET/OAuth2/OIDC background — you'll recognize the patterns instantly.

### Steps
1. Enable JWT: `AuthAltTypes=auth/jwt`, generate the signing key, set `AuthAltParameters=jwt_key=...`.
2. Start `slurmrestd` (socket or TCP listen mode; understand the security difference and why you don't expose it naked).
3. Generate a token with `scontrol token`, call the API with `curl`, submit a job over REST, poll its status.
4. Read the OpenAPI spec it publishes. Note the versioned API paths and how they change across releases.
5. Write a small client (Python or C#) that submits and monitors jobs. Bonus: compare to how SyncHPC talks to Slurm today — this is a genuinely useful work insight, not just lab work.

### Problems you WILL hit
- **`slurmrestd` refuses to run as root or as SlurmUser** — deliberate; it wants a dedicated unprivileged user.
- **401s:** token expired (short default lifetime), or the wrong username in the header.
- **API version drift:** endpoint paths change between Slurm releases; pin your version.

### Exit criteria
- [ ] Job submitted and monitored entirely over REST with JWT auth
- [ ] Small working client committed to your repo
- [ ] Can explain the auth model: munge for internal daemon traffic, JWT for external API

### Interview line
"Slurm has two auth planes — munge between daemons, JWT for the REST API — and I've built clients against the latter."

---

## Phase 11 — Containers & MPI: Enroot/Pyxis, PMIx

**Goal:** How real AI workloads actually run under Slurm.

### Steps
1. Install Enroot and the Pyxis SPANK plugin. Import a container image, run `srun --container-image=...`.
2. Understand *why* HPC uses Enroot/Apptainer rather than Docker: unprivileged, no daemon, integrates with the job's cgroup.
3. Explore Slurm's native OCI support (`oci.conf`) as the alternative path.
4. Install OpenMPI with PMIx. Run a multi-node MPI hello-world under `srun`. Understand what PMIx does — process launch and wire-up — and why `mpirun` under Slurm is usually wrong.
5. Read how `torchrun` derives rank/world-size from `SLURM_PROCID`, `SLURM_NNODES`, `SLURM_NODELIST`. Write the batch script even without GPUs to run it on.
6. Learn what a SPANK plugin is (Pyxis is one) and read its source. Bonus: write a trivial one.

### Problems you WILL hit
- **PMIx version mismatch** between Slurm and MPI — a notorious, very real HPC failure. `srun --mpi=list` shows what's available.
- **Container can't see the shared filesystem** — mounts must be passed explicitly.
- **`mpirun` vs `srun`** double-launching processes or ignoring the allocation.

### Exit criteria
- [ ] A containerized job runs under `srun`
- [ ] Multi-node MPI job works via PMIx
- [ ] Can explain how `torchrun` and Slurm hand off to each other

### Interview line
"Distributed training under Slurm is really a PMIx/env-var handshake — Slurm sets rank and world size, torchrun consumes them, NCCL takes over."

---

## Phase 12 — Slinky: Slurm on Kubernetes

**Goal:** Connect Slurm to the Kubernetes half of your résumé. Only worth doing after Phases 1–7 are solid.

### Steps
1. Stand up a small Kubernetes cluster (kubeadm on EC2, or local kind/k3s to save credits).
2. Install the Slinky CRDs and slurm-operator via Helm.
3. Deploy a `SlurmCluster` + `NodeSet`. Watch pods come up; exec in and run `sinfo`.
4. Scale the NodeSet up and down. **Observe the drain-before-delete behavior** — that's the operator's core value.
5. Read the operator's Go source, specifically the reconcile loop. You now have the Slurm knowledge to understand what it's automating.
6. Read up on Slurm Bridge (the inverse: Slurm scheduling native K8s pods) and be able to describe the difference.
7. Write your comparison piece: Slurm vs Kueue vs Volcano vs Slinky. Gang scheduling, queueing, fairshare maturity, topology awareness, and the training-plus-inference-on-one-fleet argument.

### Problems you WILL hit
- **Image/GLIBC mismatches** if you customize the slurmd image.
- **Resource requests too large** for tiny nodes — pods stay Pending, which is itself the lesson about slurmd pods needing real capacity.
- **The operator owns `slurm.conf`** — hand-editing it inside a pod gets reverted on the next reconcile. Great illustration of declarative config.

### Exit criteria
- [ ] Slinky cluster running, job submitted, NodeSet scaled both directions
- [ ] Written comparison of Slurm vs K8s-native batch schedulers
- [ ] Can explain the operator pattern (CRD + reconcile loop) from having watched one work

### Interview line
"Kubernetes owns the machines, Slurm owns the queue — and I've run both sides of that boundary."

---

## Phase 13 — Artifacts & interview narrative

**Goal:** Convert 50 hours of learning into things other people can see.

### Deliverables
- [ ] **GitHub repo:** Terraform + Ansible Slurm cluster on AWS, with elastic cloud nodes. README with an architecture diagram.
- [ ] **Blog post 1:** "Backfill scheduling explained, with `sdiag` numbers" — the tuning writeup from Phase 6.
- [ ] **Blog post 2:** "Cloud bursting: Slurm power_save vs Kubernetes autoscaling" — Phase 9's comparison.
- [ ] **Debugging notes** from Phase 4, cleaned up into a short troubleshooting guide.
- [ ] **Optional, high signal:** a bug report or patch to SchedMD's Bugzilla, or a documentation PR.

### Interview questions to be able to answer cold
1. Walk me through what happens from `sbatch` to job start.
2. How does backfill work, and what breaks it?
3. Explain fairshare. How does usage decay?
4. How does Slurm isolate GPUs between jobs?
5. Slurm or Kubernetes for a GPU training cluster? Defend it.
6. How would you scale a Slurm cluster into the cloud on demand?
7. A node keeps going DOWN. Walk me through your diagnosis.
8. What's the difference between DRAIN, DOWN, and FAIL states?
9. Why does Slurm need munge, and what happens when clocks drift?
10. How do you do multi-tenant chargeback on a shared GPU cluster?

---

## Appendix A — Minimal `slurm.conf` starting point

```
ClusterName=lab
SlurmctldHost=ctl
SlurmUser=slurm
AuthType=auth/munge

SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
SchedulerType=sched/backfill
SchedulerParameters=bf_window=1440,bf_resolution=60,bf_continue

ProctrackType=proctrack/cgroup
TaskPlugin=task/cgroup,task/affinity

PriorityType=priority/multifactor
PriorityDecayHalfLife=1-0
PriorityWeightFairshare=10000
PriorityWeightAge=1000
PriorityWeightPartition=1000
PriorityWeightQOS=10000

AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=ctl
JobAcctGatherType=jobacct_gather/cgroup

SlurmdSpoolDir=/var/spool/slurmd
StateSaveLocation=/var/spool/slurmctld
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid

NodeName=node[1-2] CPUs=2 RealMemory=800 State=UNKNOWN
PartitionName=debug Nodes=ALL Default=YES MaxTime=01:00:00 State=UP
```

Adjust `CPUs` and `RealMemory` to what `slurmd -C` reports on your instances, leaving headroom for the OS.

---

## Appendix B — Commands to know cold

**Users:** `sbatch` `srun` `salloc` `squeue` `scancel` `sacct` `sinfo` `sstat` `seff`
**Admins:** `scontrol` `sacctmgr` `sdiag` `sprio` `sshare` `sreport` `sview` `slurmd -C` `slurmd -D -vvv`
**The four you'll use most in interviews as evidence:** `sdiag`, `sprio -l`, `sshare -l`, `scontrol show node`

---

## Appendix C — RHEL-family cheat sheet (Rocky 9)

We use Rocky because HPC runs on RHEL-family. Rocky is a source rebuild of RHEL —
same packages, kernel, paths, behavior; no support contract. Everything you learn
here transfers to RHEL and Alma unchanged.

**Repos (do this first on every node):**
```bash
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled crb    # CodeReady Builder
```

**Package name translation:**

| Ubuntu | Rocky 9 |
|---|---|
| `build-essential` | `dnf groupinstall "Development Tools"` |
| `libmunge-dev` | `munge-devel` |
| `libmariadb-dev` | `mariadb-devel` |
| `libpam0g-dev` | `pam-devel` |
| `libjson-c-dev` | `json-c-devel` |
| `libyaml-dev` | `libyaml-devel` |
| `libjwt-dev` | `libjwt-devel` (EPEL) |
| `nfs-kernel-server` | `nfs-utils` |

**SELinux — learn it, don't disable it:**
```bash
getenforce                          # Enforcing by default
ausearch -m avc -ts recent          # what got denied and why
sealert -a /var/log/audit/audit.log # human-readable explanation
setsebool -P use_nfs_home_dirs on   # common Slurm-related fix
semanage fcontext -a -t <type> '<path>(/.*)?' && restorecon -Rv <path>
```
`setenforce 0` is the emergency lever, not the fix. Being able to read an AVC
denial is a real, rare skill.

**firewalld:**
```bash
firewall-cmd --list-all
firewall-cmd --permanent --add-port=6817/tcp   # slurmctld
firewall-cmd --permanent --add-port=6818/tcp   # slurmd
firewall-cmd --permanent --add-port=6819/tcp   # slurmdbd
firewall-cmd --reload
```

**Other differences:** `dnf` not `apt`; SSH user is `rocky`; `/etc/redhat-release`
exists; systemd unit paths match Ubuntu; cgroup v2 by default on 9.

---

## Session Log

| Date | Hours | Phase | What I did | What broke | What I learned |
|------|-------|-------|-----------|-----------|----------------|
| | | | | | |

---

## Currently

**Done:** _(nothing yet)_
**In progress:** Phase 0
**Next up:** Phase 1
**Blocked on:** —
