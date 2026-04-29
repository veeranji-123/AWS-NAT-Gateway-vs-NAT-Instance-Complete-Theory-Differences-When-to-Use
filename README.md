
# AWS NAT Gateway vs NAT Instance Complete Theory, Differences & When to Use

> **A comprehensive guide for AWS architects, cloud engineers, and DevOps professionals**

---

## Table of Contents

1. [What is NAT? The Core Concept](#1-what-is-nat--the-core-concept)
2. [Why Do We Need NAT in AWS?](#2-why-do-we-need-nat-in-aws)
3. [What is a NAT Gateway?](#3-what-is-a-nat-gateway)
4. [What is a NAT Instance?](#4-what-is-a-nat-instance)
5. [Deep Dive How NAT Gateway Works](#5-deep-dive--how-nat-gateway-works)
6. [Deep Dive How NAT Instance Works](#6-deep-dive--how-nat-instance-works)
7. [Full Comparison Table](#7-full-comparison-table)
8. [When to Use NAT Gateway](#8-when-to-use-nat-gateway)
9. [When to Use NAT Instance](#9-when-to-use-nat-instance)
10. [Architecture Diagrams Text Based](#10-architecture-diagrams--text-based)
11. [Step-by-Step Setup Overview](#11-step-by-step-setup-overview)
12. [Cost Comparison](#12-cost-comparison)
13. [Security Considerations](#13-security-considerations)
14. [Common Real-World Scenarios](#14-common-real-world-scenarios)
15. [Summary & Decision Guide](#15-summary--decision-guide)

---

## 1. What is NAT? The Core Concept

**NAT** stands for **Network Address Translation**.

It is a networking technique that allows multiple devices (or EC2 instances) sitting on a **private network** to communicate with the **public internet** but **without exposing their private IP addresses**.

### How NAT Works (Simply)

```
Private Instance (10.0.1.5) NAT Device (Public IP: 52.x.x.x) Internet
 
```

- The private instance sends traffic to the NAT device.
- The NAT device **replaces the private source IP** with its own **public IP**.
- The internet sees only the NAT's public IP **never the private IP**.
- Response comes back to NAT, which then **forwards it to the correct private instance**.

This is called **Source NAT (SNAT)** the source IP is translated.

> **Key Rule:** NAT allows **outbound** internet access from private subnets. 
> It does **NOT** allow unsolicited **inbound** connections from the internet to private instances.

---

## 2. Why Do We Need NAT in AWS?

In AWS, we follow the **VPC (Virtual Private Cloud)** architecture model:

### Standard AWS VPC Architecture

```
VPC (10.0.0.0/16)

 Public Subnet (10.0.1.0/24)
 Has a route to Internet Gateway (IGW)
 Resources here have public IPs

 Private Subnet (10.0.2.0/24)
 NO direct route to Internet Gateway
 Resources here have ONLY private IPs
```

### Why Private Subnets Exist

Private subnets are used for resources that should **NOT be directly accessible** from the internet:

- **Databases** (RDS, Aurora) you don't want your database exposed to the internet.
- **Application servers** backend logic that only a load balancer should reach.
- **Microservices** internal services communicating within the VPC.
- **Lambda functions** in a VPC.
- **ECS/EKS worker nodes**.

### The Problem

Resources in **private subnets** sometimes need to reach the internet for:

- Downloading software packages (`apt-get`, `yum`, `pip`, `npm`)
- Calling external APIs (payment gateways, SMS services, etc.)
- Sending data to external monitoring tools (Datadog, Splunk)
- Reaching AWS services in other regions or public endpoints
- Fetching OS patches and security updates

**But** they cannot route through the Internet Gateway because:
1. They have no public IP.
2. Their route tables don't point to the IGW.

### The Solution NAT

A NAT device is placed in the **public subnet**. Private instances route their outbound internet traffic through this NAT device. The NAT device has a public IP and can communicate with the internet on behalf of the private instances.

---

## 3. What is a NAT Gateway?

A **NAT Gateway** is a **fully managed, highly available AWS service** that provides Network Address Translation for instances in private subnets.

You don't manage any EC2 instance. AWS handles everything the hardware, the OS, the software, the scaling, the redundancy. You simply deploy it and point your route table to it.

### Key Characteristics of NAT Gateway

| Property | Details |
|---|---|
| Type | Managed AWS Service (not an EC2 instance) |
| Availability | Highly available within a single AZ |
| Bandwidth | Scales automatically up to 100 Gbps |
| Managed by | AWS (no user intervention needed) |
| OS access | None it's a black box |
| Public IP | Uses an Elastic IP (EIP) |
| Security Groups | NOT supported only NACLs apply |
| Source/Dest Check | Not applicable |

### Two Types of NAT Gateway

**1. Public NAT Gateway**
- Placed in a **public subnet**
- Assigned an **Elastic IP (EIP)**
- Routes traffic to the **Internet Gateway (IGW)**
- Used when private instances need to reach the **public internet**

**2. Private NAT Gateway**
- Placed in a **private subnet**
- No Elastic IP
- Routes traffic to another **VPC** (via Transit Gateway or VPC Peering)
- Used for **private-to-private** cross-VPC communication

> For most common use cases (private instances accessing the internet), you will use a **Public NAT Gateway**.

---

## 4. What is a NAT Instance?

A **NAT Instance** is a regular **EC2 instance** that you manually configure to perform NAT functionality. It runs a special AMI (Amazon Machine Image) provided by AWS that includes NAT software pre-installed.

You manage it like any other EC2 instance you pick the instance type, configure the OS, manage security groups, apply patches, and handle failures yourself.

### Key Characteristics of NAT Instance

| Property | Details |
|---|---|
| Type | EC2 Instance (user-managed) |
| Availability | Single instance NOT HA by default |
| Bandwidth | Limited by EC2 instance type |
| Managed by | You (the user/team) |
| OS access | Full (SSH into it, configure, modify) |
| Public IP | Elastic IP assigned manually |
| Security Groups | YES you configure them |
| Source/Dest Check | Must be **DISABLED** manually |

### Important: Disable Source/Destination Check

By default, every EC2 instance in AWS checks that it is either the **source** or **destination** of all traffic it handles. If it's neither, the traffic is dropped.

Since a NAT Instance **forwards traffic on behalf of others**, it is neither the source nor destination so you **must disable this check**.

```
EC2 Instance Actions Networking Change Source/Destination Check DISABLE
```

---

## 5. Deep Dive How NAT Gateway Works

### Step-by-Step Traffic Flow

```
Step 1: Private EC2 (10.0.2.100) wants to reach google.com (142.250.x.x)

Step 2: Route Table for Private Subnet:
 Destination Target
 0.0.0.0/0 nat-xxxxxxxx (NAT Gateway ID)

Step 3: Traffic arrives at NAT Gateway in Public Subnet

Step 4: NAT Gateway translates:
 Source IP: 10.0.2.100 52.10.5.100 (Elastic IP)
 Source Port: 34521 51022 (mapped port for tracking)

Step 5: NAT Gateway sends packet to Internet Gateway (IGW)

Step 6: IGW routes traffic to the internet google.com receives request

Step 7: google.com responds to 52.10.5.100:51022

Step 8: IGW receives response, sends to NAT Gateway

Step 9: NAT Gateway uses its connection tracking table to find:
 Destination: 52.10.5.100:51022 10.0.2.100:34521

Step 10: NAT Gateway forwards response to private EC2 (10.0.2.100)
```

### Connection Tracking

NAT Gateway maintains a **connection tracking table** in memory. Each outbound connection gets an entry like:

```
Private IP:Port Public IP:Port External IP:Port
10.0.2.100:34521 52.10.5.100:51022 142.250.x.x:443
10.0.2.101:52341 52.10.5.100:51023 8.8.8.8:53
```

This is how NAT knows where to send return traffic.

### High Availability of NAT Gateway

- Each NAT Gateway is **tied to a single Availability Zone (AZ)**.
- If the AZ goes down, the NAT Gateway goes down too.
- **Best Practice:** Deploy **one NAT Gateway per AZ**, and update each AZ's private subnet route table to point to its own NAT Gateway.

```
AZ-1 Private Subnet NAT-GW-AZ1 IGW
AZ-2 Private Subnet NAT-GW-AZ2 IGW
AZ-3 Private Subnet NAT-GW-AZ3 IGW
```

This eliminates AZ cross-traffic costs and eliminates single points of failure.

---

## 6. Deep Dive How NAT Instance Works

### Step-by-Step Traffic Flow

```
Step 1: Private EC2 (10.0.2.100) wants to reach pypi.org

Step 2: Route Table for Private Subnet:
 Destination Target
 0.0.0.0/0 eni-xxxxxxxx (NAT Instance's ENI/Network Interface)

Step 3: Traffic arrives at NAT Instance (EC2 in public subnet)

Step 4: NAT Instance OS-level iptables NAT rule translates:
 Source IP: 10.0.2.100 54.200.x.x (NAT Instance public IP)

Step 5: NAT Instance forwards packet to Internet Gateway

Step 6: Response comes back to NAT Instance

Step 7: iptables NAT translates destination back to 10.0.2.100

Step 8: NAT Instance forwards response to Private EC2
```

### iptables Configuration on NAT Instance

Under the hood, the NAT AMI runs these Linux iptables rules:

```bash
# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# NAT rule masquerade all outbound traffic
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

The `MASQUERADE` target replaces the source IP with the EC2 instance's public IP automatically.

### What You Must Configure on NAT Instance

1. **Use the AWS NAT AMI** `amzn-ami-vpc-nat-*` in the AMI catalog.
2. **Disable Source/Destination Check** in EC2 settings.
3. **Assign an Elastic IP** so it has a stable public IP.
4. **Configure Security Group** allow inbound from private subnet CIDR, allow outbound to 0.0.0.0/0.
5. **Update Route Table** point private subnet's default route to the NAT instance's ENI (not instance ID).

---

## 7. Full Comparison Table

| Feature | NAT Gateway | NAT Instance |
|---|---|---|
| **Type** | Managed AWS Service | EC2 Instance (self-managed) |
| **Setup Complexity** | Very simple a few clicks | Complex many manual steps |
| **Availability** | High availability within AZ | Single point of failure (unless you set up HA manually) |
| **Failover** | Automatic (managed by AWS) | Manual (custom scripts or ASG required) |
| **Bandwidth** | Scales automatically (up to 100 Gbps) | Limited by instance type (e.g. t3.micro = very low) |
| **Performance** | Optimized by AWS | Depends on instance size and CPU |
| **Security Groups** | Not supported | Fully supported |
| **Network ACLs** | Supported | Supported |
| **SSH Access** | Not possible | Can SSH in and manage OS |
| **Port Forwarding** | Not supported | Supported via iptables |
| **Bastion Host** | Cannot be used as bastion | Can double as a bastion host |
| **Source/Dest Check** | Not applicable | Must be manually disabled |
| **Elastic IP Required** | Yes | Yes (recommended) |
| **OS Patching** | Not needed (managed by AWS) | You must patch manually |
| **Cost** | Higher (hourly + data processing fee) | Lower (just EC2 + EIP cost) |
| **Monitoring** | CloudWatch metrics built-in | CloudWatch agent needed |
| **Protocol Support** | TCP, UDP, ICMP | TCP, UDP, ICMP + custom protocols |
| **Custom Routing/Proxy** | Not possible | Can configure custom routes |
| **Deprecation Status** | Current & recommended | AWS-provided NAT AMIs are deprecated (Aug 2023) |
| **Use in Production** | Strongly recommended | Only for special use cases |

---

## 8. When to Use NAT Gateway

### Use NAT Gateway When...

#### 1. You Want Zero Maintenance Overhead
NAT Gateway is fully managed. AWS handles patching, scaling, and high availability. You set it up once and forget it. Perfect for teams that don't want to manage networking infrastructure.

#### 2. You Need High Performance and Scalability
NAT Gateway can handle **up to 100 Gbps** and scales automatically. If your workload has bursty or high-volume internet traffic from private instances, NAT Gateway handles it without any configuration.

#### 3. You Are Running Production Workloads
In production, downtime = money lost. NAT Gateway is the safe choice because it has **built-in redundancy** within its AZ. No single EC2 instance failure can take down your NAT.

#### 4. You Have Multiple Private Subnets at Scale
For large environments with dozens of subnets and thousands of instances, NAT Gateway is the only practical choice. A NAT Instance would bottleneck instantly.

#### 5. Your Team Lacks Deep Linux/Networking Knowledge
NAT Gateway requires zero Linux configuration. NAT Instance requires iptables knowledge, EC2 networking knowledge, and ongoing OS management.

#### 6. You Are Using EKS, ECS, or Serverless (Lambda)
Containers and serverless functions often need internet access for dependencies. NAT Gateway integrates cleanly without any extra complexity.

#### 7. You Want Built-in CloudWatch Metrics
NAT Gateway automatically sends metrics to CloudWatch:
- `BytesOutToDestination`
- `BytesOutToSource`
- `PacketsDropCount`
- `ErrorPortAllocation`

No setup required.

### Real-World Scenarios for NAT Gateway

- **E-commerce platform** with an RDS database in a private subnet that needs to send emails via SES or call payment APIs.
- **Microservices architecture** where dozens of services in private subnets call external APIs.
- **Data pipeline** where EMR/Glue in private subnets downloads datasets from public S3 endpoints or external sources.
- **Mobile backend** where hundreds of Lambda functions need internet access.

---

## 9. When to Use NAT Instance

### Use NAT Instance When...

#### 1. You Need to Minimize Costs (Small/Non-Production Workloads)
NAT Gateway charges **per hour + per GB of data processed**. For low-traffic environments (dev, staging, sandbox), a `t3.micro` or `t3.small` NAT Instance is significantly cheaper.

**Example Cost Comparison (US-East-1):**
```
NAT Gateway:
 - $0.045/hour = ~$32.40/month
 - + $0.045 per GB processed

NAT Instance (t3.micro):
 - $0.0104/hour = ~$7.50/month
 - No data processing fee
```

For a dev environment with low traffic, a NAT Instance saves **$25+/month per AZ**.

#### 2. You Need Port Forwarding
NAT Gateway does **NOT** support port forwarding. If you need to forward specific ports from the public internet to specific private instances (e.g., for SSH tunneling, custom protocols, or VPN), you need a NAT Instance with custom iptables rules.

```bash
# Example: forward public port 2222 to internal host 10.0.2.100:22
iptables -t nat -A PREROUTING -p tcp --dport 2222 -j DNAT --to-destination 10.0.2.100:22
iptables -t nat -A POSTROUTING -j MASQUERADE
```

#### 3. You Want to Use the Same Instance as a Bastion Host
A NAT Instance is a regular EC2 instance you can SSH into it. This means you can configure it as **both** a NAT device **and** a **Bastion/Jump Host** for accessing private resources. This reduces infrastructure and cost.

> Note: This is only advisable in small-scale or non-critical environments. In production, keep bastion hosts and NAT devices separate.

#### 4. You Need Custom Traffic Inspection / Filtering
Because NAT Instance gives you OS-level access, you can install tools like:
- `iptables` for custom packet filtering and port-based rules
- `squid` for HTTP proxy and URL filtering
- `fail2ban` to block suspicious IPs
- Custom VPN software

NAT Gateway offers none of this flexibility.

#### 5. You Need to Support Non-Standard Protocols
NAT Gateway only supports **TCP, UDP, and ICMP**. If you have applications using non-standard protocols (like custom GRE tunnels or OSPF), you'll need a NAT Instance.

#### 6. You Are in a Learning / Lab Environment
NAT Instances are excellent for understanding how NAT and networking actually work. If you're studying for the AWS SAA or SAP exam, setting up a NAT Instance manually teaches you much more than using a managed gateway.

### Real-World Scenarios for NAT Instance

- **Small startup** with a dev VPC where cost matters more than features.
- **Legacy application** that requires a specific port forwarding rule that NAT Gateway can't handle.
- **Security-conscious setup** needing a proxy layer to inspect or log outbound HTTP traffic.
- **Custom VPN solution** where the NAT device also terminates VPN tunnels.
- **Lab and learning environment** for studying networking.

---

## 10. Architecture Diagrams Text Based

### Architecture with NAT Gateway (Recommended for Production)

```
 
 AWS VPC (10.0.0.0/16) 
 
 
 Public Subnet (10.0.1.0/24) 
 
 [NAT Gateway] Elastic IP 
 
 
 
 
 Private Subnet (10.0.2.0/24) 
 
 [EC2 App Server] [EC2 App Server] 
 [RDS Database] [Lambda/ECS] 
 
 Route: 0.0.0.0/0 NAT-GW 
 
 
 
 [Internet Gateway (IGW)]
 
 INTERNET
```

### Architecture with NAT Instance

```
 
 AWS VPC (10.0.0.0/16) 
 
 
 Public Subnet (10.0.1.0/24) 
 
 [NAT Instance EC2] Elastic IP 
 (Source/Dest Check: DISABLED) 
 (Security Group configured) 
 (iptables MASQUERADE enabled) 
 
 
 
 
 Private Subnet (10.0.2.0/24) 
 
 [EC2 App Server] [EC2 App Server] 
 
 Route: 0.0.0.0/0 eni-xxxxxxxx 
 (NAT Instance ENI) 
 
 
 
 [Internet Gateway (IGW)]
 
 INTERNET
```

### Multi-AZ High Availability with NAT Gateway (Best Practice)

```
VPC (10.0.0.0/16)

 AZ-1 (ap-south-1a)
 Public Subnet: NAT-GW-1 (EIP: 52.x.x.1)
 Private Subnet: Route 0.0.0.0/0 NAT-GW-1

 AZ-2 (ap-south-1b)
 Public Subnet: NAT-GW-2 (EIP: 52.x.x.2)
 Private Subnet: Route 0.0.0.0/0 NAT-GW-2

 AZ-3 (ap-south-1c)
 Public Subnet: NAT-GW-3 (EIP: 52.x.x.3)
 Private Subnet: Route 0.0.0.0/0 NAT-GW-3
```

Each AZ has its own NAT Gateway. If one AZ fails, the other AZs are completely unaffected.

---

## 11. Step-by-Step Setup Overview

### Setting Up NAT Gateway

```
Step 1: Create a VPC with public and private subnets

Step 2: Attach an Internet Gateway to the VPC

Step 3: Go to VPC NAT Gateways Create NAT Gateway
 - Subnet: Select public subnet
 - Connectivity Type: Public
 - Elastic IP: Allocate new EIP

Step 4: Update Private Subnet Route Table
 - Destination: 0.0.0.0/0
 - Target: nat-xxxxxxxx (your NAT Gateway ID)

Step 5: Done. Test by SSH-ing into private instance and running:
 curl https://checkip.amazonaws.com
 # Should return the NAT Gateway's EIP
```

### Setting Up NAT Instance

```
Step 1: Launch an EC2 instance in the public subnet
 - AMI: search "amzn-ami-vpc-nat" in Community AMIs
 - Instance Type: t3.micro (dev) or t3.small/medium (prod)
 - Assign Elastic IP

Step 2: Disable Source/Destination Check
 - EC2 Console Instance Actions Networking
 Change Source/Destination Check Disabled

Step 3: Configure Security Group on NAT Instance
 Inbound:
 - All traffic from Private Subnet CIDR (e.g. 10.0.2.0/24)
 - SSH from your IP (for management)
 Outbound:
 - All traffic (0.0.0.0/0)

Step 4: Update Private Subnet Route Table
 - Destination: 0.0.0.0/0
 - Target: eni-xxxxxxxx (NAT Instance's network interface ID)
 Note: Point to the ENI, NOT the instance ID (for stability)

Step 5: Verify iptables on NAT Instance (SSH in)
 sudo iptables -t nat -L -n
 # Should see MASQUERADE rule on POSTROUTING chain

Step 6: Enable IP forwarding (if not already enabled)
 sudo sysctl net.ipv4.ip_forward
 # Should return: net.ipv4.ip_forward = 1
```

---

## 12. Cost Comparison

### NAT Gateway Pricing (us-east-1 example)

```
Hourly charge: $0.045 per hour
Monthly (720 hrs): ~$32.40 per NAT Gateway

Data processing: $0.045 per GB processed
(applies to all data that passes through the NAT Gateway)

Note: Standard EC2 data transfer rates still apply separately.
```

### NAT Instance Pricing (us-east-1 example)

```
t3.micro (dev): $0.0104/hour = ~$7.49/month
t3.small: $0.0208/hour = ~$14.98/month
t3.medium: $0.0416/hour = ~$29.95/month

Elastic IP: Free when attached to running instance

No data processing fee!
```

### Cost Scenario

**Scenario: 3 AZs, 100 GB/month data transfer**

| Option | Monthly Cost |
|---|---|
| 3x NAT Gateways | (3 $32.40) + (100 GB $0.045) = $101.70 |
| 1x NAT Instance (t3.small) | $14.98 + $0.00 (no processing fee) = $14.98 |
| 3x NAT Instances (t3.small, one per AZ) | 3 $14.98 = $44.94 |

> **Takeaway:** NAT Instance saves money at small scale. NAT Gateway saves time and operational overhead at any scale.

---

## 13. Security Considerations

### NAT Gateway Security

- NAT Gateway itself **cannot be assigned a Security Group**.
- Security is applied at the **NACL (Network Access Control List)** level on the subnet.
- Because it's managed, there's no OS to compromise or patch.
- The Elastic IP becomes a **known exit IP** you can whitelist it on external firewalls.

**Best Practices:**
- Use **NACLs** to restrict which private subnets can send traffic through the NAT Gateway.
- Monitor with **VPC Flow Logs** to detect unusual outbound traffic.
- Use **AWS Network Firewall** in front of the NAT Gateway for deep packet inspection.

### NAT Instance Security

- Security Groups ARE supported giving you more granular control.
- You can restrict traffic at the protocol and port level from both the source (private subnet) and destination (internet).
- The EC2 instance itself must be **kept patched** an unpatched NAT Instance is a security risk.
- If the instance is compromised, an attacker could intercept or redirect private subnet traffic.

**Best Practices:**
- Use the **latest Amazon Linux NAT AMI** and apply updates regularly.
- **Restrict SSH access** to a specific IP or use AWS Systems Manager Session Manager (no SSH needed).
- Use **CloudWatch Logs** and the CloudWatch agent to monitor instance-level metrics.
- Enable **GuardDuty** to detect suspicious network activity.
- Never store sensitive credentials or data on the NAT Instance.

---

## 14. Common Real-World Scenarios

### Scenario 1: Startup with Dev + Prod Environments

```
DEV Environment:
 Cost priority
 Low traffic, non-critical
 Use NAT Instance (t3.micro) saves ~$25/month per AZ

PROD Environment:
 Availability priority
 High traffic, mission-critical
 Use NAT Gateway (one per AZ) worth the cost for reliability
```

### Scenario 2: RDS Database Needs to Download Updates

```
Problem:
 RDS in private subnet needs to reach AWS update servers

Solution:
 Private Subnet Route NAT Gateway IGW Internet
 RDS never gets a public IP. Internet only sees NAT Gateway's EIP.
 NAT Gateway is ideal here no custom config needed
```

### Scenario 3: Private EC2 Needs to Call Stripe Payment API

```
Private App Server (10.0.2.50)
 NAT Gateway (52.10.5.100)
 api.stripe.com:443

Stripe sees requests from 52.10.5.100 (your EIP)
You whitelist this EIP on Stripe's webhook IP allowlist

 NAT Gateway is perfect consistent outbound IP, no maintenance
```

### Scenario 4: Need to SSH Tunnel into Private DB from Office

```
Problem:
 Developer needs to connect to RDS in private subnet from their laptop

Option A NAT Instance as Bastion:
 Laptop SSH to NAT Instance (Public IP) SSH tunnel to RDS
 NAT Instance doubles as bastion, saves one EC2 instance

Option B Separate Bastion Host + NAT Gateway:
 Laptop SSH to Bastion Host
 Private Subnet NAT Gateway Internet (for outbound)
 Cleaner architecture, better security separation
```

### Scenario 5: Compliance Audit Outbound Traffic

```
Requirement:
 All outbound internet traffic must be logged and inspected

With NAT Instance:
 Install Squid proxy on NAT Instance
 Configure iptables to redirect HTTP/HTTPS to Squid
 Squid logs all URLs, enforces allowlist
 NAT Instance gives this flexibility

With NAT Gateway:
 NAT Gateway alone cannot inspect traffic
 Use AWS Network Firewall in front of NAT Gateway for filtering
 More expensive but more scalable
```

---

## 15. Summary & Decision Guide

### Quick Decision Flowchart

```
Do you need outbound internet access from a private subnet?

 YES
 
 Is it a production/critical environment?
 YES Use NAT Gateway (one per AZ)
 NO Continue below...
 
 Is cost a primary concern?
 YES Consider NAT Instance (t3.micro or t3.small)
 NO Use NAT Gateway
 
 Do you need port forwarding or custom iptables rules?
 YES Use NAT Instance
 NO Use NAT Gateway
 
 Do you need a bastion host too?
 YES Consider combined NAT Instance + Bastion (small envs only)
 NO Use NAT Gateway
 
 Do you need traffic inspection/proxy?
 YES NAT Instance + Squid, OR NAT Gateway + AWS Network Firewall
 NO Use NAT Gateway
```

### Final Summary

| Use Case | Recommendation |
|---|---|
| Production workloads | NAT Gateway |
| High availability required | NAT Gateway (one per AZ) |
| Large scale / high bandwidth | NAT Gateway |
| Zero maintenance desired | NAT Gateway |
| Dev / sandbox / low traffic | NAT Instance (cost savings) |
| Port forwarding needed | NAT Instance |
| Bastion + NAT combined | NAT Instance (small envs) |
| Custom traffic filtering | NAT Instance + Squid proxy |
| Non-standard protocols | NAT Instance |
| Learning / lab setup | NAT Instance (better learning) |

### General Rule of Thumb

> **"If you can afford NAT Gateway, use NAT Gateway."**

NAT Gateway costs a bit more, but it gives you back hours of operational time, eliminates a single point of failure, and scales automatically. For most production AWS deployments, NAT Gateway is the standard and expected choice.

NAT Instance is the right tool when you have specific requirements it uniquely satisfies (port forwarding, proxy, custom protocols) or when you are working in a cost-sensitive non-production environment.

---

## Key Terminology Reference

| Term | Meaning |
|---|---|
| **NAT** | Network Address Translation translating private IPs to public |
| **SNAT** | Source NAT translating the source IP of outbound packets |
| **VPC** | Virtual Private Cloud your isolated network on AWS |
| **IGW** | Internet Gateway connects your VPC to the public internet |
| **EIP** | Elastic IP a static public IPv4 address |
| **NACL** | Network Access Control List subnet-level stateless firewall |
| **Security Group** | Instance-level stateful firewall |
| **ENI** | Elastic Network Interface a virtual network card on EC2 |
| **Route Table** | Defines where network traffic is directed |
| **Source/Dest Check** | AWS check that drops traffic not meant for the instance |
| **iptables** | Linux firewall and NAT tool used by NAT Instances |
| **MASQUERADE** | iptables target that performs SNAT using the interface's public IP |
| **Bastion Host** | A jump server used to SSH into private instances |
| **AZ** | Availability Zone a distinct data center within an AWS Region |

---

*Document prepared for AWS cloud architecture reference.* 
*Covers NAT Gateway and NAT Instance theory, comparison, and practical usage guidance.*
