## 1. Purpose & Scope

**Purpose:** provide a clear, copy‑ready Notion document that you (or a teammate) can follow to complete the entire hands‑on project from creating EC2 instances through monitoring and alerting with Datadog.

**Scope (what this doc includes):**

- Launching 2 EC2 Ubuntu 22.04 instances
- Connecting to both instances using **MobaXterm** from Windows
- Running updates and upgrades interactively and via a patch script
- Changing hostnames with `hostnamectl` to `Testrun-Server1` and `Testrun-Server2`
- Stress testing the servers (CPU/disk) to validate monitoring
- Installing, configuring and verifying the **Datadog Agent** on both hosts
- Opening a Datadog account (basic steps) and linking servers (API key)
- Creating Datadog monitors (Host not reporting, CPU, Memory, Disk, Patch failure) and integrating alerts to **Slack** and **Email**

---

## 2. Prerequisites

Before you begin, make sure you have:

- An AWS account with permissions to create EC2 instances, security groups and key pairs.
- A Datadog account (or ability to create one) — you will need a Datadog API key.
- Windows PC with **MobaXterm** installed and internet access.
- Your public IPv4 address (so you can restrict SSH in the Security Group).

**Security notes:**

- Restrict inbound SSH to your IP only in the EC2 Security Group (do not use 0.0.0.0/0).
- Keep your `.pem` file protected and never share it.

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/fdeb2401edba0edfb0a4e2d41659e5b938910730/Screenshot_2025-08-29_135641.png)

---

## 3. Stage 1 — Launch two EC2 instances

**AMI & instance type (recommended):** Ubuntu Server 22.04 LTS, `t3.micro` for testing.

### Console steps (quick):

1. AWS Console → EC2 → Launch instances → `Launch`.
2. Choose **Ubuntu Server 22.04 LTS** AMI (select region-specific AMI in console).
3. Instance type: `t3.micro` (or `t3.small` if you want more resources).
4. Key pair: Create new key pair or use an existing one. Download the `.pem` file and store securely.
5. Network: pick your VPC and subnet.
6. Security group (create new): add inbound rules:
    - **SSH (TCP 22)** — Source: **YOUR_PUBLIC_IP/32**
    - (Optional) **ICMP (ping)** — Source: **YOUR_PUBLIC_IP/32**
7. Launch the instance and give it a Name tag: `Testrun-Server1`.
8. Repeat steps to launch a second instance and name it `Testrun-Server2`.

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/1ae934c88c15beeb1bc52508b0eadb58b370a783/Screenshot_2025-08-27_142345.png)

---

## 4. Stage 2 — Connect to instances with MobaXterm

**Option A — Use `.pem` directly (recommended):**

1. Open **MobaXterm** → `Session` → `SSH`.
2. Remote host: `ubuntu@<PUBLIC_IP>` (Ubuntu default user is `ubuntu`).
3. In **Advanced SSH settings** → tick **Use private key** → browse to your `my-key.pem` file.
4. Save session and connect. Repeat for the second host and save as `Testrun-Server2`.

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/513311f93b71cd7822ac1d37135aab828f586581/Screenshot%202025-07-30%20125215.png)

---

## 5. Stage 3 — Update / Upgrade (manual + script)

You will perform package updates and optionally automate via a script.

### Manual (interactive, Ubuntu):

```bash
# connect via MobaXterm terminal
sudo apt update            # refresh package lists
apt list --upgradable      # optional: list upgradable packages
sudo apt upgrade           # interactively approve upgrades
sudo apt full-upgrade -y   # upgrade including kernel packages non-interactively
sudo apt autoremove -y     # remove unused packages

```

**Important:** reboot if kernel upgrades were installed:

```bash
sudo reboot

```

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/e811b848cc45da8034614a52d0fc5b29fa0173b5/Screenshot%202025-07-30%20131338.png)

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/f8b1024cfbb53809f92a6693c6a8f3e6c5336762/Screenshot%202025-07-30%20130820.png)

---

## 6. Stage 4 — Change hostnames to `Testrun-Server1` / `Testrun-Server2`

On each server run (as `sudo`):

```bash
# On server 1
sudo hostnamectl set-hostname Testrun-Server1
# On server 2
sudo hostnamectl set-hostname Testrun-Server2

# Verify
hostnamectl status
hostname -f

```

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/9311e42621d6982456b63e92ec5b9fed55abad79/Screenshot_2025-08-27_153715.png)

---

## 7. Stage 5 — Stress test the servers

Install `stress` utility and run short tests to validate monitoring and alerting.

```bash
# Install stress (Ubuntu)
sudo apt update
sudo apt install -y stress

# Run CPU stress test for 2 minutes using 2 workers
stress --cpu 2 --timeout 120

# Observe CPU spike in Datadog (within a minute) and confirm monitor triggers or metrics show the spike.

```

**Disk stress (careful on small volumes):**

```bash
# Create a temporary large file (adjust size to fit available disk); remove after test
sudo fallocate -l 2G /tmp/hugefile
# monitor disk usage in Datadog
rm /tmp/hugefile

```

**Undo stress tool:** `sudo apt remove --purge -y stress` when done.

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/8174291144f333621b84ef864e74fdd475e0c418/Screenshot_2025-08-28_134311.png)

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/3cb317b8725dbdb3dcb73ca9f9c6eff90c082515/Screenshot_2025-08-28_133744.png)

---

## 8. Stage 6 — Install, configure & verify Datadog Agent

> Replace YOUR_DATADOG_API_KEY and DD_SITE if you use the EU site (datadoghq.eu).
> 

### One‑liner installer (Ubuntu / Debian):

```bash
DD_AGENT_MAJOR_VERSION=7 DD_API_KEY=YOUR_DATADOG_API_KEY DD_SITE="datadoghq.com" \
  bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"

```

### Quick manual steps (if you prefer package manager):

```bash
# add apt repo (example)
sudo apt-get update
sudo apt-get install -y gnupg ca-certificates
# Add Datadog repository & key as per official docs, then:
sudo apt-get install -y datadog-agent
# Add API key to /etc/datadog-agent/datadog.yaml
sudo sed -i "s/^api_key:.*/api_key: YOUR_DATADOG_API_KEY/" /etc/datadog-agent/datadog.yaml
sudo systemctl restart datadog-agent

```

Then restart:

```bash
sudo systemctl restart datadog-agent

```

### Verify agent is running and data flow is healthy:

```bash
sudo datadog-agent status
# check for host, checks, and no long error traces
sudo tail -n 200 /var/log/datadog/agent.log

```

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/d1041453462eef896e1d78fc368e6cb47a9507ec/Screenshot_2025-08-27_175915.png)

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/b23d2ee76a5202ec32b8f7b102060d54c52421b2/Screenshot%20_2025-08-27_181800.png)

---

## 9. Stage 7 — Create Datadog account & link servers

If you do not yet have a Datadog account:

1. Go to `https://www.datadoghq.com/` (or `datadoghq.eu`) and click *Get Started / Sign Up*.
2. Register, verify email and login.
3. In Datadog UI: go to **Integrations → APIs** to view/generate your **API key** and **Application key** (API key is required on hosts).

**Linking servers:**

- Use the API key in the agent install (see Stage 6). When the agent starts, the host will register automatically with Datadog.

**Verify:** Datadog → Infrastructure → Hosts → find `Testrun-Server1` and `Testrun-Server2`.

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/43eb0e3e47fdb3e3b27fd518621dec5f2fb0e432/Screenshot_2025-08-27_175030.png)

![image alt](https://github.com/GodwinChineduNedu/Ec2_Datadog_Monitoring_Project/blob/8f54d025622981e1dbf8a695c0dea36a4a5f39e6/Screenshot_2025-08-27_175159.png)

---

## 10. Stage 8 — Create monitors and integrate Slack & Email

This section shows recommended monitors and how to connect Slack and Email alerts to them.

### Recommended monitors (create in Datadog → Monitors → New Monitor):

1. **Host not reporting**
    - Type: Host (Datadog built-in)
    - Condition: Host not reporting for 5 minutes
    - Notify: add email addresses and Slack channel mention
2. **CPU high (per host)**
    - Type: Metric
    - Query (example): `avg(last_5m):avg:system.cpu.user{env:testrun} by {host} > 80`
    - Notify: `team@example.com, @slack-#alerts` and include runbook
3. **Memory low (usable %)**
    - Query: `avg(last_5m):avg:system.mem.pct_usable{env:testrun} by {host} < 0.15`
4. **Disk usage (>85%)**
    - Query: `max(last_5m):avg:system.disk.in_use{env:testrun} by {host,device} > 0.85`
5. **Patch script failure (event monitor)**
    - Type: Event
    - Search query: `title:"Patch FAILED on"` or `sources:patch_script` if you tag events
    - Trigger: at least 1 matching event in the last 5 minutes

### Notification field example (use in each monitor):

```
{{host.name}} triggered {{trigger.name}} at {{timestamp}}
Runbook: SSH via MobaXterm and run: sudo datadog-agent status ; tail -n 200 /var/log/datadog/agent.log
Notify: team@example.com, ops@example.com, @slack-#alerts

```

### Integrate Slack (Datadog → Slack):

1. Datadog UI → Integrations → Slack → Install integration.
2. You will be redirected to Slack to authorize the Datadog app — choose the channel to post alerts (e.g., `#alerts`).
3. After install, Datadog shows the Slack integration mention string (e.g. `@slack-#alerts`). Use that in monitor Notify field.

**Integrate Email:**

- In monitor's notify box, just add `team@example.com, oncall@example.com` — Datadog sends emails to listed addresses.

### Test monitors & alerts:

- Stop the datadog-agent on `Testrun-Server1` to test Host not reporting:

```bash
sudo systemctl stop datadog-agent
# wait > 5 minutes for monitor to trigger
sudo systemctl start datadog-agent

```

- Generate CPU load (see Stage 5) to test CPU monitor.

- Force the patch script to fail (`exit 1`) and run it to ensure the Patch FAILED event triggers the event monitor.












