Project Overview

This project demonstrates a production-style AWS network architecture built completely from scratch. The environment consists of a centralized Hub VPC connected to multiple Spoke VPCs through AWS Transit Gateway. Application servers remain completely private while traffic enters through an internet-facing Application Load Balancer protected by Cloudflare and HTTPS certificates from ACM.

No SSH keys or bastion hosts were used. All administration was performed securely through Systems Manager Session Manager (SSM).

Architecture
                    Internet
                        │
                HTTPS (443)
                        │
                 Cloudflare Proxy
                        │
                 app.basitcloudlab.com
                        │
                 Route53 Hosted Zone
                        │
                 ACM SSL Certificate
                        │
                Application Load Balancer
                Hub VPC (10.0.0.0/16)
                        │
                ───────────────────
                        │
                  Transit Gateway
                   (Central Router)
                   /            \
                  /              \
                 /                \
      Spoke1 VPC                  Spoke2 VPC
      10.1.0.0/16                 10.2.0.0/16
            │                           │
      Private EC2                  Private EC2
      Nginx Server                 Nginx Server
Services Used
Amazon VPC
Transit Gateway
EC2
Systems Manager (SSM)
Interface VPC Endpoints
IAM Roles
NAT Gateway
Application Load Balancer
Target Groups
ACM
Route53
Cloudflare
Security Groups
Route Tables
Nginx
Network Layout
Hub VPC

CIDR:

10.0.0.0/16

Contains:

Public ALB subnet A
Public ALB subnet B
NAT Gateway subnet
TGW attachment subnet
Spoke1 VPC

CIDR:

10.1.0.0/16

Contains:

Private Application subnet
TGW subnet
SSM Interface Endpoints
Private EC2
Spoke2 VPC

CIDR:

10.2.0.0/16

Contains:

Private Application subnet
TGW subnet
SSM Interface Endpoints
Private EC2
Security Groups
Hub-ALB-SG

Inbound:

HTTP 80     0.0.0.0/0
HTTPS 443   0.0.0.0/0

Outbound:

All traffic
Spoke1-App-SG

Inbound:

TCP 80 from 10.0.0.0/24
TCP 80 from 10.0.1.0/24

Outbound:

All traffic
Spoke2-App-SG

Inbound:

TCP 80 from 10.0.0.0/24
TCP 80 from 10.0.1.0/24

Outbound:

All traffic
Transit Gateway Configuration

Created:

Central Transit Gateway

Attached:

Hub VPC
Spoke1 VPC
Spoke2 VPC
TGW Route Table

Routes:

10.0.0.0/16 → Hub Attachment
10.1.0.0/16 → Spoke1 Attachment
10.2.0.0/16 → Spoke2 Attachment
0.0.0.0/0 → Hub Attachment
Why 0.0.0.0/0 Points to Hub

Spoke VPCs have no Internet Gateway.

Internet access is centralized.

Traffic flow:

Private EC2
      ↓
Spoke Route Table
      ↓
Transit Gateway
      ↓
Hub Attachment
      ↓
Hub TGW Route Table
      ↓
NAT Gateway
      ↓
Internet

This provides:

Centralized egress
Better security
Reduced NAT costs
Simplified management
Hub Route Table

Hub-TGW-RT

10.0.0.0/16 → local
10.1.0.0/16 → TGW
10.2.0.0/16 → TGW
0.0.0.0/0 → NAT Gateway
Spoke1 Application Route Table
10.1.0.0/16 → local
0.0.0.0/0 → Transit Gateway
Spoke2 Application Route Table
10.2.0.0/16 → local
0.0.0.0/0 → Transit Gateway
Systems Manager Architecture

No SSH was used.

No bastion host was deployed.

Administration was performed using:

AWS Systems Manager Session Manager
IAM Role

Attached to both EC2 instances:

AmazonSSMManagedInstanceCore

Role:

EC2-SSM-Role
Interface Endpoints

Each spoke VPC contains:

SSM
com.amazonaws.us-east-1.ssm
SSM Messages
com.amazonaws.us-east-1.ssmmessages
EC2 Messages
com.amazonaws.us-east-1.ec2messages

Total:

6 Interface Endpoints
IMDSv2 Configuration

Initially SSM was offline.

Problem:

SSM Agent unable to acquire credentials

Solution:

Modified Metadata Options:

IMDSv2 Required

HTTP PUT response hop limit = 2

SSM immediately became online.

NAT Gateway

Deployed inside Hub Public Subnet.

Purpose:

Allow private EC2 instances to:

yum update
install nginx
download packages

Without exposing servers to the Internet.

Installing Nginx

Connected using SSM.

Update system:

sudo yum update -y

Install nginx:

sudo yum install nginx -y

Enable:

sudo systemctl enable nginx

Start:

sudo systemctl start nginx
Spoke1 Web Page
echo "Spoke1 Application Server" | sudo tee /usr/share/nginx/html/index.html
Spoke2 Web Page
echo "Spoke2 Application Server" | sudo tee /usr/share/nginx/html/index.html
Application Load Balancer

Internet Facing

Across:

AZ A
AZ B

Listener:

80

Later added:

443
Target Group

Type:

IP

Registered targets:

10.1.1.192
10.2.1.134

Health check:

HTTP /
Port 80
Target Group Troubleshooting

Initial State:

Unhealthy
Request Timed Out

Problem:

Security Group rules missing.

Solution:

Allowed:

TCP 80
Source:
10.0.0.0/24
10.0.1.0/24

Targets became healthy.

ACM Certificate

Requested:

app.basitcloudlab.com

Validation:

DNS

Certificate status:

Issued
Cloudflare Configuration

Created:

CNAME
app.basitcloudlab.com

Points to:

central-edge-alb-xxxx.us-east-1.elb.amazonaws.com

Proxy:

Enabled

SSL Mode:

Full (Strict)

Always HTTPS:

Enabled
HTTPS Traffic Flow
Browser
     │
HTTPS
     │
Cloudflare
     │
HTTPS
     │
ALB
     │
HTTP
     │
Transit Gateway
     │
Private EC2
Route53

Hosted Zone:

basitcloudlab.com

Created:

A Alias

Purpose:

ACM DNS validation and future AWS DNS integration.

Actual client traffic enters through Cloudflare.

Validation
SSM
whoami

Output:

ssm-user
OS Verification
cat /etc/os-release

Output:

Amazon Linux 2023
Verify Internet
curl google.com

Successful

Verify Nginx
systemctl status nginx

Active:

running
Verify ALB

Open:

http://ALB-DNS

Refresh repeatedly.

Responses alternate:

Spoke1 Application Server

and

Spoke2 Application Server
Verify HTTPS

Open:

https://app.basitcloudlab.com

Successful.

Troubleshooting
SSM Offline

Cause:

Missing credentials.

Fix:

IAM Role
+
SSM Interface Endpoints
+
IMDSv2 hop limit = 2
yum update Hanging

Cause:

No Internet access.

Fix:

Centralized NAT Gateway

and

0.0.0.0/0
→ Transit Gateway
→ Hub
→ NAT Gateway
Target Group Unhealthy

Cause:

Security Groups blocked port 80.

Fix:

Added HTTP inbound from Hub ALB subnets.

ACM Certificate Pending

Cause:

DNS validation record missing.

Fix:

Added ACM CNAME inside Cloudflare.

Cannot Delete ENI

Cause:

Transit Gateway attachment dependency.

Fix:

Delete:

TGW Attachment
before
VPC
Skills Demonstrated
VPC Design
Transit Gateway
Hub-and-Spoke Architecture
Centralized Internet Egress
NAT Gateway
Route Tables
Security Groups
IAM
SSM Session Manager
Interface Endpoints
Application Load Balancer
Target Groups
Health Checks
ACM
Route53
Cloudflare
HTTPS
DNS
Nginx
High Availability
Multi-VPC Networking
Troubleshooting
Resume Description

Designed and implemented a production-style AWS hub-and-spoke architecture using Transit Gateway, centralized NAT Gateway egress, private EC2 instances, Systems Manager Session Manager, interface VPC endpoints, Application Load Balancer, ACM certificates, Route53, and Cloudflare. Built secure end-to-end HTTPS connectivity without public servers or SSH access and validated highly available application delivery across multiple isolated VPCs.

Future Improvements
Auto Scaling Groups
Multi-Region Deployment
Route53 Failover Routing
AWS WAF
CloudFront
Terraform Automation
CI/CD Pipeline
ECS/Fargate Migration
Kubernetes Integration

Project Type: Production-Style AWS Networking Architecture
Difficulty: Advanced
Built From Scratch: Yes
Public EC2 Instances: None
SSH Used: No
Administration Method: AWS Systems Manager Session Manager
Load Balancing: Application Load Balancer
SSL: ACM + Cloudflare Full Strict HTTPS
High Availability: Multi-AZ ALB + Multi-VPC Architecture
