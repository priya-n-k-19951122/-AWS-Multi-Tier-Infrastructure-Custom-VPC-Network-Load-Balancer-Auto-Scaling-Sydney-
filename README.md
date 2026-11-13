# AWS Multi-Tier Infrastructure: Custom VPC, Network Load Balancer & Auto Scaling (Sydney)

This repository documents the end-to-end deployment of a fault-tolerant, multi-tier web architecture in the **AWS Asia Pacific (Sydney) — `ap-southeast-2`** region. It features a custom Virtual Private Cloud (VPC), an Internet-facing **Network Load Balancer (NLB)**, two Target Groups, and two **Auto Scaling Groups** dynamically managing **3 Mobile EC2 servers** and **4 Electronics EC2 servers**.

---

## Architecture Overview & Resource Inventory

| Component | Configuration & Resource Identifiers |
| :--- | :--- |
| **AWS Region** | Asia Pacific (Sydney) — `ap-southeast-2` |
| **Custom VPC** | `Sydney-Mock-VPC` (`vpc-039a0b16597470b92`) — CIDR: `10.0.0.0/16` |
| **Public Subnets** | `subnet-0dbc7e396502edad9` (`ap-southeast-2a`) & `subnet-03d1ddd2a7f273a75` (`ap-southeast-2b`) |
| **Internet Gateway & Route Table** | `igw-0040dd6abc14329da` attached to `Sydney-Mock-VPC-rtb-public` (`0.0.0.0/0` → `igw`) |
| **Security Group** | `Mock-NLB-EC2-SG` (`sg-05afe67e3ee6230e0`) — Inbound Ports: `22`, `80`, `81` |
| **Network Load Balancer** | `Sydney-Mock-NLB` (Internet-facing, IPv4, Dual-AZ) |
| **Target Groups** | `TG-Mobile-MO` (`TCP:80`) & `TG-Electronics-EL` (`TCP:80`) |
| **Launch Templates** | `MO-Template` (Mobile Web App) & `EL-Template` (Electronics Web App) |
| **Auto Scaling Groups** | `ASG-Mobile-MO` (`3` instances) & `ASG-Electronics-EL` (`4` instances) |
| **Compute Fleet** | `7x` Amazon Linux EC2 Instances running Apache HTTP Server (`httpd`) |

---

## Step-by-Step Deployment Guide

### Step 1: Create Custom VPC in Sydney (`ap-southeast-2`)
1. Switch the AWS Management Console region to **Asia Pacific (Sydney) `ap-southeast-2`**.
2. Open **VPC Console** → **Your VPCs** → **Create VPC** and select **VPC and more**.
3. Configure the network settings:
   * **Name tag auto-generation:** `Sydney-Mock-VPC`
   * **IPv4 CIDR block:** `10.0.0.0/16`
   * **Availability Zones:** `2` (`ap-southeast-2a` and `ap-southeast-2b`)
   * **Public subnets:** `2`
   * **Private subnets:** `0`
4. Click **Create VPC** and confirm that `Sydney-Mock-VPC-rtb-public` has a `0.0.0.0/0` route pointing to the Internet Gateway (`igw-0040dd6abc14329da`).
5. Navigate to **Subnets**, select both public subnets individually, click **Actions** → **Edit subnet settings**, and enable **Auto-assign public IPv4 address**.

---

### Step 2: Configure Security Group (`Mock-NLB-EC2-SG`)
1. Navigate to **EC2 Console** → **Security Groups** → **Create security group**.
2. Name it `Mock-NLB-EC2-SG` and attach it to `Sydney-Mock-VPC` (`vpc-039a0b16597470b92`).
3. Configure the following **Inbound Rules**:

| Type / Protocol | Port | Source | Description |
| :--- | :--- | :--- | :--- |
| **SSH** | `22` | `0.0.0.0/0` | Remote server administration access |
| **HTTP** | `80` | `0.0.0.0/0` | Mobile Store NLB listener & EC2 web traffic |
| **Custom TCP** | `81` | `0.0.0.0/0` | Electronics Store NLB listener traffic |

4. Confirm **Outbound Rules** allow `All Traffic` to `0.0.0.0/0` and click **Create security group**.

---

### Step 3: Create Launch Templates (`MO-Template` & `EL-Template`)
Create two separate EC2 Launch Templates using **Amazon Linux 2023**, `t2.micro` / `t3.micro`, and `Mock-NLB-EC2-SG` (`sg-05afe67e3ee6230e0`).

#### A. Mobile Launch Template (`MO-Template`) — User Data
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Welcome to MOBILE Store (MO) - Instance: $(hostname -f)</h1>" > /var/www/html/index.html
```

#### B. Electronics Launch Template (`EL-Template`) — User Data
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Welcome to ELECTRONICS Store (EL) - Instance: $(hostname -f)</h1>" > /var/www/html/index.html
```

---

### Step 4: Create Target Groups & Network Load Balancer
1. **Create Mobile Target Group:**
   * **Target type:** `Instances` | **Name:** `TG-Mobile-MO` | **Protocol/Port:** `TCP:80` | **VPC:** `Sydney-Mock-VPC`
2. **Create Electronics Target Group:**
   * **Target type:** `Instances` | **Name:** `TG-Electronics-EL` | **Protocol/Port:** `TCP:80` | **VPC:** `Sydney-Mock-VPC`
3. **Create Network Load Balancer (`Sydney-Mock-NLB`):**
   * **Scheme:** `Internet-facing` | **IP address type:** `IPv4`
   * **Network mapping:** Select `Sydney-Mock-VPC` and map both public subnets (`ap-southeast-2a` and `ap-southeast-2b`).
   * **Security groups:** Replace the `default` VPC security group with `Mock-NLB-EC2-SG` (`sg-05afe67e3ee6230e0`).
   * **Listener 1:** `TCP:80` → Forward to `TG-Mobile-MO`
   * **Listener 2:** `TCP:81` → Forward to `TG-Electronics-EL` *(NLB translates incoming Port `81` traffic to Port `80` on the target EC2 instances)*

---

### Step 5: Provision Auto Scaling Groups (`3` Mobile + `4` Electronics)

1. **Mobile Auto Scaling Group (`ASG-Mobile-MO`):**
   * **Launch template:** `MO-Template`
   * **Network:** `Sydney-Mock-VPC` (both public subnets)
   * **Load balancing:** Attach to existing target group `TG-Mobile-MO` + enable **ELB health checks**
   * **Capacity:** Desired = `3`, Minimum = `3`, Maximum = `3`
2. **Electronics Auto Scaling Group (`ASG-Electronics-EL`):**
   * **Launch template:** `EL-Template`
   * **Network:** `Sydney-Mock-VPC` (both public subnets)
   * **Load balancing:** Attach to existing target group `TG-Electronics-EL` + enable **ELB health checks**
   * **Capacity:** Desired = `4`, Minimum = `4`, Maximum = `4`

---

## Verification & Load Balancing Tests

1. **Verify EC2 Fleet:** Confirm **7 EC2 instances** (`3` Mobile + `4` Electronics) are in the `Running` state across both Availability Zones.
2. **Verify Target Health:** Check the **Targets** tab inside both `TG-Mobile-MO` and `TG-Electronics-EL` to confirm all instances show `Healthy`.
3. **Test Endpoints in Browser (HTTP):**
   * **Mobile Store (`3` instances):**
     ```text
     http://<NLB-DNS-NAME>/
     ```
     *Expected output:* `Welcome to MOBILE Store (MO) - Instance: ip-10-0-x-x.ap-southeast-2.compute.internal`
   * **Electronics Store (`4` instances):**
     ```text
     http://<NLB-DNS-NAME>:81/
     ```
     *Expected output:* `Welcome to ELECTRONICS Store (EL) - Instance: ip-10-0-x-x.ap-southeast-2.compute.internal`

---

## Key Troubleshooting Lessons & Cleanup

* **NLB Security Group:** Always swap the default VPC security group on the NLB for `Mock-NLB-EC2-SG` (`sg-05afe67e3ee6230e0`) so external traffic on Ports `80` and `81` is not dropped.
* **Port Syntax (`:81` vs `/81`):** Because an NLB routes Layer 4 TCP traffic by port, append `:81` with a colon directly after `.com` rather than a forward slash (`/81`).
* **Post-Lab Resource Cleanup:** Delete both Auto Scaling Groups (`ASG-Mobile-MO`, `ASG-Electronics-EL`) and the Network Load Balancer (`Sydney-Mock-NLB`) after testing to avoid ongoing hourly charges.
