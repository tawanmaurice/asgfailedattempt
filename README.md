# VPC + ALB + Auto Scaling (ASG) — Attempt Log (Paused)

> **Date:** November 5, 2025
> **Status:** Paused / Will retry
> **Owner:** Tawan Perry

---

## What I Tried to Do

I set out to **build and test a fully automated web architecture using AWS core services**, including a VPC, public/private subnets, an Application Load Balancer (ALB), and an Auto Scaling Group (ASG) based on a Launch Template.
The goal was to **automatically deploy web servers that scale based on load**, run `httpd` via user data, and serve traffic through the ALB—demonstrating a complete end-to-end setup suitable for a cloud portfolio project.

Specifically, I tried to:

* Build a **custom VPC** with **public subnets** (NAT optional).
* Deploy an **Application Load Balancer (ALB)** tied to a **Target Group** on HTTP port 80.
* Create a **Launch Template (LT)** to automatically install and start Apache (`httpd`).
* Configure an **Auto Scaling Group (ASG)** using the LT to maintain two EC2 instances.
* Use **SSM Session Manager** for access instead of SSH keys.
* Verify the ALB health checks and scaling functionality.

---

## Current Outcome

* **ALB Target Group health:** `Unhealthy (Request timed out)`
* **Attempts:** 4 refresh cycles after updating user data
* **SSM Access:** Working (proves IAM role and outbound SSM access fine)
* **Root Issue:** Instances not serving HTTP traffic on port 80 despite user data

---

## Likely Root Causes

1. **ASG using an old Launch Template version**

   * The new user data changes weren’t applied.

2. **Lack of outbound internet (no NAT Gateway)**

   * EC2 instances couldn’t download and install `httpd`.

3. **Security Group misconfiguration**

   * ALB unable to reach EC2 on port 80.

4. **Health check too strict or incorrect**

   * `/` endpoint failing; TCP health check might work.

5. **Restrictive Network ACLs**

   * Missing ephemeral port rules for return traffic.

---

## User Data Used

```bash
#!/bin/bash
set -euxo pipefail
exec > /var/log/user-data.log 2>&1

(dnf -y update || yum -y update)
(dnf -y install httpd || yum -y install httpd)

systemctl enable httpd
systemctl start httpd

mkdir -p /var/www/html
echo "OK" > /var/www/html/health
echo "Hello from ASG $(hostname -f)" > /var/www/html/index.html
```

---

## Diagnostics Plan (Next Attempt)

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
sudo tail -n 200 /var/log/user-data.log
sudo systemctl status httpd
sudo ss -ltnp | grep :80
curl -I http://localhost/
```

**Expected Results:**

* `httpd` installed and running
* `curl -I` returns `HTTP/1.1 200 OK`

---

## Decision

Marking this build as **incomplete** but keeping it as a **learning log for my GitHub portfolio**.
The next session will focus on building cleanly, confirming the Launch Template version, and ensuring proper network routing and security.

---

## Clean-up Steps

1. Delete **ASG** (`11525asg`)
2. Delete **Target Group** (`11525TargetGroups`)
3. Delete **ALB** (`11525webloadbalancer`)
4. Delete **Launch Template** and all versions
5. Delete **Security Groups** (after dependencies removed)
6. Release **NAT Gateway** and **Elastic IP** if used
7. Delete **VPC** if created solely for this lab

---

## Retry Checklist

* [ ] Ensure **ASG → Launch Template Version** = Latest
* [ ] Use **Public Subnets** for initial testing
* [ ] Verify **SG rules** between ALB and EC2
* [ ] Start with **TCP health checks**, then switch to HTTP `/health`
* [ ] Confirm `httpd` is running via SSM before trusting ALB
* [ ] Take screenshots of healthy targets and ALB response

---

## Lessons Learned

* Always double-check the Launch Template version before refreshing.
* Without NAT, EC2 can’t install packages—no web service = no health.
* SSM access is the best way to diagnose in real-time.
* TCP health checks help isolate networking issues before HTTP-level checks.

---

## Next Session Goals

* Rebuild cleanly with verified connectivity.
* Confirm healthy targets (`2/2 Healthy`) and a valid ALB DNS `200 OK` response.
* Capture final screenshots for GitHub documentation before teardown.

**End of Attempt — Logged 11/5/25**
