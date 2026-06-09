# Tomcat AWS 3-Tier Architecture Project

## Project Overview
A production-grade 3-tier architecture deployed on AWS with:
- Java application running on Tomcat
- MariaDB database on Amazon RDS
- Real-time monitoring with Prometheus and Grafana

## Architecture
- **Tier 1**: Application Load Balancer (Public)
- **Tier 2**: EC2 instances running Tomcat (Public Subnet)
- **Tier 3**: RDS MariaDB (Private Subnet)

## AWS Resources Created
- Custom VPC (10.0.0.0/16)
- Public Subnets: us-east-1a, us-east-1b
- Private Subnets: us-east-1a, us-east-1c
- Internet Gateway + NAT Gateway
- 2 EC2 instances (t3.micro) running Tomcat
- RDS MariaDB (db.t3.micro)
- Application Load Balancer
- Auto Scaling Group (min 2, max 4)
- Golden AMI + Launch Template

## Monitoring Stack
- Prometheus (port 9090)
- Grafana (port 3000)
- node_exporter on each EC2 (port 9100)

## Project Structure

configs/
├── context.xml      # Tomcat DB connection config
├── tomcat.service   # systemd service file
└── prometheus.yml   # Prometheus scrape config
database/
└── student.sql      # Database schema
screenshots/         # Project screenshots

## Tech Stack
- Java + Tomcat 9
- MariaDB 10.6
- AWS EC2, RDS, ALB, Auto Scaling
- Prometheus + Grafana
