# Slurm Lab

## What this is
Learning project to reach expert-level Slurm plus AWS infrastructure
fluency. Following docs/PLAN.md — 14 phases. Read that file for current
progress; the tracker table and the "Currently" block at the bottom are
the source of truth.

## About me
Senior software engineer. .NET/C#/Angular/Kubernetes background. I work
on an enterprise HPC scheduling and VDI platform, so I already integrate
with Slurm — the goal here is operational depth, not introductions.
Comfortable with Linux, cloud, and distributed systems. Skip beginner
explanations.

## How I want you to work with me
- Teach, don't just solve. When something breaks, walk me through the
  diagnosis before giving the fix. The debugging IS the curriculum.
- Never skip ahead in the plan. Ask before moving to a new phase.
- Explain WHY a config value matters, not just what to set it to.
- When I paste an error, ask what I've already checked before answering.
- Don't run AWS console steps for me in early phases — I want to do
  those by hand.

## Environment
- Laptop: Windows + WSL2 (Ubuntu 26.04). All work happens in WSL.
- AWS free account plan, region ap-south-1, credit-limited.
  Instances get stopped after every session.
- Cluster: ctl (slurmctld + slurmdbd + MariaDB), node1, node2
- EC2 nodes run Rocky Linux 9 (RHEL-family). Slurm built from source to /opt/slurm,
  shared over NFS.

## Conventions
- terraform/ = infrastructure, ansible/ = configuration
- configs/ = slurm.conf, cgroup.conf, gres.conf variants
- scripts/ = helper scripts (resume/suspend for Phase 9)
- docs/ = plan and writeups
- Never commit secrets, especially munge keys or .pem files
- Update the tracker in docs/PLAN.md at the end of each session
