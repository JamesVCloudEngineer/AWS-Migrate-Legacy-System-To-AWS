# Custom VPC Build: Single-AZ Architecture with WordPress Deployment

**Author:** James Victor
**Status:** Phase 1 complete (single-AZ). Multi-AZ redundancy planned as the next phase.

## Executive Summary

This project builds a custom Amazon VPC from scratch and deploys a WordPress application into it, replacing reliance on the AWS default VPC. The goal was to demonstrate core networking fundamentals (CIDR planning, public/private subnet separation, route table logic, and security boundaries) while running a real application on purpose-built infrastructure.

The finished environment consists of one custom VPC, one public subnet, one private subnet, a dedicated route table for each, and an EC2 instance running a self-hosted WordPress site on a LAMP stack. The design is intentionally single-AZ for this first phase, with multi-AZ redundancy identified as the next iteration.

## Architecture Overview

```
                              Internet
                                 │
                          ┌──────┴───────┐
                          │   Internet   │
                          │   Gateway    │
                          └──────┬───────┘
                                 │
        VPC: james-project-vpc-vpc (10.0.0.0/16)
        us-east-1a (single Availability Zone)
                                 │
        ┌────────────────────────┴───────────────────┐
        │                                            │
┌───────▼─────────────────┐           ┌──────────────▼───────────┐
│  PUBLIC SUBNET          │           │  PRIVATE SUBNET          │
│  10.0.0.0/20            │           │  10.0.128.0/20           │
│  rtb: ...rtb-public     │           │  rtb: ...rtb-private1    │
│                         │           │                          │
│  ┌───────────────────┐  │           │  (reserved for future    │
│  │ EC2: james-project│  │           │   private EC2 / RDS      │
│  │ -ec2 (t3.micro)   │  │           │   tier)                  │
│  │ WordPress + Apache│  │           │                          │
│  │ Elastic IP        │  │           │                          │
│  └───────────────────┘  │           │                          │
└─────────────────────────┘           └──────────────────────────┘
```

Network ACL: one shared ACL across both subnets (default allow-all baseline).

## CIDR & Subnet Strategy

| Resource | CIDR | Addresses | Rationale |
|---|---|---|---|
| VPC | 10.0.0.0/16 | 65,536 | Standard private range, large enough for future subnets (multi-AZ, additional tiers) without re-architecting |
| Public Subnet | 10.0.0.0/20 | 4,096 (4,091 usable) | Sized above a minimal /24 to leave room for load balancers, NAT gateways, and additional web or bastion instances |
| Private Subnet | 10.0.128.0/20 | 4,096 (4,091 usable) | Non-overlapping block, isolated from direct internet exposure, reserved for database and internal application tiers |

**Design decision:** The /20 sizing was a deliberate scalability choice. It costs nothing in a VPC this size, and subnets cannot be resized after creation, so under-provisioning early means rebuilding later. AWS reserves 5 addresses in every subnet, which is why 4,091 are usable.

## Route Table Logic

| Route Table | Associated Subnet | Routes | Purpose |
|---|---|---|---|
| james-project-vpc-rtb-public | Public (10.0.0.0/20) | local, 0.0.0.0/0 → Internet Gateway | Public subnet resources can send and receive internet traffic |
| james-project-vpc-rtb-private1-us-east-1a | Private (10.0.128.0/20) | local only | Private resources can reach the rest of the VPC but nothing outside it. A 0.0.0.0/0 → NAT Gateway route is planned for outbound-only access (patching, package installs) |

Each subnet has its own dedicated route table rather than sharing one, which is the correct pattern for enforcing different traffic rules per network tier.

## Security Configuration

- **Network ACL:** A single shared ACL across both subnets for this phase. A production environment would use separate, more restrictive ACLs per tier (for example, blocking inbound SSH at the private subnet entirely).
- **Security group:** The instance's security group allows HTTP (80) for the website and SSH (22) for administration. Security groups are stateful and apply per instance, while the NACL is stateless and applies per subnet. SSH should be restricted to a single admin IP.
- **Public IP assignment:** Auto-assign public IPv4 was found set to **No** on the public subnet, so the instance was given an Elastic IP manually. Recommended fix: enable auto-assign public IPv4 on the public subnet so future instances get a public IP at launch.
- **Database credentials:** wp-config.php was configured with the MySQL root user and a blank password during initial setup. This is flagged as a hardening item. Production practice is a dedicated least-privilege MySQL user with a strong password stored in AWS Secrets Manager, not in wp-config.php.

## Deployment Steps

### 1. VPC & Subnet Creation (Console)

1. VPC Dashboard → Create VPC
2. CIDR block `10.0.0.0/16`, name `james-project-vpc-vpc`
3. Public subnet `10.0.0.0/20` in us-east-1a
4. Private subnet `10.0.128.0/20` in us-east-1a
5. Create an Internet Gateway and attach it to the VPC
6. Create `rtb-public`, add route `0.0.0.0/0 → IGW`, associate it with the public subnet
7. Create `rtb-private1-us-east-1a`, associate it with the private subnet

### 2. EC2 Instance Launch

Launched from the console: Ubuntu Server, t3.micro, in the public subnet, key pair `james-project-key`. An Elastic IP was attached because auto-assign public IP was disabled on the subnet.

### 3. Connect and Update the Server

```bash
ssh -i ~/Desktop/james-key.pem ubuntu@<elastic-ip>
sudo apt update && sudo apt upgrade -y
# Reboot if a kernel upgrade is pending
```

### 4. Install the LAMP Stack

```bash
sudo apt install apache2 mysql-server -y
sudo apt install php libapache2-mod-php php-mysql php-curl php-xml php-mbstring php-zip php-gd -y
sudo systemctl enable --now apache2 mysql
```

Apache's default Ubuntu page confirmed the web server was serving correctly before WordPress took over the document root.

### 5. Install WordPress

```bash
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
sudo mv wordpress/* .
sudo rm index.html latest.tar.gz
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 755 /var/www/html
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php    # set DB_NAME, DB_USER, DB_PASSWORD
sudo systemctl restart apache2
```

### 6. AWS CLI Verification Commands

```bash
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=james-project-vpc-vpc"
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>"
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>"
aws ec2 describe-instances --filters "Name=tag:Name,Values=james-project-ec2"
```

## Cost Analysis

| Resource | Type | Est. Monthly Cost (us-east-1) |
|---|---|---|
| EC2 Instance | t3.micro | ~$7.50 (or $0 under Free Tier, 750 hrs/month) |
| EBS Volume | 8 GiB gp3 | ~$0.64 |
| Elastic IP | 1 (attached) | ~$3.60 (AWS bills every public IPv4 address since February 2024, attached or not) |
| VPC, Subnets, Route Tables, IGW | N/A | $0 (no charge for these resources) |
| NAT Gateway (planned, not deployed) | N/A | ~$32.40/month + data processing; the main cost driver to plan for |

**Cost notes:** The single-instance, single-AZ design keeps this project near Free Tier limits. The NAT Gateway is the main future cost because it bills hourly whether or not traffic flows. A NAT instance is a cheaper option for low-traffic dev environments, traded off against lower availability and manual patching. Stopping the instance and releasing the Elastic IP when the environment is idle avoids ongoing charges.

## Interview Talking Points

- **Why a custom VPC instead of the default?** Full control over CIDR planning, subnet segmentation, and routing. The default VPC is flat and public by design, which isn't appropriate beyond a quick test.
- **Security Groups vs. NACLs:** Security groups are stateful and apply at the instance (ENI) level. NACLs are stateless and apply at the subnet level. A NACL deny cannot be overridden by a permissive security group, so defense in depth requires both.
- **What makes a subnet public?** Not its name. It takes an Internet Gateway route in the route table and a public IP on the instance. This project surfaced that directly when auto-assign public IP was found disabled on the "public" subnet.
- **Single-AZ limitation:** Intentional for this phase. The next step is a second AZ with a duplicate public/private subnet pair, plus an Application Load Balancer and Multi-AZ RDS to remove the single point of failure.
- **NAT Gateway vs. Internet Gateway:** An IGW allows two-way traffic for public subnet resources. A NAT Gateway allows outbound-only traffic for private subnet resources, which is how you patch a private database server without exposing it to inbound connections.

## Lessons Learned

- **SSH troubleshooting:** SSH attempts failed with `Permission denied (publickey)` and `command not found`. The cause was stray characters (`00~`, `01~`) inserted into the terminal when pasting the command. Retyping the command cleanly fixed it. Confirming the prompt read `ubuntu@ip-...` before running server commands became a habit.
- **Kernel and package drift:** The instance flagged a pending kernel upgrade and 12 package updates right after launch. A fresh EC2 instance still needs an `apt update && apt upgrade` pass (and a reboot for kernel updates) before it's production-ready.
- **Defaults aren't always what they seem:** Naming a subnet "public" doesn't make it public. Auto-assign public IP has to be enabled explicitly, which was missed in the initial build and caught during documentation review.

## Next Phase

1. **Multi-AZ redundancy:** duplicate the public/private subnet pair into us-east-1b.
2. **Enable auto-assign public IPv4** on the public subnet.
3. **Harden database credentials:** a dedicated least-privilege MySQL user, with the password in AWS Secrets Manager.
4. **Move the database to RDS** in the private subnet.
5. **Deploy a NAT Gateway** (or NAT instance for dev) for the private tier's outbound access.
6. **Separate NACLs per tier,** with the private subnet locked down more tightly than the public one.
7. **Restrict SSH** in the security group to a single admin IP.

## Documentation

- `James_Victor_Custom_VPC_Build_Report.docx`: full build report with annotated console and terminal screenshots
