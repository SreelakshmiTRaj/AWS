# AWS Multi-Region VPC Peering and Network Setup

A step-by-step guide to connecting two VPCs across different AWS regions using VPC Peering, with secure internet access for private resources via a NAT Gateway, and a LAMP stack server deployed in the public subnet.

📄 [Full tutorial PDF](./AWS_Multi-Region_VPC_Peering_and_Network_Setup__Sreelakshmi_T_Raj_.pdf)

---

## Introduction

Cloud environments rarely exist inside a single isolated network. Modern infrastructures are usually distributed across multiple regions, segmented into public and private networks, and designed with strict access control rules.

This project builds a multi-region AWS networking architecture connecting two VPCs across different regions using VPC Peering, while also implementing secure internet access for private resources using a NAT Gateway. A LAMP stack server is deployed inside the public subnet to complete the setup.

## Project Architecture

The infrastructure spans two AWS regions:

| Region | Purpose |
|--------|---------|
| Mumbai (ap-south-1) | Private application network |
| Tokyo (ap-northeast-1) | Public and internal network |

The Mumbai VPC contains a private subnet for internal communication, while the Tokyo VPC hosts both public and private workloads. Internet access for private resources is managed using a NAT Gateway, and cross-region connectivity is achieved through VPC Peering.

**Request flow for private-to-private communication:**
```
Mumbai Private Subnet ⇄ VPC Peering Connection ⇄ Tokyo Private Subnets
```

---

## Step-by-Step Implementation

### Step 1: Create VPC A in Mumbai Region with Private Subnet

1. Switch to region **Mumbai (ap-south-1)**
2. Go to **VPC Dashboard → Your VPCs → Create VPC**
   - Name: `VPC-A-Mumbai`
   - IPv4 CIDR block: `192.168.0.0/16`
3. Create a private subnet inside it:
   - VPC: `VPC-A-Mumbai`
   - Subnet name: `Private-Subnet-1(mb)`
   - Availability Zone: `ap-south-1a`
   - IPv4 CIDR block: `192.168.1.0/24`
   - Auto-assign public IP: disabled (kept private)
4. Create a route table for the subnet:
   - Name: `Private-RT-1(Mb)`
   - VPC: `VPC-A-Mumbai`
   - Keep the local route (`192.168.0.0/16 → Local`)
   - Associate with `Private-Subnet-1(Mb)`
   - No internet gateway attached — this subnet stays truly private

### Step 2: Create VPC B in Tokyo Region with Three Subnets

1. Switch to region **Tokyo (ap-northeast-1)**
2. Create the VPC:
   - Name: `VPC-B-Tokyo`
   - IPv4 CIDR block: `10.0.0.0/16`
3. Create an Internet Gateway and attach it to the VPC (needed since this VPC hosts public-facing resources):
   - Name: `IGW-Tokyo`
   - Attach to `VPC-B-Tokyo`
4. Create three subnets:
   - **Public-Subnet-1(Ty)** — `10.0.1.0/24`, auto-assign public IP enabled
   - **Private-Subnet-2(Ty)** — `10.0.2.0/24`
   - **Private-Subnet-3(Ty)** — `10.0.3.0/24`
5. Create route tables:
   - **Public-RT-Tokyo** → route `0.0.0.0/0` to the Internet Gateway → associate with `Public-Subnet-1(Ty)`
   - **Private-RT-2(Ty)** → associate with `Private-Subnet-2(Ty)`
   - **Private-RT-3-NAT(Ty)** → associate with `Private-Subnet-3(Ty)`

### Step 3: Configure NAT Gateway for Private Subnet Internet Access

A NAT Gateway allows a private subnet to initiate outbound internet connections while blocking inbound traffic.

1. Go to **NAT Gateways → Create NAT Gateway**
   - Subnet: `Public-Subnet-1(Ty)`
   - Elastic IP: allocate a new one
   - Name: `NAT-Gateway-Public`
2. Wait for status to become **Available**
3. Edit `Private-RT-3-NAT(Ty)`:
   - Add route: `0.0.0.0/0 → NAT Gateway`

### Step 4: Security Groups Configuration

| Security Group | VPC | Inbound Rules |
|----------------|-----|----------------|
| `SG-VPC-A-Private` | VPC-A-Mumbai | SSH (22), ALL ICMP — Anywhere IPv4 |
| `SG-VPC-B-Public` | VPC-B-Tokyo | SSH (22), HTTP (80), HTTPS (443) — Anywhere IPv4 |
| `SG-VPC-B-Private` | VPC-B-Tokyo | SSH (22), ALL ICMP — 0.0.0.0/0 |

Outbound rules were left as default for all groups.

### Step 5: Launching EC2 Instances

| Instance | VPC | Subnet | Security Group |
|----------|-----|--------|-----------------|
| `EC2-Mb-Pvt-Subnet` | VPC-A-Mumbai | Private-Subnet-1(Mb) | SG-VPC-A-Private |
| `EC2-Ty-Pvt-Subnet2` | VPC-B-Tokyo | Private-Subnet-2(Ty) | SG-VPC-B-Private |
| `EC2-Ty-Pvt-Subnet3` | VPC-B-Tokyo | Private-Subnet-3(Ty) | SG-VPC-B-Private |
| `EC2-Ty-Public` | VPC-B-Tokyo | Public-Subnet-1(Ty) | SG-VPC-B-Public |

All instances used `t3.micro` with Amazon Linux 2023.

### Step 6: Establishing Cross-Region VPC Peering

1. In **Mumbai** region, go to **VPC Dashboard → Peering Connections → Create Peering Connection**
   - Name: `Mumbai-Tokyo-Peering`
   - Requester VPC: `VPC-A-Mumbai`
   - Region: `ap-northeast-1 (Tokyo)`
   - Accepter VPC: `VPC-B-Tokyo`
2. Switch to **Tokyo** region, accept the pending peering request
3. Update route tables on both sides so traffic actually flows:
   - **Mumbai** — `Private-RT-1(Mb)`: add route `10.0.2.0/24 → Peering Connection`
   - **Tokyo** — `Private-RT-2(Ty)`: add route `192.168.1.0/24 → Peering Connection`

### Step 7: Testing the Peering Connection

Since the Mumbai EC2 instance sits in a private subnet with no public internet access, the Tokyo public EC2 instance was used as a **bastion host** to reach it securely:

1. Connect to `EC2-Ty-Public`
2. Copy the Tokyo key pair contents into a `key.pem` file and secure it:
   ```bash
   chmod 400 key.pem
   ```
3. SSH into the Tokyo private instance:
   ```bash
   ssh -i key.pem ec2-user@10.0.2.8
   ```
4. From there, SSH into the Mumbai private instance using its private IP:
   ```bash
   ssh -i key.pem ec2-user@192.168.1.223
   ```
5. Test communication with a ping — successful replies confirmed the peering connection was working end-to-end.

### Step 8: Deploying the LAMP Stack Server

1. Launch a new instance in `VPC-B-Tokyo`, `Public-Subnet-1(Ty)`:
   - Name: `Web-Server-LAMP`
   - AMI: Ubuntu Server (latest LTS)
   - Security Group: `SG-VPC-B-Public`
   - Auto-assign public IP: enabled
2. Install and configure Apache, MySQL, and PHP (see the [LAMP Stack tutorial](../lamp-stack-tutorial) for the full setup)
3. Verify by visiting `http://<public-ip>/database_test.php` — a successful database connection confirms the setup works.

---

## Conclusion

This project provided hands-on experience with how AWS networking components work together in a real-world environment — multi-region VPCs, secure subnet communication, NAT Gateway routing, VPC peering, bastion host access, and deploying a LAMP stack server. Together, these pieces demonstrate the practical side of building secure, scalable cloud infrastructure on AWS.
