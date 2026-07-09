# Implementing Dynamic Auto Scaling on AWS Using Target Tracking Policy

A step-by-step guide to configuring a Target Tracking Scaling Policy on AWS, so EC2 capacity adjusts automatically based on real CPU demand instead of staying fixed.

📄 [Full tutorial PDF](./Implementing_Dynamic_Auto_Scaling_on_AWS_using_Target_Tracking_Policy.pdf)

---

## Overview

This tutorial covers configuring a Target Tracking Scaling Policy on AWS, which automatically adjusts the number of EC2 instances based on CPU utilization. Instead of a fixed capacity that stays constant regardless of load, this setup reacts to real time demand, scaling out under load and back in once it subsides, within a defined capacity ceiling that keeps cost predictable.

## Why Target Tracking Scaling?

A fixed capacity Auto Scaling Group keeps a set number of instances running at all times, useful for predictable, steady workloads, but wasteful during low traffic and risky during unexpected spikes, since it can't respond on its own.

Target Tracking Scaling solves this by using CloudWatch metrics (like average CPU utilization) to automatically adjust capacity toward a target value you define. It helps balance performance and cost, scaling out only when demand actually increases, and scaling back in once it drops, rather than running excess instances around the clock.

## Technology Stack

| Technology | Purpose |
|------------|---------|
| Amazon EC2 | Compute hosting |
| Auto Scaling Group | Instance management |
| Launch Template | Instance configuration |
| Application Load Balancer | Traffic distribution |
| Amazon CloudWatch | Monitoring |
| Amazon Linux | Operating system |
| `stress` | Load testing |

## Objectives

- Configure a Target Tracking Scaling Policy
- Monitor Auto Scaling behavior using CloudWatch metrics
- Simulate application load using `stress`
- Verify automatic scale-out and scale-in
- Understand how Target Tracking helps reduce over-provisioning

## Prerequisites

- AWS Account
- Basic knowledge of Launch Templates and Auto Scaling
- Familiarity with EC2 and Application Load Balancers

> **Note:** This tutorial recreates the infrastructure required to demonstrate Target Tracking Scaling. For the basic structure of Launch Templates, Auto Scaling Groups, and Load Balancers, refer to the [Auto Scaling tutorial](../auto-scaling-tutorial) in this repo.

---

## Step-by-Step Implementation

### Step 1: Create the Launch Template

A Launch Template defines how new EC2 instances get built automatically, so the Auto Scaling Group knows what to launch.

- Go to **Amazon EC2 → Launch Templates → Create Launch Template**
- Configure the template:
  - Name: `ec2-launch-template`
  - AMI: Amazon Linux
  - Instance type: `t3.micro` (or available alternative)
  - Key pair: select existing or create a new one
  - Security group: allow HTTP (80), SSH (22)
  - User data:
    ```bash
    #!/bin/bash
    sudo su
    dnf install httpd -y
    systemctl start httpd
    echo "<h1>Target Tracking Auto Scaling Demo</h1> <p>This EC2 instance was launched automatically by an Auto Scaling Group.</p> <p>Scaling is managed using a Target Tracking Policy based on CPU Utilization.</p>" > /var/www/html/index.html
    ```

### Step 2: Create the Auto Scaling Group

This is where the actual scaling behavior starts to take shape, the group manages how many instances run at any time.

- Go to **Auto Scaling Groups → Create Auto Scaling Group**
- Configure:
  - Name: `web-server-asg`
  - Launch Template: select the one created in Step 1
  - VPC/Subnets: preferred VPC and at least two Availability Zones
  - Load Balancing: attach to a new Application Load Balancer
  - Scheme: Internet-facing
  - Target Group: new, default HTTP (Port 80) listener
  - Minimum capacity: 1
  - Desired capacity: 1
  - Maximum capacity: 2 (or 3)

### Step 3: Configure the Target Tracking Scaling Policy

The target tracking policy watches CPU usage and adjusts capacity to stay near the target you set.

- Open the Auto Scaling Group → **Automatic scaling** tab
- Click **Create dynamic scaling policy**
- Configure:
  - Policy type: Target Tracking
  - Metric: Average CPU Utilization
  - Target value: 50%
  - Instance warm-up: default
  - Disable scale-in: unchecked

### Step 4: Deploy and Verify

Before testing scaling, confirm the base setup actually works.

- Navigate to **EC2 → Load Balancers**, copy the DNS name
- Open the DNS name in a browser to confirm the app loads
- Check the Target Group to confirm targets are healthy

### Step 5: Generate CPU Load

- Connect to a running instance
- Install the load testing tool:
  ```bash
  sudo yum install stress -y
  ```
- Run a sustained CPU load test:
  ```bash
  stress --cpu 2 --timeout 600
  ```

### Step 6: Monitor Scaling

- Open the instance's **Monitoring** tab, watch CPU climb toward and past 50%
- Open the Auto Scaling Group's **Activity** tab, watch for a scale-out event
- Watch desired capacity increase (1 → 2)

> **Note:** scaling isn't instant, AWS waits for the metric to stay above target for a sustained period, then applies a cooldown period before evaluating again.

### Step 7: Verify Scale-In

- Stop the stress test (or let the timeout expire)
- Wait for CPU to drop back to baseline on the Monitoring tab
- Open the **Activity** tab, watch for a scale-in event
- Watch desired capacity drop back down (2 → 1)

The Auto Scaling Group automatically removes the extra instance once CPU stays below target long enough, no manual intervention needed.

---

## Production Considerations

Target Tracking is commonly combined with:

- Minimum and maximum capacity limits, so cost stays within a known ceiling even if the policy overshoots
- Scheduled Scaling, to handle predictable traffic patterns (like business hours) in advance, rather than only reacting after load increases
- CloudWatch alarms, to monitor broader application health beyond CPU, such as memory or error rates
- Load Balancer health checks, so traffic is only routed to instances that are actually ready to handle it

---

## Conclusion

A fixed-capacity Auto Scaling Group works, but it doesn't actually respond to demand, it just holds a set number of instances regardless of load. Configuring a Target Tracking Policy changes that. In this setup, a stress test pushed CPU past the 50% target, the group scaled out from 1 to 2 instances, and once load dropped, it scaled back in on its own after the cooldown period, all while the maximum capacity setting kept a ceiling on cost. This is the core idea behind Target Tracking Scaling: capacity that moves with real demand instead of staying static, without losing control over how far it can scale.
