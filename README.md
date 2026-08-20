# Managing Services — Monitoring (AWS Lab Walkthrough)

Documented run-through of an AWS Training lab on managing the `httpd` service with `systemctl` and monitoring an Amazon Linux 2 EC2 instance with `top` and CloudWatch. Screenshots below are taken directly from my own lab session (`us-west-2`, EC2 instance `ip-10-0-10-114`), in the order I actually ran them.

## Environment

- Amazon Linux 2 (AL2 EOL 2026-06-30), `t3.micro`
- Region: `us-west-2` (Oregon)
- Instance: `ip-10-0-10-114`, user `ec2-user`, key auth via `imported-openssh-key`

---

## 1. Console Home

Logged into the AWS Management Console for the lab account.

![Console Home](images/01-console-home.png)

## 2. Checked `httpd` status — inactive

SSH'd into the instance and ran:

```
sudo systemctl status httpd.service
```

Output confirmed the service was **loaded but inactive**:

```
Loaded: loaded (/usr/lib/systemd/system/httpd.service; disabled; vendor preset: disabled)
Active: inactive (dead)
```

![httpd inactive](images/02-httpd-status-inactive.png)

## 3. Started `httpd` — active and running

Ran:

```
sudo systemctl start httpd.service
sudo systemctl status httpd.service
```

Status changed to:

```
Active: active (running) since Thu 2026-08-20 19:22:02 UTC; 12s ago
Main PID: 2615 (httpd)
```

with worker processes 2615–2620 spawned under `/system.slice/httpd.service`. Journal lines confirmed `systemd[1]: Started The Apache HTTP Server.`

![httpd active](images/03-httpd-status-active.png)

## 4. Baseline `top` — before load

Ran `top` with the instance idle:

```
top - 19:24:03 up 9 min, 1 user, load average: 0.00, 0.04, 0.04
Tasks: 89 total, 1 running, 48 sleeping
%Cpu(s): 0.2 us, 0.0 sy, 98.8 id
```

Essentially no CPU load — top process was `systemd` itself at 0.0% CPU.

![top baseline](images/04-top-baseline.png)

## 5. `top` under load from `stress.sh`

Ran:

```
./stress.sh & top
```

Within the same minute, load jumped noticeably:

```
top - 19:24:55 up 10 min, 1 user, load average: 1.12, 0.26, 0.11
Tasks: 105 total, 15 running, 50 sleeping
%Cpu(s): 61.2 us, 38.8 sy, 0.0 id
```

12 `stress` processes (PIDs 2662–2675, user `ec2-user`) were running, each pulling ~14.0–14.3% CPU.

![top under stress](images/05-top-stress-load.png)

## 6. CloudWatch — Custom Dashboards (empty)

Opened CloudWatch → Dashboards. No custom dashboards existed for this account (`Custom Dashboards (0)`).

![CloudWatch custom dashboards empty](images/06-cloudwatch-custom-dashboards-empty.png)

## 7. CloudWatch — Automatic Dashboards

Switched to the **Automatic dashboards** tab — AWS had already generated 7 of these for the account, including **EBS**, **EC2**, **EventBridge**, **Prometheus**, and **CloudWatch Usage**.

![CloudWatch automatic dashboards](images/07-cloudwatch-automatic-dashboards.png)

## 8. Opened the EC2 automatic dashboard — no data yet

Clicked into the **EC2** dashboard (3h view). At this point CPU Utilization, DiskReadBytes, and DiskReadOps all showed **"No data available."**

![EC2 dashboard no data](images/08-ec2-dashboard-no-data.png)

## 9. CPU spike appears

A short refresh later, the same dashboard showed the CPU spike from the `stress.sh` run — a sharp jump to about **4.4%** CPU Utilization right around 19:00–19:xx on the graph. Disk metrics remained empty (the stress test never touched disk I/O, so that's expected — not a gap in monitoring).

![EC2 dashboard CPU spike](images/09-ec2-dashboard-cpu-spike.png)

## 10. Alarm permissions error

Came back to the same EC2 dashboard ~14 minutes later (8:42 PM vs. 8:28 PM in step 9). The page attempted to load alarm data and threw:

```
Failed to retrieve alarms
You don't have permissions to perform the following operations: CloudWatch:DescribeAlarms.
```

This is expected in the lab sandbox — the lab environment restricts IAM permissions to only what the exercise requires, and viewing/describing alarms isn't part of that scope. The CPU graph itself was unaffected and still showed the earlier spike.

![Alarm permission error](images/10-ec2-dashboard-alarm-permission-error.png)

---

## What this confirmed

- `systemctl status/start` correctly reported and changed `httpd`'s state (`inactive` → `active (running)`), with the worker processes and journal entries to back it up.
- `top` showed the exact CPU shift caused by `stress.sh`: idle at 98.8% id → 61.2% us / 38.8% sy with 12 concurrent `stress` processes.
- CloudWatch's **automatic EC2 dashboard** picked up that same CPU spike without any manual dashboard setup, on a short delay (no data at first look, populated on the next refresh).
- Disk metrics stayed empty throughout — consistent with a CPU-only stress test.
- The `CloudWatch:DescribeAlarms` permission error is a lab-sandbox restriction, not a real issue with the setup.

## Repo structure

```
.
├── README.md
└── images/
    ├── 01-console-home.png
    ├── 02-httpd-status-inactive.png
    ├── 03-httpd-status-active.png
    ├── 04-top-baseline.png
    ├── 05-top-stress-load.png
    ├── 06-cloudwatch-custom-dashboards-empty.png
    ├── 07-cloudwatch-automatic-dashboards.png
    ├── 08-ec2-dashboard-no-data.png
    ├── 09-ec2-dashboard-cpu-spike.png
    └── 10-ec2-dashboard-alarm-permission-error.png
```

## Source

Based on an AWS Training and Certification lab exercise ("Managing Services - Monitoring"). © Amazon Web Services, Inc. and its affiliates — original lab content used here for personal learning/documentation purposes only.
