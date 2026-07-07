# Deploying a Scalable Web Application on AWS using Auto Scaling & Load Balancer

A step-by-step guide to deploying a fault-tolerant, auto-scaling web application on AWS using an Auto Scaling Group (ASG) behind an Application Load Balancer (ALB).

📄 [Full tutorial PDF](./Auto_Scaling_Tutorial.pdf) 

---

## Introduction

Modern web applications need to be scalable, highly available, and fault-tolerant. This guide walks through designing and deploying a simple web application on AWS using an **Auto Scaling Group** behind an **Elastic Load Balancer**.

## Key Concepts

- **Auto Scaling** — automatically adjusts the number of EC2 instances in response to demand, keeping performance consistent at optimal cost.
- **Load Balancer** — distributes incoming traffic across multiple EC2 instances in multiple Availability Zones, so no single server gets overloaded.

In simple terms: Auto Scaling adds servers when traffic increases and removes them when traffic drops. The Load Balancer acts like a traffic controller, spreading requests evenly.

## Request Flow

```
User → Load Balancer → Auto Scaling Group → EC2 Instances
```

---

## Step-by-Step Implementation

### Step 1: Create a Launch Template
A Launch Template defines how EC2 instances get created automatically.

1. Go to **Amazon EC2 → Launch Templates**
2. Click **Create Launch Template**
3. Configure the template:
   - **Template name**: any name
   - **AMI**: select your desired AMI
   - **Instance type**: `t3.micro` (or an available alternative)
   - **Key pair**: select an existing one or create a new one
4. **Network Settings** — Security Group:
   - Allow HTTP (Port 80)
   - Allow SSH (Port 22)
5. Add a **User Data script** under Advanced Details:
   ```bash
   #!/bin/bash
   sudo su
   dnf install httpd -y
   systemctl start httpd
   echo "Hello from user" > /var/www/html/index.html
   ```
6. Click **Create Launch Template**

### Step 2: Create an Auto Scaling Group
An Auto Scaling Group automatically launches instances, maintains desired capacity, and distributes traffic via the Load Balancer.

1. Go to **Auto Scaling Groups → Create Auto Scaling Group**
2. Select your Launch Template
3. Configure network settings (VPC, Availability Zones, subnets)
4. Configure the Load Balancer:
   - Type: Application Load Balancer
   - Scheme: Internal (or Internet-facing, depending on your use case)
5. Configure the Target Group (creates a new one tied to the ALB)
6. Skip VPC Lattice (not needed for this setup)
7. Configure Group Size and Scaling:
   - **Desired capacity**: 3
   - **Min desired capacity**: 1
   - **Max desired capacity**: 3
8. Click **Create Auto Scaling Group**

### Step 3: Verify Deployment
1. Confirm 3 healthy instances are running, along with the Load Balancer and Target Group
2. Navigate to **EC2 → Load Balancers**, copy the **DNS name**
3. Open the DNS name in a browser — you should see the "Hello from user" response

### Step 4: Validate Auto Scaling
1. Connect to any running instance
2. Simulate load using the `stress` tool:
   ```bash
   yum install stress -y
   stress -c 10 -t 300
   ```
3. Open the **Monitoring** tab on the instance and watch **CPU Utilization** spike — this is what triggers Auto Scaling to add more instances automatically.

---

## Conclusion

This project demonstrates how to deploy a scalable web application using a Launch Template, Auto Scaling Group, and Application Load Balancer on AWS. The system automatically adjusts the number of instances based on CPU utilization and distributes traffic efficiently, ensuring high availability — with minimal manual intervention.

---

🔗 More AWS tutorials in this repo — check the [main README](../README.md) for the full list.
