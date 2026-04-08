# AWS Infrastructure Assignment – Secure Web App Deployment

## 📌 Overview

This project demonstrates a secure and production-oriented AWS infrastructure for hosting a web application with a PostgreSQL database.

The architecture follows best practices:

* Separation of public and private resources
* Controlled network access
* Secure database deployment

---

## 🏗️ Architecture Overview

### Components Used:

* Custom VPC (10.0.0.0/16)
* Public Subnet (Application Layer)
* Private Subnet (Database Layer)
* EC2 Instance (Ubuntu + Nginx)
* RDS PostgreSQL
* Internet Gateway
* Security Groups

### Architecture Flow:

Client → Internet → EC2 (Public Subnet) → RDS (Private Subnet)

---

## 📸 Infrastructure Proof

### 1. VPC and Subnet Design

<img width="733" height="365" alt="01-vpc-created" src="https://github.com/user-attachments/assets/c30075b4-b929-4fe6-b726-cdb7a2c8a74e" />


---

### 2. Security Groups Configuration

#### App Security Group (EC2)

<img width="665" height="202" alt="02-app-security-group" src="https://github.com/user-attachments/assets/7b11cafd-bf62-4979-8325-623ed6ba89b8" />


#### Database Security Group (RDS)

<img width="635" height="183" alt="03-db-security-group" src="https://github.com/user-attachments/assets/32cd7454-f38f-4077-b444-18acc9e47014" />


---

### 3. EC2 Instance Running

<img width="869" height="410" alt="04-ec2-running" src="https://github.com/user-attachments/assets/9b9fba0a-8a65-42dd-a9d8-8583b9b76557" />


---

### 4. SSH Access to EC2

<img width="865" height="218" alt="05-ec2-ssh-connected" src="https://github.com/user-attachments/assets/cca95bef-f3b4-43cc-a37e-3fb5b750ed2a" />


---

### 5. Web Server Running (Nginx)

<img width="943" height="261" alt="06-app-running-browser" src="https://github.com/user-attachments/assets/f2546997-22be-4f2f-a394-e1fbda17def2" />


---

### 6. RDS Instance Available

<img width="836" height="23" alt="07-rds-available" src="https://github.com/user-attachments/assets/2680e3ee-5887-4121-88c4-2593fd752cca" />


---

### 7. EC2 to RDS Connectivity

<img width="842" height="404" alt="08-rds-connectivity" src="https://github.com/user-attachments/assets/10455ea8-9749-49b2-9b34-c5fe0bb74b2a" />


---

### 8. PostgreSQL Connection Success

<img width="576" height="304" alt="09-db-connection-success" src="https://github.com/user-attachments/assets/ad578dd0-e692-4a7c-b6ec-a0adcc9c3838" />


---

## 🌐 Networking Design

### VPC

* CIDR: 10.0.0.0/16

### Subnets

* Public Subnet:

  * Hosts EC2
  * Internet access via Internet Gateway
* Private Subnet:

  * Hosts RDS
  * No direct internet access

### Routing

* Public Route Table:

  * 0.0.0.0/0 → Internet Gateway
* Private Route Table:

  * No internet route (isolated)

### Design Decisions

* Database is placed in private subnet to improve security
* Only EC2 can access RDS using Security Group rules

---

## 🔐 Security Design

### EC2 Security Group

* Allow HTTP (80) from anywhere
* Allow SSH (22) only from my IP
* Outbound: Allow all

### RDS Security Group

* Allow PostgreSQL (5432) only from EC2 Security Group

### Security Best Practices

* Database is not publicly accessible
* No open database ports to internet
* Restricted SSH access
* Internal communication via private IP

---

## ⚙️ Setup Steps (Manual)

### Step 1: Create VPC

* CIDR: 10.0.0.0/16

### Step 2: Create Subnets

* Public Subnet (for EC2)
* Private Subnet (for RDS)

### Step 3: Configure Internet Gateway

* Attach IGW to VPC
* Associate with public route table

### Step 4: Launch EC2 Instance

* Ubuntu AMI
* Install Nginx:

```bash
sudo apt update
sudo apt install nginx -y
```

### Step 5: Create RDS PostgreSQL

* Private subnet
* Disable public access

### Step 6: Configure Security Groups

* EC2 → allow HTTP & SSH
* RDS → allow only EC2 SG

### Step 7: Connect EC2 to RDS

```bash
psql -h <rds-endpoint> -U adminuser -d postgres -p 5432
```

---

## 🔄 DevOps Practices

### Current Approach

* Infrastructure created manually via AWS Console

### Improvements (Future)

* Use Terraform for Infrastructure as Code
* Automate provisioning
* Version control infrastructure

---

## 📈 Scalability & Reliability

### Current Limitations

* Single EC2 instance
* Single RDS instance

### Scaling Strategy

* Add Application Load Balancer (ALB)
* Use Auto Scaling Group
* Enable RDS Multi-AZ

### Reliability Improvements

* Health checks via ALB
* Automated failover for DB

---

## 💰 Cost Estimation

### Monthly Cost (Approx)

| Service            | Cost           |
| ------------------ | -------------- |
| EC2 (t3.micro)     | ~$8            |
| RDS (db.t4g.micro) | ~$15           |
| Storage            | ~$5            |
| Data Transfer      | ~$2            |
| **Total**          | **~$30/month** |

### Cost Optimization

* No NAT Gateway (avoided high cost)
* Used small instance sizes
* Minimal resources

### Hidden Costs

* NAT Gateway (~$30/month)
* Data transfer charges
* Storage scaling

---

## ⚠️ Trade-offs

* Did not use NAT Gateway to reduce cost
* No Load Balancer (kept simple)
* Manual setup instead of IaC

---

## 🚀 Future Enhancements

* Terraform implementation
* CI/CD pipeline
* HTTPS (SSL via ACM)
* Secrets Manager for credentials
* CloudWatch monitoring

---

## ✅ Conclusion

This project demonstrates a secure AWS infrastructure with proper network isolation, controlled access, and production-oriented design decisions while keeping cost efficiency in mind.

---
