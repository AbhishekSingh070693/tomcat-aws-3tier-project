# Tomcat AWS 3-Tier Architecture - Complete Project Documentation

## Author: Abhishek Singh
## Date: June 2026
## Project: Production-grade 3-Tier AWS Architecture with Monitoring

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Prerequisites](#3-prerequisites)
4. [VPC and Networking Setup](#4-vpc-and-networking-setup)
5. [Security Groups](#5-security-groups)
6. [EC2 Instance Setup](#6-ec2-instance-setup)
7. [Application Configuration](#7-application-configuration)
8. [RDS Database Setup](#8-rds-database-setup)
9. [Database Import](#9-database-import)
10. [Tomcat systemd Service](#10-tomcat-systemd-service)
11. [AMI Creation](#11-ami-creation)
12. [Launch Template](#12-launch-template)
13. [Target Group](#13-target-group)
14. [Application Load Balancer](#14-application-load-balancer)
15. [Auto Scaling Group](#15-auto-scaling-group)
16. [Prometheus Installation](#16-prometheus-installation)
17. [Grafana Installation](#17-grafana-installation)
18. [node_exporter Installation](#18-node_exporter-installation)
19. [Prometheus Configuration](#19-prometheus-configuration)
20. [Grafana Dashboard Setup](#20-grafana-dashboard-setup)
21. [Challenges Faced and Solutions](#21-challenges-faced-and-solutions)
22. [Key Learnings](#22-key-learnings)
23. [Resource Naming Convention](#23-resource-naming-convention)

---

## 1. Project Overview

This project demonstrates a complete production-grade 3-tier architecture on AWS:

- **Tier 1 (Presentation)**: Application Load Balancer distributes incoming traffic
- **Tier 2 (Application)**: EC2 instances running Apache Tomcat with a Java student registration application
- **Tier 3 (Database)**: Amazon RDS running MariaDB in a private subnet

Additional features:
- Auto Scaling Group for fault tolerance and high availability
- Golden AMI for consistent EC2 launches
- Prometheus and Grafana for real-time monitoring
- Multi-AZ deployment for high availability

---

## 2. Architecture Diagram

```
Internet
    ↓
Application Load Balancer (Public)
    ↓
EC2 app-1 (us-east-1a)    EC2 app-2 (us-east-1b)
Tomcat + student.war       Tomcat + student.war
    ↓                           ↓
            RDS MariaDB
          (Private Subnet)
              ↓
    Prometheus + Grafana
    (Monitoring EC2)
```

---

## 3. Prerequisites

- AWS Account with appropriate permissions
- GitHub account
- SSH key pair
- Basic knowledge of Linux commands

**Tools Used:**
- AWS Console
- AWS CLI (CloudShell)
- SSH terminal
- Git
- vi/nano text editors

---

## 4. VPC and Networking Setup

### 4.1 Create VPC

**Why /16 and not /24?**
- /24 gives only 256 IPs — too small for multiple subnets
- /16 gives 65,536 IPs — enough for all subnets

```
AWS Console → VPC → Your VPCs → Create VPC

Settings:
Name            : tomcat-aws-3tier-vpc
IPv4 CIDR       : 10.0.0.0/16
IPv6            : No
Tenancy         : Default
```

**What AWS automatically creates with VPC:**
1. Default Route Table
2. Default Security Group
3. Default Network ACL

### 4.2 Create Subnets

**Why 4 subnets?**
- 2 public subnets (different AZs) for EC2 instances
- 2 private subnets (different AZs) for RDS
- RDS requires minimum 2 subnets in 2 different AZs

**Public Subnet 1:**
```
Name            : tomcat-aws-3tier-subnet-public
VPC             : tomcat-aws-3tier-vpc
AZ              : us-east-1a
CIDR            : 10.0.1.0/24
```

**Public Subnet 2:**
```
Name            : tomcat-aws-3tier-subnet-public-2
VPC             : tomcat-aws-3tier-vpc
AZ              : us-east-1b
CIDR            : 10.0.3.0/24
```

**Private Subnet 1:**
```
Name            : tomcat-aws-3tier-subnet-private
VPC             : tomcat-aws-3tier-vpc
AZ              : us-east-1a
CIDR            : 10.0.2.0/24
```

**Private Subnet 2:**
```
Name            : tomcat-aws-3tier-subnet-private-2
VPC             : tomcat-aws-3tier-vpc
AZ              : us-east-1c
CIDR            : 10.0.4.0/24
```

**Key concept:**
> A subnet is made public or private by its route table — not by the subnet itself.
> Public subnet route table points to IGW
> Private subnet route table points to NAT Gateway

### 4.3 Create Internet Gateway

**Purpose:** Allows public subnets to communicate with the internet

```
VPC → Internet Gateways → Create Internet Gateway

Name    : tomcat-aws-3tier-igw

After creation:
Actions → Attach to VPC → select tomcat-aws-3tier-vpc
```

### 4.4 Create NAT Gateway

**Purpose:** Allows private subnets to access internet outbound only (for updates/patches). No inbound traffic allowed.

**Important:** NAT Gateway sits in PUBLIC subnet but private subnet route table points to it.

```
VPC → NAT Gateways → Create NAT Gateway

Name                : tomcat-aws-3tier-nat
Subnet              : tomcat-aws-3tier-subnet-public
Connectivity type   : Public
Elastic IP          : Automatic
```

### 4.5 Create Route Tables

**Public Route Table:**
```
VPC → Route Tables → Create Route Table

Name    : tomcat-aws-3tier-rt-public
VPC     : tomcat-aws-3tier-vpc

After creation → Edit routes → Add route:
Destination : 0.0.0.0/0
Target      : tomcat-aws-3tier-igw

Subnet Associations → Edit:
Add both public subnets
```

**Private Route Table:**
```
Name    : tomcat-aws-3tier-rt-private
VPC     : tomcat-aws-3tier-vpc

After creation → Edit routes → Add route:
Destination : 0.0.0.0/0
Target      : tomcat-aws-3tier-nat

Subnet Associations → Edit:
Add both private subnets
```

---

## 5. Security Groups

### 5.1 EC2 Security Group (tomcat-aws-3tier-SG)

**Inbound Rules:**
| Type | Protocol | Port | Source | Purpose |
|------|----------|------|--------|---------|
| SSH | TCP | 22 | 0.0.0.0/0 | SSH access |
| HTTP | TCP | 80 | 0.0.0.0/0 | HTTP traffic |
| HTTPS | TCP | 443 | 0.0.0.0/0 | HTTPS traffic |
| Custom TCP | TCP | 8080 | 0.0.0.0/0 | Tomcat |
| Custom TCP | TCP | 9100 | 0.0.0.0/0 | node_exporter |

### 5.2 RDS Security Group (tomcat-aws-3tier-sg-db)

**Inbound Rules:**
| Type | Protocol | Port | Source | Purpose |
|------|----------|------|--------|---------|
| MySQL/Aurora | TCP | 3306 | EC2 Security Group | DB access from EC2 |

### 5.3 Monitoring Security Group

**Inbound Rules:**
| Type | Protocol | Port | Source | Purpose |
|------|----------|------|--------|---------|
| SSH | TCP | 22 | 0.0.0.0/0 | SSH access |
| Custom TCP | TCP | 9090 | 0.0.0.0/0 | Prometheus |
| Custom TCP | TCP | 3000 | 0.0.0.0/0 | Grafana |

---

## 6. EC2 Instance Setup

### 6.1 Launch EC2 Instances

**app-1 Settings:**
```
Name            : tomcat-aws-3tier-app-1
AMI             : Ubuntu Server 26.04 LTS
Instance type   : t3.micro
Key pair        : tomcat-aws-3tier
VPC             : tomcat-aws-3tier-vpc
Subnet          : tomcat-aws-3tier-subnet-public (us-east-1a)
Auto-assign IP  : Enable
Security Group  : tomcat-aws-3tier-SG
Storage         : 8 GiB gp3
```

**app-2 Settings:**
```
Name            : tomcat-aws-3tier-app-2
AMI             : Ubuntu Server 26.04 LTS
Instance type   : t3.micro
Key pair        : tomcat-aws-3tier
VPC             : tomcat-aws-3tier-vpc
Subnet          : tomcat-aws-3tier-subnet-public-2 (us-east-1b)
Auto-assign IP  : Enable
Security Group  : tomcat-aws-3tier-SG
Storage         : 8 GiB gp3
```

**Why different AZs?**
> If us-east-1a goes down → app-1 goes down but app-2 in us-east-1b stays up.
> Load Balancer automatically routes traffic to healthy instance.

**Important lesson learned:**
> AZ is determined by subnet — not directly selectable during EC2 launch.
> To launch in different AZ → create subnet in that AZ first.

### 6.2 SSH into app-1

```bash
ssh -i tomcat-aws-3tier.pem ubuntu@APP1-PUBLIC-IP
```

### 6.3 Initial Setup Commands

```bash
# Switch to root
sudo su

# Update package list and upgrade
apt update -y && apt upgrade -y

# Install Java (compatible with Tomcat 9)
apt install default-jdk -y

# Verify Java installation
java -version

# Install MariaDB client (NOT server - RDS is our server)
apt install mariadb-client -y
```

**Why default-jdk and not a specific version?**
> Tomcat 9 works with Java 8, 11, and 17.
> default-jdk installs the recommended version for the OS.

### 6.4 Create Project Directory and Download Tomcat

```bash
# Create project directory
mkdir Tomcat-aws-3Tier
cd Tomcat-aws-3Tier

# Download Tomcat 9
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.118/bin/apache-tomcat-9.0.118.tar.gz

# Extract
tar -xvzf apache-tomcat-9.0.118.tar.gz

# List files to verify
ls
```

### 6.5 Pull Application Files from GitHub

```bash
# Initialize git
git init

# Add remote repository
git remote add origin https://github.com/AbhishekSingh070693/Hypertech-Project1.git

# Fetch from master branch
git fetch origin master

# Pull specific files only
git checkout origin/master -- student.war mysql-connector.jar student.sql
```

**Why pull specific files instead of cloning?**
> We only need 3 files from the repository.
> Cloning would download everything unnecessarily.

**What each file does:**
- `student.war` — The Java web application (compiled and packaged)
- `mysql-connector.jar` — JDBC driver that allows Java to talk to MySQL/MariaDB
- `student.sql` — SQL file to create the students table in database

---

## 7. Application Configuration

### 7.1 Deploy Files to Tomcat

```bash
# Copy WAR file to Tomcat webapps directory
cp student.war apache-tomcat-9.0.118/webapps/

# Copy JDBC driver to Tomcat lib directory
cp mysql-connector.jar apache-tomcat-9.0.118/lib/

# Verify both files are in place
ls apache-tomcat-9.0.118/webapps/
ls apache-tomcat-9.0.118/lib/ | grep mysql
```

### 7.2 Configure context.xml

**Purpose:** Tells Tomcat how to connect to the database

```bash
vi apache-tomcat-9.0.118/conf/context.xml
```

**Add this inside the `<Context>` tag (around line 21):**

```xml
<Resource name="jdbc/TestDB" auth="Container"
    type="javax.sql.DataSource"
    maxTotal="100" maxIdle="30" maxWaitMillis="10000"
    username="admin"
    password="master123"
    driverClassName="com.mysql.jdbc.Driver"
    url="jdbc:mysql://RDS-ENDPOINT:3306/studentapp"/>
```

**Replace RDS-ENDPOINT with your actual RDS endpoint.**

**What each parameter means:**
- `name` — JNDI name the application uses to look up the connection
- `username/password` — RDS credentials
- `driverClassName` — JDBC driver class
- `url` — Connection string: jdbc:mysql://HOST:PORT/DATABASE
- `maxTotal` — Maximum total connections in pool
- `maxIdle` — Maximum idle connections
- `maxWaitMillis` — Maximum wait time for connection

**Key lesson:**
> The database name in the URL must match exactly what was specified during RDS creation.
> In this project: studentapp

---

## 8. RDS Database Setup

### 8.1 Create DB Subnet Group

**Why needed?**
> RDS requires a DB Subnet Group to know which subnets it can use.
> The VPC dropdown in RDS creation only shows VPCs that have a DB Subnet Group.

```
RDS Console → Subnet Groups → Create DB Subnet Group

Name        : tomcat-aws-3tier-db-subnet-group
Description : DB subnet group for tomcat-aws-3tier
VPC         : tomcat-aws-3tier-vpc

Add subnets:
AZ1: us-east-1a → tomcat-aws-3tier-subnet-private (10.0.2.0/24)
AZ2: us-east-1c → tomcat-aws-3tier-subnet-private-2 (10.0.4.0/24)
```

**Challenge faced:**
> RDS requires minimum 2 subnets in 2 different AZs.
> Initially only had 1 private subnet — had to create a second one.

### 8.2 Create RDS Instance via AWS CLI

**Why CLI instead of Console?**
> AWS Console had a bug where the VPC dropdown was empty.
> CLI worked perfectly without UI issues.

```bash
aws rds create-db-instance \
  --db-instance-identifier tomcat-aws-3tier-rds \
  --db-instance-class db.t3.micro \
  --engine mariadb \
  --engine-version 10.6 \
  --master-username admin \
  --master-user-password master123 \
  --db-name studentapp \
  --db-subnet-group-name tomcat-aws-3tier-db-subnet-group \
  --no-publicly-accessible \
  --allocated-storage 20 \
  --region us-east-1
```

**RDS Details:**
```
Endpoint : tomcat-aws-3tier-rds.czlrvse8onkj.us-east-1.rds.amazonaws.com
Port     : 3306
Database : studentapp
Username : admin
Password : master123
```

**Why Public Access = No?**
> RDS is in private subnet — no direct internet access.
> EC2 connects to RDS via private IP within VPC.
> More secure — database not exposed to internet.

**Why MariaDB?**
> MariaDB is a fork of MySQL — nearly identical commands.
> AWS manages the server — we just use it.
> Cost effective for this project.

---

## 9. Database Import

### 9.1 Connect to RDS from EC2

```bash
# Connect to RDS (from EC2 app-1)
mysql -h tomcat-aws-3tier-rds.czlrvse8onkj.us-east-1.rds.amazonaws.com \
-u admin -pmaster123
```

**If connection times out:**
> Check RDS Security Group inbound rules.
> Port 3306 must be open from EC2 Security Group.

### 9.2 Import Database Schema

```sql
-- Select the database
use studentapp;

-- Import the SQL file
source /home/ubuntu/Tomcat-aws-3Tier/student.sql;

-- Verify table created
show tables;

-- Check table structure
describe students;

-- Exit
exit
```

**Expected output after describe students:**
```
+--------------------+--------------+------+-----+---------+----------------+
| Field              | Type         | Null | Key | Default | Extra          |
+--------------------+--------------+------+-----+---------+----------------+
| student_id         | int(11)      | NO   | PRI | NULL    | auto_increment |
| student_name       | varchar(100) | NO   |     | NULL    |                |
| student_addr       | varchar(100) | NO   |     | NULL    |                |
| student_age        | varchar(3)   | NO   |     | NULL    |                |
| student_qual       | varchar(20)  | NO   |     | NULL    |                |
| student_percent    | varchar(10)  | NO   |     | NULL    |                |
| student_year_passed| varchar(10)  | NO   |     | NULL    |                |
+--------------------+--------------+------+-----+---------+----------------+
```

---

## 10. Tomcat systemd Service

### 10.1 Why systemd?

> Without systemd — Tomcat must be started manually after every reboot.
> Auto Scaling creates new EC2s at any time — no one available to start Tomcat manually.
> systemd ensures Tomcat starts automatically on every boot.

### 10.2 Create Service File

```bash
vi /etc/systemd/system/tomcat.service
```

**Content:**
```ini
[Unit]
Description=Tomcat 9 Service
After=network.target

[Service]
Type=forking
User=root
ExecStart=/home/ubuntu/Tomcat-aws-3Tier/apache-tomcat-9.0.118/bin/startup.sh
ExecStop=/home/ubuntu/Tomcat-aws-3Tier/apache-tomcat-9.0.118/bin/shutdown.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**What each section means:**
- `[Unit]` — Description and dependencies
- `After=network.target` — Start only after network is ready
- `[Service]` — How to run the service
- `Type=forking` — Process forks after startup
- `ExecStart` — Command to start Tomcat
- `ExecStop` — Command to stop Tomcat
- `Restart=on-failure` — Auto restart if crashes
- `[Install]` — When to start during boot
- `WantedBy=multi-user.target` — Start in normal multi-user mode

### 10.3 Enable and Start Service

```bash
# Reload systemd daemon
systemctl daemon-reload

# Enable service (start on boot)
systemctl enable tomcat

# Start service
systemctl start tomcat

# Check status
systemctl status tomcat
```

**Expected output:**
```
● tomcat.service - Tomcat 9 Service
   Active: active (running)
   Enabled: enabled
```

### 10.4 Verify Auto-start

```bash
# Reboot the machine
reboot

# After reboot - SSH back in
ssh -i tomcat-aws-3tier.pem ubuntu@APP1-PUBLIC-IP

# Check if Tomcat started automatically
sudo su
systemctl status tomcat
```

---

## 11. AMI Creation

### 11.1 What is a Golden AMI?

> AMI (Amazon Machine Image) is a snapshot of the EC2 instance's hard drive.
> Golden AMI = pre-configured AMI with all software and settings ready.
> New EC2s launched from AMI have everything pre-installed.

**What AMI captures:**
- Operating System
- All installed software (Java, Tomcat, MariaDB client)
- All files and directories
- All configurations (context.xml, tomcat.service)
- student.war in webapps/
- mysql-connector.jar in lib/

**What AMI does NOT capture:**
- Instance type
- Security Group
- Key pair
- VPC/Subnet
- Public IP

### 11.2 Create AMI

```
EC2 Console → Instances
→ Select tomcat-aws-3tier-app-1
→ Actions → Image and templates → Create image

Image name  : tomcat-aws-3tier-ami
Description : Golden AMI with Tomcat Java and student app
No reboot   : ✅ Enable (keeps instance running during AMI creation)
```

**Wait for AMI status to show "Available" before proceeding.**

---

## 12. Launch Template

### 12.1 What is a Launch Template?

> Launch Template stores EC2 launch configuration.
> Auto Scaling uses it to know HOW to launch new instances.
> Contains hardware and network settings — NOT subnet or VPC (those are in ASG).

**Difference between AMI and Launch Template:**
```
AMI             = WHAT is installed (software, configs)
Launch Template = HOW to launch (instance type, SG, key pair)
```

### 12.2 Create Launch Template

```
EC2 Console → Launch Templates → Create Launch Template

Name            : tomcat-aws-3tier-lt
Description     : Launch template for tomcat 3tier project
AMI             : tomcat-aws-3tier-ami
Instance type   : t3.micro
Key pair        : tomcat-aws-3tier
Security Group  : tomcat-aws-3tier-SG
VPC             : ❌ leave blank
Subnet          : ❌ leave blank

Advanced network configuration:
→ Add network interface
→ Auto-assign public IP: Enable
```

**Why leave VPC and Subnet blank?**
> Auto Scaling Group decides WHERE to launch.
> Launch Template decides WHAT to launch.

---

## 13. Target Group

### 13.1 What is a Target Group?

> Target Group is a collection of EC2 instances that Load Balancer routes traffic to.
> Health checks determine which instances receive traffic.

### 13.2 Create Target Group

```
EC2 Console → Target Groups → Create Target Group

Target type     : Instances
Name            : tomcat-aws-3tier-tg
Protocol        : HTTP
Port            : 8080
VPC             : tomcat-aws-3tier-vpc
Health check path: /student

Advanced health check settings:
Healthy threshold   : 5
Unhealthy threshold : 2
Timeout             : 5 seconds
Interval            : 30 seconds
Success codes       : 200,302
```

**Challenge faced:**
> Health checks were failing even though application was working.
> Root cause: Tomcat returns HTTP 302 (redirect) for /student → /student/
> Health check only accepted 200 — not 302.
> Fix: Changed success codes to 200,302

**Why /student as health check path?**
> Using / checks only Tomcat — not our application.
> If application crashes, Tomcat default page still responds with 200.
> Using /student checks if our actual application is working.

### 13.3 Register EC2 Instances

```
Target Groups → tomcat-aws-3tier-tg → Targets
→ Register targets
→ Select app-1 and app-2
→ Port: 8080
→ Register pending targets
```

---

## 14. Application Load Balancer

### 14.1 What is ALB?

> ALB distributes incoming traffic across multiple EC2 instances.
> Uses Round Robin by default.
> Performs health checks every 30 seconds.
> Stops sending traffic to unhealthy instances.

**Port 80 vs Port 8080:**
```
User → port 80 → Load Balancer → port 8080 → EC2 (Tomcat)
```
- Users access via standard HTTP port 80
- Tomcat runs on 8080 (doesn't require root privileges)
- ALB handles the port translation

### 14.2 Create Load Balancer

```
EC2 Console → Load Balancers → Create Load Balancer
→ Application Load Balancer → Create

Name            : tomcat-aws-3tier-alb
Scheme          : Internet-facing
IP type         : IPv4

Network mapping:
VPC             : tomcat-aws-3tier-vpc
AZs             :
  us-east-1a    → tomcat-aws-3tier-subnet-public
  us-east-1b    → tomcat-aws-3tier-subnet-public-2

Security Group  : tomcat-aws-3tier-SG

Listeners:
Protocol        : HTTP
Port            : 80
Forward to      : tomcat-aws-3tier-tg
```

**Test ALB:**
```
http://tomcat-aws-3tier-alb-XXXXXX.us-east-1.elb.amazonaws.com/student
```

---

## 15. Auto Scaling Group

### 15.1 What is Auto Scaling?

> Auto Scaling automatically adjusts number of EC2 instances.
> Two types of triggers:
> 1. Health failure → replace unhealthy instance
> 2. CPU/Load based → add more instances when busy

**Desired vs Minimum vs Maximum:**
```
Desired (2) = Normal state — always maintain 2 instances
Minimum (2) = Never go below 2 — ensures minimum availability
Maximum (4) = Never go above 4 — controls cost
```

### 15.2 Create Auto Scaling Group

```
EC2 Console → Auto Scaling Groups → Create

Name            : tomcat-aws-3tier-asg
Launch Template : tomcat-aws-3tier-lt

VPC             : tomcat-aws-3tier-vpc
Subnets         :
  tomcat-aws-3tier-subnet-public
  tomcat-aws-3tier-subnet-public-2

Load Balancing  : Attach to existing Load Balancer
Target Group    : tomcat-aws-3tier-tg
Health checks   : Turn on ELB health checks ✅

Desired capacity : 2
Minimum capacity : 2
Maximum capacity : 4

Scaling Policy:
Type            : Target tracking
Policy name     : tomcat-aws-3tier-cpu-policy
Metric          : Average CPU utilization
Target value    : 50%
```

**How Auto Scaling and Load Balancer work together:**
```
EC2 becomes unhealthy
    ↓
Load Balancer stops sending traffic (doctor - diagnoses)
    ↓
Auto Scaling detects unhealthy instance (hospital - acts)
    ↓
Terminates unhealthy EC2
    ↓
Launches new EC2 from Launch Template (using Golden AMI)
    ↓
Registers new EC2 with Target Group automatically
    ↓
Load Balancer starts sending traffic to new EC2
```

---

## 16. Prometheus Installation

### 16.1 What is Prometheus?

> Prometheus is a monitoring and alerting toolkit.
> Scrapes metrics from exporters at regular intervals.
> Stores metrics as time series data.
> Cannot directly read metrics from EC2 or RDS — needs exporters.

### 16.2 Why Separate Monitoring EC2?

> Auto Scaling can terminate app servers anytime — monitoring data would be lost.
> Prometheus on app server would compete for resources.
> Separate server has one dedicated job — monitoring.

### 16.3 Create Monitoring EC2

```
Name            : tomcat-aws-3tier-monitoring
AMI             : Ubuntu 26.04
Instance type   : t3.micro
VPC             : tomcat-aws-3tier-vpc
Subnet          : tomcat-aws-3tier-subnet-public
Security Group  : open ports 22, 9090, 3000
Key pair        : tomcat-aws-3tier
```

### 16.4 Install Prometheus

```bash
sudo su
apt update -y
apt install prometheus -y
systemctl start prometheus
systemctl enable prometheus
systemctl status prometheus
```

**Verify:**
```
http://MONITORING-EC2-IP:9090
```

---

## 17. Grafana Installation

### 17.1 What is Grafana?

> Grafana is a visualization tool for metrics.
> Connects to data sources like Prometheus.
> Creates beautiful dashboards and graphs.
> Cannot collect data itself — needs Prometheus as data source.

**Prometheus vs Grafana:**
```
Prometheus = Excel sheet storing all data
Grafana    = Charts and graphs made from that data
```

### 17.2 Install Grafana

```bash
# Install prerequisites
apt install -y apt-transport-https wget gnupg

# Import GPG key
mkdir -p /etc/apt/keyrings
wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
chmod 644 /etc/apt/keyrings/grafana.asc

# Add repository
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | \
tee -a /etc/apt/sources.list.d/grafana.list

# Install
apt update -y
apt install grafana -y

# Start and enable
systemctl start grafana-server
systemctl enable grafana-server
systemctl status grafana-server
```

**Access Grafana:**
```
http://MONITORING-EC2-IP:3000
Username: admin
Password: admin (change on first login)
```

---

## 18. node_exporter Installation

### 18.1 What is node_exporter?

> node_exporter is a bridge between EC2 and Prometheus.
> Collects OS metrics: CPU, RAM, disk, network.
> Exposes them on port 9100 in Prometheus format.
> Must be installed on EVERY EC2 you want to monitor.

**Exporter reference:**
```
node_exporter   → Linux OS metrics        → port 9100
mysqld_exporter → MySQL/MariaDB metrics   → port 9104
jmx_exporter    → Java/Tomcat metrics     → port 9180
```

### 18.2 Install on Each EC2

```bash
# Run on app-1 and app-2
apt install prometheus-node-exporter -y
systemctl enable prometheus-node-exporter
systemctl status prometheus-node-exporter
```

**Note:** apt installation automatically starts node_exporter.
If you see "address already in use" error on port 9100 — node_exporter is already running!

**Verify:**
```
http://EC2-PUBLIC-IP:9100/metrics
```

**Important:** Open port 9100 in EC2 Security Group!

---

## 19. Prometheus Configuration

### 19.1 Edit prometheus.yml

```bash
# On monitoring EC2
# Download fresh default config
cd /etc/prometheus
wget https://raw.githubusercontent.com/prometheus/prometheus/main/documentation/examples/prometheus.yml -O prometheus.yml

# Add targets using cat append
cat >> /etc/prometheus/prometheus.yml << 'EOF'

  - job_name: 'node_exporter_app1'
    static_configs:
      - targets: ['APP1-PRIVATE-IP:9100']

  - job_name: 'node_exporter_app2'
    static_configs:
      - targets: ['APP2-PRIVATE-IP:9100']
EOF
```

**Complete prometheus.yml:**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: 'node_exporter_app1'
    static_configs:
      - targets: ['10.0.1.99:9100']

  - job_name: 'node_exporter_app2'
    static_configs:
      - targets: ['10.0.2.247:9100']
```

### 19.2 Validate and Restart

```bash
# Validate config
promtool check config /etc/prometheus/prometheus.yml

# Expected output:
# SUCCESS: prometheus.yml is valid

# Restart Prometheus
systemctl restart prometheus

# Check targets in browser:
# http://MONITORING-EC2-IP:9090/targets
```

**Challenge faced:**
> YAML is very strict about indentation.
> Adding a second scrape_configs block caused INVALIDARGUMENT error.
> Fix: Use cat >> to append to existing scrape_configs block.
> Use promtool to validate before restarting.

**Why use private IPs for targets?**
> Monitoring EC2 and app servers are in same VPC.
> Private IPs work within VPC — faster and more reliable.
> Public IPs can change — private IPs stay consistent.

---

## 20. Grafana Dashboard Setup

### 20.1 Add Prometheus Data Source

```
Grafana → Connections → Data sources → Add data source
→ Select Prometheus

URL: http://localhost:9090
→ Save & Test
→ Should show: "Data source is working"
```

**Why localhost:9090?**
> Grafana and Prometheus are on the SAME EC2.
> localhost = this machine itself.
> No need to go through internet — direct internal connection.

### 20.2 Import Node Exporter Dashboard

```
Grafana → Dashboards → New → Import

Dashboard ID: 1860
Click Load
Select Prometheus data source
Click Import
```

**Dashboard 1860 shows:**
- CPU usage per instance
- RAM usage
- Disk I/O
- Network traffic
- System load
- All in real time!

### 20.3 Select Correct Instance

In dashboard dropdowns at top:
```
Job      : node_exporter_app1 or node_exporter_app2
Instance : 10.0.1.99:9100 or 10.0.2.247:9100
```

---

## 21. Challenges Faced and Solutions

### Challenge 1: VPC CIDR too small
**Problem:** Created VPC with /24 — couldn't create second subnet.
**Solution:** Add secondary CIDR block (10.1.0.0/16) to existing VPC or recreate with /16.
**Lesson:** Always use /16 for VPC CIDR.

### Challenge 2: RDS requires 2 subnets in 2 AZs
**Problem:** Only had 1 private subnet — RDS subnet group creation failed.
**Solution:** Created second private subnet in different AZ.
**Lesson:** Always create 2 subnets per tier for high availability.

### Challenge 3: RDS VPC dropdown empty in console
**Problem:** VPC not showing in RDS creation dropdown.
**Root cause:** DB Subnet Group must exist before VPC appears.
**Solution:** Created DB Subnet Group first — then VPC appeared.
**Alternative:** Used AWS CLI to create RDS directly.

### Challenge 4: Health checks failing (502 Bad Gateway)
**Problem:** Load Balancer showing 502 error.
**Root cause:** Tomcat returns HTTP 302 redirect for /student.
**Solution:** Changed health check success codes to 200,302.
**Lesson:** Always check actual HTTP response code before setting health check codes.

### Challenge 5: node_exporter not on Auto Scaled instances
**Problem:** New instances launched by Auto Scaling didn't have node_exporter.
**Root cause:** AMI was created before node_exporter was installed.
**Solution:** Install node_exporter → create new AMI → update Launch Template.
**Lesson:** Always install ALL required software before creating Golden AMI.

### Challenge 6: prometheus.yml YAML syntax error
**Problem:** Prometheus failing to start after editing config file.
**Root cause:** Added duplicate scrape_configs section.
**Solution:** Downloaded fresh default config and appended targets using cat >>.
**Lesson:** YAML is strict about indentation and duplicate keys. Always validate with promtool.

### Challenge 7: EC2 instances in same AZ
**Problem:** Both EC2s were in us-east-1a — single point of failure.
**Solution:** Created new subnet in us-east-1b, relaunched app-2 in it.
**Lesson:** Always deploy across multiple AZs for high availability.

### Challenge 8: Tomcat not auto-starting
**Problem:** Tomcat not running on new Auto Scaled instances.
**Solution:** Created systemd service file — Tomcat now starts automatically on boot.
**Lesson:** Any service that needs to run continuously must have a systemd service.

---

## 22. Key Learnings

### AWS Networking
```
1. VPC = your private network in AWS
2. /16 CIDR for VPC — always
3. Subnets carved from VPC CIDR
4. Route table makes subnet public or private (not the subnet itself)
5. IGW = door to internet for public subnets
6. NAT GW = one-way door for private subnets (outbound only)
7. Security Group = stateful firewall at instance level
8. NACL = stateless firewall at subnet level
```

### High Availability
```
1. Always deploy across 2+ AZs
2. AZ = data center — if one fails, others continue
3. Load Balancer distributes traffic across AZs
4. Auto Scaling replaces failed instances automatically
5. RDS managed by AWS — automatic backups and failover
```

### DevOps Best Practices
```
1. Golden AMI = pre-baked image with all configs
2. Launch Template = how to launch new instances
3. systemd = ensures services start automatically
4. Separate monitoring server = dedicated monitoring
5. Private subnet for database = security best practice
6. Health checks = detect failures early
```

### Port Reference
```
22   → SSH
80   → HTTP
443  → HTTPS
3000 → Grafana
3306 → MySQL/MariaDB
8080 → Tomcat
9090 → Prometheus
9100 → node_exporter
9104 → mysqld_exporter
```

---

## 23. Resource Naming Convention

All resources follow this naming pattern: `tomcat-aws-3tier-[resource]`

| Resource | Name |
|----------|------|
| VPC | tomcat-aws-3tier-vpc |
| Public Subnet 1 | tomcat-aws-3tier-subnet-public |
| Public Subnet 2 | tomcat-aws-3tier-subnet-public-2 |
| Private Subnet 1 | tomcat-aws-3tier-subnet-private |
| Private Subnet 2 | tomcat-aws-3tier-subnet-private-2 |
| Internet Gateway | tomcat-aws-3tier-igw |
| NAT Gateway | tomcat-aws-3tier-nat |
| Public Route Table | tomcat-aws-3tier-rt-public |
| Private Route Table | tomcat-aws-3tier-rt-private |
| EC2 Instance 1 | tomcat-aws-3tier-app-1 |
| EC2 Instance 2 | tomcat-aws-3tier-app-2 |
| Monitoring EC2 | tomcat-aws-3tier-monitoring |
| Key Pair | tomcat-aws-3tier |
| EC2 Security Group | tomcat-aws-3tier-SG |
| RDS Security Group | tomcat-aws-3tier-sg-db |
| RDS Instance | tomcat-aws-3tier-rds |
| DB Subnet Group | tomcat-aws-3tier-db-subnet-group |
| Golden AMI | tomcat-aws-3tier-ami |
| Launch Template | tomcat-aws-3tier-lt |
| Target Group | tomcat-aws-3tier-tg |
| Load Balancer | tomcat-aws-3tier-alb |
| Auto Scaling Group | tomcat-aws-3tier-asg |
| Scaling Policy | tomcat-aws-3tier-cpu-policy |

---

## Project Complete! 🎉

This project demonstrates:
- ✅ Custom VPC design from scratch
- ✅ Multi-AZ deployment
- ✅ Public/Private subnet architecture
- ✅ Secure database in private subnet
- ✅ Auto Scaling for fault tolerance
- ✅ Load Balancing for traffic distribution
- ✅ Golden AMI for consistent deployments
- ✅ Real-time monitoring with Prometheus and Grafana

**Resume line:**
> "Designed and deployed a production-grade 3-Tier AWS architecture with custom VPC, multi-AZ EC2 deployment, RDS MariaDB in private subnet, Application Load Balancer, Auto Scaling Group, and real-time monitoring using Prometheus and Grafana — validated fault tolerance by simulating instance failure and confirming automatic recovery."
