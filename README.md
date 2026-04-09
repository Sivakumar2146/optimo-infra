# AWS Infrastructure Setup – IT Systems Assignment

## Time Taken
I estimated around 10–12 hours for this assignment. Most of the time went into getting the VPC networking right and making sure the RDS instance was properly isolated from the internet.

---

## What I Built

The goal was to set up a secure and simple infrastructure on AWS for an internal service — one application server and one PostgreSQL database, properly separated so the database is never exposed to the internet.

Here is the high-level picture of what I ended up with:

```
Internet
    |
    |  (port 80 only)
    |
[Internet Gateway]
    |
[Public Subnet - ap-south-1a]
    |
[EC2 - optimo-app-server]   <-- Nginx web server, t3.micro
    |
    |  (port 5432, internal VPC traffic only)
    |
[Private Subnet - ap-south-1b]
    |
[RDS PostgreSQL - optimo-db]  <-- db.t4g.micro, NOT public
```

Everything sits inside a custom VPC (`optimo-vpc-vpc`, CIDR `10.0.0.0/16`) in the Mumbai region (`ap-south-1`).

---

## How to Run This

### Prerequisites
- AWS account with access to EC2 and RDS
- The `optimo-key.pem` key file (shared separately)
- A terminal (Linux/Mac) or PuTTY on Windows

### Step 1 – SSH into the app server
```bash
chmod 400 optimo-key.pem
ssh -i optimo-key.pem ubuntu@3.109.4.117
```

### Step 2 – Check the web server is running
Open a browser and go to:
```
http://3.109.4.117
```
You should see the Nginx default page confirming the server is up.

### Step 3 – Connect to the database (from inside EC2)
```bash
sudo apt install postgresql-client -y

psql -h optimo-db.cjkww8kqmo4u.ap-south-1.rds.amazonaws.com \
     -U adminuser \
     -d postgres \
     -p 5432
```
Enter the password when prompted. You should get a `postgres=>` prompt.

Type `\q` to exit.

---

## Networking – How I Thought About It

### VPC and Subnets

I created a custom VPC instead of using the AWS default one. The default VPC is fine for quick testing but it is not suitable when you want proper isolation between components.

Inside the VPC I created 4 subnets across 2 availability zones:

| Subnet | Type | CIDR Block | Zone |
|---|---|---|---|
| optimo-vpc-subnet-public1 | Public | 10.0.0.0/20 | ap-south-1a |
| optimo-vpc-subnet-public2 | Public | 10.0.16.0/20 | ap-south-1b |
| optimo-vpc-subnet-private1 | Private | 10.0.128.0/20 | ap-south-1a |
| optimo-vpc-subnet-private2 | Private | 10.0.144.0/20 | ap-south-1b |

The app server (EC2) lives in the public subnet because it needs to be reachable from the internet on port 80. The database lives in the private subnet — there is no route from the internet to the private subnet at all.

### Internet Gateway and Routing

I attached an Internet Gateway to the VPC and created a route table for the public subnet that sends all outbound traffic (`0.0.0.0/0`) through it. The private subnet has no such route — it can only communicate within the VPC.

### Why No NAT Gateway?

A NAT Gateway would allow the private subnet to reach the internet for things like OS updates, without being publicly accessible. I chose not to add one here because it costs around $32 per month and is not needed for this assignment. In a real production setup I would add it.

### EC2 to RDS Communication

I configured the RDS instance to allow access only from the EC2 instance using security group rules. I created a dedicated security group for the RDS instance that allows access only from the application server security group on port 5432.
---

## Security

This was the part I spent the most time thinking about.

### Security Groups

**App server (`optimo-app-sg`):**

| Protocol | Port | Source | Why |
|---|---|---|---|
| TCP | 80 | 0.0.0.0/0 | Web traffic from internet |
| TCP | 22 | 115.108.41.182/32 | SSH from my IP only |

**Database (`rds-ec2-2`):**

| Protocol | Port | Source | Why |
|---|---|---|---|
| TCP | 5432 | optimo-app-sg only | Only EC2 can connect to DB |

### Key Decisions I Made

**Database not public.** RDS is set to `Publicly Accessible = No`. Even if someone has the endpoint URL, they cannot connect from outside the VPC. This is the most important control in the whole setup.

**SSH locked to my IP.** Port 22 is only open to `115.108.41.182/32`. It is not open to the internet. This prevents brute force attacks.

**Least privilege.** Each security group allows only the minimum traffic needed. The DB security group only allows port 5432, and only from the app server's security group — not from any IP range.

**Credentials not in code.** The database password is not hardcoded anywhere in the codebase. In production I would store it in AWS Secrets Manager and retrieve it at runtime. For this assignment it is kept outside the codebase.

**Encryption at rest.** RDS encryption is enabled using AWS-managed KMS keys. This was on by default and I kept it enabled.

### Trade-offs

I did not set up HTTPS. For production I would use ACM with a load balancer to terminate SSL. For this demo with a plain IP address, HTTP is acceptable.

SSH is still open on port 22. In a more hardened setup I would remove SSH entirely and use AWS Systems Manager Session Manager instead, which gives shell access without opening any inbound ports.

No WAF is configured. For a public-facing API I would put AWS WAF in front of the load balancer to filter malicious traffic.

### Monitoring

EC2 basic monitoring is enabled — CloudWatch collects CPU, network, and disk metrics every 5 minutes. RDS has Performance Insights and Database Insights Standard enabled so I can see query performance and connection counts.

In production I would add CloudWatch Alarms: alert if CPU goes above 80%, if free storage drops below 20%, or if there are repeated failed login attempts on the database.

---

## DevOps – Making This Repeatable

I set everything up manually through the AWS Console. I did this on purpose so I could understand each component properly before writing it as code.

In a real environment I would use Terraform to define all of this infrastructure as code. The main benefit is repeatability — if the environment needs to be recreated, anyone can run `terraform apply` and get the exact same setup. All changes go through Git and can be reviewed before applying.

A basic Terraform structure would look like this:

```
infrastructure/
├── main.tf              # AWS provider, region
├── vpc.tf               # VPC, subnets, IGW, route tables
├── security_groups.tf   # Firewall rules
├── ec2.tf               # App server
├── rds.tf               # PostgreSQL instance
└── variables.tf         # Environment-specific values
```

For deployments, a simple script handles pushing updates to the server:

```bash
#!/bin/bash
# scripts/deploy.sh

SERVER="ubuntu@3.109.4.117"
KEY="optimo-key.pem"

echo "Starting deployment..."
ssh -i $KEY $SERVER "sudo systemctl restart nginx"
echo "Done at $(date)"
```

For CI/CD I would use GitHub Actions — automatic deployment whenever code is merged to the main branch, with a rollback step if the health check fails after deployment.

---

## Scalability and Reliability

The current setup uses a single EC2 and a single RDS instance. It works but has obvious single points of failure. Here is how I would scale it.

### Application Layer

Replace the single EC2 with an Auto Scaling Group behind an Application Load Balancer. The ASG would keep at least 2 instances running across different availability zones. It scales out when CPU goes above 70% and scales in when it drops below 30%. The load balancer does health checks and stops sending traffic to unhealthy instances automatically.

### Database Layer

Enable RDS Multi-AZ. AWS keeps a standby replica in a second availability zone and automatically fails over if the primary goes down — usually within 60 seconds. For read-heavy workloads, a Read Replica handles reporting queries so they don't compete with write traffic on the main instance.

### Where Bottlenecks Happen

The database is almost always the first bottleneck. When traffic grows, the app servers can scale horizontally but database connections pile up fast. Adding a connection pooler like PgBouncer between the app and the database helps a lot — it reuses existing connections instead of opening a new one for every request.

### Failure Handling

| What Fails | Today | In Production |
|---|---|---|
| EC2 instance goes down | App is down | ASG replaces it automatically |
| RDS primary fails | DB is unavailable | Multi-AZ failover in ~60 seconds |
| Full AZ outage | Partial outage | Multi-AZ + multi-subnet spans two zones |
| Disk fills up | DB stops accepting writes | Storage auto scaling + CloudWatch alert |

---

## Cost Estimate

### What Is Running Right Now

| Resource | Spec | Monthly Cost |
|---|---|---|
| EC2 instance | t3.micro, on-demand | ~$8 |
| RDS PostgreSQL | db.t4g.micro | ~$15 |
| EBS (EC2 disk) | 8 GiB gp2 | ~$1 |
| RDS storage | 20 GiB gp2 | ~$2.30 |
| Data transfer | Outbound | ~$1–2 |
| **Total** | | **~$27–28/month** |

Since this is a new AWS account, EC2 and RDS likely fall within the Free Tier for the first 12 months — so the actual cost right now is close to zero.

### Things That Could Unexpectedly Cost Money

**NAT Gateway** – I did not create one. If I had, it would cost $32 per month just for having it running, plus $0.045 per GB of traffic. It adds up fast if you are not watching it.

**RDS backups** – AWS keeps backup storage equal to your DB size for free. If you increase the retention period beyond 7 days, extra storage is charged.

**Data transfer** – Traffic inside the same VPC is free. Outbound traffic to the internet is charged after 100 GB per month.

**Idle Elastic IPs** – AWS charges for Elastic IPs that are allocated but not attached to a running instance. I used the auto-assigned public IP to avoid this.

### If This Were Scaled for Production

| Resource | Spec | Monthly Cost |
|---|---|---|
| Application Load Balancer | Standard | ~$20 |
| 2x EC2 t3.small | Auto Scaling Group | ~$30 |
| RDS Multi-AZ db.t3.small | PostgreSQL | ~$55 |
| NAT Gateway | Single AZ | ~$32 |
| CloudWatch logs and alarms | Standard | ~$5 |
| **Total** | | **~$142/month** |

To manage costs in production I would set up AWS Budgets with an email alert at 80% of the monthly limit, and use Savings Plans for compute — which can cut EC2 and RDS costs by 30–40% on predictable workloads.

---

## Assumptions and Trade-offs

**Single EC2, no load balancer** – A load balancer only makes sense when you have more than one app instance. I described how I would add it when scaling up.

**No HTTPS** – Setting up SSL requires a domain name and an ACM certificate. Since this uses a plain IP address, HTTP is fine for the demo. Production must have HTTPS.

**Manual setup** – I provisioned everything manually to show that I understand each AWS component. The logical next step would be converting it all to Terraform.

**Nginx as the app layer** – The focus of this assignment was infrastructure, not application code. Nginx is running and accessible at the public IP as the application layer.

**No Multi-AZ on RDS** – This saves money for the demo. In production, enabling Multi-AZ would be the first thing I do.

---

## Images

All screenshots of the AWS Console setup are in the `/images` folder:

- `01-vpc-created.png` – Custom VPC and all subnets created
- `02-app-security-group.png` – Security group rules for EC2
- `03-db-security-group.png` – Security group rules for RDS
- `04-ec2-running.png` – EC2 instance in running state
- `05-ec2-ssh-connected.png` – Terminal connected via SSH
- `06-app-running-browser.png` – Nginx running at public IP
- `07-rds-available.png` – RDS instance status available
- `08-rds-connectivity.png` – RDS endpoint and EC2 connection confirmed
- `09-db-connection-success.png` – Successful psql connection from EC2
