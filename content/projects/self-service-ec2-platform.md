---
title: "Building a Self-Service Analysis Environment for Data Scientists"
date: 2024-03-15
tags: ["python", "aws", "cli", "platform-engineering"]
summary: "Designing and building a Python CLI that lets data scientists create, manage, and safely shut down cloud research environments without needing to know Terraform or the AWS console."
---

Data scientists need powerful compute. This post covers how I built a CLI tool that gives scientists self-service access to EC2 instances while quietly solving the operational problems they don't think about: data persistence, cost control, and environment reproducibility.

## The Problem

Our data science team needed on-demand cloud compute for analysis, sometimes genomics workloads that could run for hours on large-memory instances. The initial approach was manual: someone with AWS access would spin up an EC2 instance, configure it, and hand over SSH credentials.

This had the usual problems:

- **Environment drift**: each instance was configured slightly differently depending on who set it up and when
- **Data loss risk**: work stored on EBS volumes would be lost if an instance was terminated without backup
- **Cost leaks**: instances left running overnight or over weekends, sometimes for days
- **Bottleneck**: scientists couldn't spin up their own environments without asking an engineer

The goal was to productise this workflow into a tool that scientists could use independently.

## The Design

I built a Python CLI using [Typer](https://typer.tiangolo.com/) and [Rich](https://rich.readthedocs.io/) that wraps the full lifecycle of a research EC2 instance into five commands:

```
create   → provision a new instance with persistent storage
start    → resume a stopped instance, restore data, configure tools
stop     → sync data to S3, then stop the instance
destroy  → terminate with confirmation guards
list     → show all instances with state and ownership
```

### Architecture

Each instance has three layers of state:

1. **Ephemeral compute** — the EC2 instance itself, which can be stopped and started
2. **Persistent volume** — an EBS volume mounted at `/data` that survives stop/start cycles
3. **Durable backup** — automatic S3 sync so data survives even instance termination

## Data Persistence

The trickiest engineering problem wasn't provisioning, it was making sure scientists never lost work.

### The S3 Sync Model

Every instance is tagged with an S3 bucket and prefix at creation time. The stop command syncs the `/data` volume to S3 before shutting down, and the start command restores it. This means scientists can:

- Stop an instance at the end of the day (saving money)
- Start it the next morning and pick up where they left off
- Even destroy and recreate an instance and get their data back

The sync uses [s5cmd](https://github.com/peak/s5cmd) for high-throughput parallel transfers: significantly faster than the AWS CLI's `s3 sync` for the large genomics datasets we work with.

### Guard Rails

Data sync has several safety checks:

- **Volume size validation at creation**: the CLI checks the size of existing data in S3 and ensures the requested EBS volume is large enough, with headroom
- **Cloud-init completion check before stop**: ensures the instance has finished its bootstrap before attempting any sync
- **Empty sync protection**: if the local volume is empty (which would be a bug, not a real state), the sync is blocked to prevent overwriting good S3 data with nothing
- **SSM readiness gate**: sync runs via SSM commands on the instance; the CLI waits for the SSM agent to be healthy before attempting any remote operation

If sync fails, the CLI warns but lets the user decide whether to proceed with the stop; the data is still on the EBS volume either way.

## Instance Bootstrap

When a new instance is created, a `user_data` shell script runs on first boot. This script provisions the entire research environment automatically:

- **System packages**: Python, Docker, AWS CLI, GitHub CLI, s5cmd, CloudWatch agent
- **Storage setup**: partitions and mounts the EBS data volume, configures Docker to use a separate volume for image storage
- **Data restoration**: syncs the user's project data from S3 to `/data`
- **Monitoring**: configures the CloudWatch agent for operational logs
- **Shell defaults**: sets up the user's environment for internal package workflows

The start command (for resuming stopped instances) does a lighter version of this, remounting volumes, restarting Docker, and reapplying Git configuration, rather than re-running the full bootstrap.

### SSH Configuration

The CLI auto-generates SSH config entries using SSM Session Manager as a proxy:

```
Host my-analysis-node
    HostName i-0abc123...
    User ubuntu
    ProxyCommand sh -c "aws ssm start-session --target %h ..."
    ForwardAgent yes
```

This means scientists can connect via `ssh my-analysis-node` or use VS Code's Remote-SSH extension with a friendly hostname. SSH agent forwarding is enabled by default, so their local GitHub keys work on the instance without copying credentials.

The CLI offers to append this entry automatically, and the destroy command offers to remove it, keeping `~/.ssh/config` clean.

### Key Pair Management

Instance creation walks users through key pair selection interactively, listing existing AWS key pairs, offering to create new ones, saving the private key with correct permissions, and prompting for the local path. This eliminates the "I lost my key" support request.

### Git Identity Propagation

The CLI has a `configure` command for persisting settings like GitHub username and email. On create and start, these are pushed to the instance via SSM so `git commit` works immediately without manual setup. The fallback chain is: explicit CLI flags → persisted config → local `git config` on the user's machine.

### Instance Resize

Scientists often discover mid-project that they need more memory. The `change-type` command lets them resize a stopped instance without reprovisioning: just stop, change type, start.

## Cost Control

The biggest operational win wasn't the CLI itself - it was the cost automation built around it.

### Cron-Based Auto-Shutdown

Sometimes the data scientists would forget to stop their instances, leading to idle instances costing us significantly. However, the data scientists had variable schedules and long-running programmes. So I documented a pattern using local cron jobs that they could use:

- **4pm** — discover all running instances owned by the user, broadcast a warning
- **5pm** — stop all running instances that haven't been snoozed

The snooze mechanism is deliberately simple: `touch /tmp/snooze_<instance-name>`. If the file exists, the 5pm job skips that instance. The 4pm job clears all snooze files daily, so the default is always "shut down unless you actively say otherwise."
