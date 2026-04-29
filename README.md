OVERVIEW

This project demonstrates the creation of a fully functional custom AWS Virtual Private Cloud (VPC) from scratch, including networking components, EC2 instances, security groups, IAM roles, and internet/NAT connectivity. The goal was to build a production-style network architecture with both public and private subnets across multiple Availability Zones.

VPC Name : ez-custom-vpc
CIDR Block : 10.0.0.0/24
DNS Resolution : Enabled
DHCP Option Set : Enabled

ARCHITECTURE

The VPC is divided into four subnets across two Availability Zones (1a and 1c), separating public-facing and private resources:

Subnet CIDR AZ Type

Public-1a 10.0.0.0/26 1a Public
Public-1c 10.0.0.64/26 1c Public
Private-1a 10.0.0.128/26 1a Private
Private-1c 10.0.0.192/26 1c Private

Key Components:

Internet Gateway : E-IGW-Custom (attached to ez-custom-vpc)
NAT Gateway : Deployed into a public subnet (Zonal), with Elastic IP
Route Tables : One per subnet (public_rt_1a, public_rt_1c, private_rt_1a, private_rt_1c)
EC2 Instances : 1 public-facing, 1 private (Amazon Linux AMI)
Security Group : Custom group for testing inbound/outbound rules
IAM Instance Profile : Role with SSM (Session Manager) permissions

Public subnets have auto-assign public IP enabled and a route to the Internet Gateway (0.0.0.0/0 → E-IGW-Custom), enabling direct internet access.

Private subnets route outbound traffic through the NAT Gateway, keeping instances inaccessible from the internet while allowing outbound connectivity.

DEPLOYMENT STEPS

CREATE THE VPC

Name: ez-custom-vpc
CIDR: 10.0.0.0/24
Enable DNS resolution and DHCP option set

CREATE AND ATTACH INTERNET GATEWAY

Create IGW named E-IGW-Custom
Attach to ez-custom-vpc (state changes to "attached")

CREATE SUBNETS

Create 4 subnets (see Architecture section above)
Associate each subnet with ez-custom-vpc
Enable auto-assign public IP on both public subnets

CREATE ROUTE TABLES

Create one route table per subnet (4 total)
Associate each route table with its corresponding subnet

For public subnets:
Add route 0.0.0.0/0 → E-IGW-Custom

Private subnets:
Retain local routing only at this stage

CREATE NAT GATEWAY

Create NAT Gateway in a public subnet (e.g., Public-1a)
Subnet level: choose "Zonal"
Allocate and associate an Elastic IP

UPDATE PRIVATE ROUTE TABLES

Add route to both private route tables:
0.0.0.0/0 → NAT Gateway

Private subnets now have outbound internet access without inbound exposure

LAUNCH EC2 INSTANCES

AMI: Amazon Linux

Instance 1:
Place in public subnet
Enable public IP

Instance 2:
Place in private subnet
No public IP

CONFIGURE SECURITY GROUP

Create a security group for testing
Attach to EC2 instances as needed

SET UP IAM INSTANCE PROFILE

Create IAM role with SSM (Systems Manager) permissions
Attach role to both EC2 instances as an instance profile

CONNECT AND TEST (PUBLIC INSTANCE)

Connect to the public EC2 via AWS Session Manager (no SSH key needed)

Run:
traceroute google.com

Confirm packets route through the Internet Gateway successfully

CONNECT AND TEST (PRIVATE INSTANCE)

Connect via Session Manager

Run:
curl google.com

Confirm outbound traffic routes through the NAT Gateway

CONFIGURE PRIVATE SUBNET OUTBOUND ACCESS (ALTERNATIVE)

Instead of using a NAT Gateway, configure 3 VPC Endpoints:

com.amazonaws.region.ssm
com.amazonaws.region.ssmmessages
com.amazonaws.region.ec2messages

This allows Session Manager connectivity without internet access

KEY LEARNINGS

SUBNET SIZING WITH /26
Dividing a /24 CIDR into four /26 subnets provides 64 IPs per subnet (62 usable after AWS reserves 5). This is a practical pattern for small to mid-sized environments needing AZ redundancy.

ROUTE TABLE GRANULARITY
Creating one route table per subnet (rather than sharing) gives precise control over traffic flows. Public and private subnets have fundamentally different routing needs and should not share a table.

INTERNET GATEWAY VS. NAT GATEWAY

Internet Gateway:
Bidirectional; used by public subnets for full internet access

NAT Gateway:
Outbound-only; used by private subnets to initiate outbound connections without exposing instances to inbound traffic
Must be deployed in a public subnet to reach the Internet Gateway

SESSION MANAGER OVER SSH
Using AWS Session Manager (via IAM instance profile + SSM permissions) eliminates the need to manage SSH keys or open port 22. The instance does NOT need to be in a public subnet to connect via Session Manager, as long as it can reach SSM endpoints (via NAT Gateway or VPC endpoints).

VPC ENDPOINTS AS A NAT ALTERNATIVE
For private instances that only need AWS service access (not general internet), three VPC endpoints (SSM, SSMMessages, EC2Messages) can replace a NAT Gateway — reducing cost and keeping traffic on the AWS network.

ELASTIC IP REQUIREMENT FOR NAT GATEWAY
NAT Gateways require a static Elastic IP. This IP is the source address seen by external services for all outbound traffic from private subnets.

================================================================================

Tools & Services Used:
AWS VPC, EC2, Internet Gateway, NAT Gateway, Route Tables, Security Groups, IAM, Systems Manager (Session Manager)
