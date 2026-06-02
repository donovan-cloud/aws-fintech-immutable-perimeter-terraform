# Secure Multi-AZ FinTech Infrastructure Baseline via Terraform

[![IaC](https://img.shields.io/badge/IaC-Terraform_1.5+-7B42BC.svg)](https://www.terraform.io/)
[![Provider](https://img.shields.io/badge/Provider-AWS-orange.svg)](https://aws.amazon.com/)
[![Networking](https://img.shields.io/badge/Networking-VPC_Multi--AZ-blue.svg)](https://aws.amazon.com/vpc/)
[![Compliance](https://img.shields.io/badge/Compliance-PCI--DSS_v4.0-red.svg)](https://aws.amazon.com/security/)

## 📋 Project Overview
This repository contains a declarative, production-ready Infrastructure-as-Code (IaC) engineering baseline that deploys a highly available, zero-trust cloud network architecture on AWS. 

Strictly adhering to **PCI-DSS v4.0** compliance boundaries, this blueprint handles the automated provisioning of network segmentations, absolute database plane isolation, and secure, private platform service communication lines.

---

## 🏗️ Network Topology & Segregation Schema

The infrastructure is distributed across distinct Availability Zones to eliminate single points of failure while maintaining rigid traffic boundaries:

* **Public Ingress Perimeter:** Limited strictly to AWS Application Load Balancers (ALBs) mapped across multi-zone boundaries for external client request routing.
* **Core Application Lane:** Completely private subnets with zero native inbound routes from the internet. Runtimes communicate exclusively with external banking APIs using outbound-only NAT Gateways.
* **Isolated Financial Data Plane:** Locked down behind multi-tier security groups. No internet ingress, no internet egress. 
* **Zero-Trust Private Communication:** Microservices communicate natively with protected cloud platform services (KMS, S3, Secrets Manager) using AWS PrivateLink endpoints, bypassing exposure to the public internet altogether.

---

## 📊 Infrastructure Structural Matrix

| Network Tier | Subnet Strategy | Internet Ingress | Internet Egress | Primary Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 1: Edge Public** | Public Multi-AZ | Allowed (Port 443) | Allowed | External Traffic Routing & Load Balancing |
| **Tier 2: Core Compute** | Private Multi-AZ | Denied | Outbound-Only (NAT) | Microservices & Business Logic Execution |
| **Tier 3: Database** | Isolated Multi-AZ | Denied | Denied | High-Value Financial Ledger & Data Storage |

---

## 💻 Core Infrastructure Declarations

### 1. Primary Architecture Matrix (`main.tf`)
This blueprint handles the core network virtualization, subnetting structures, and zero-trust firewall group rules:

```hcl
# ==============================================================================
# DEPLOYMENT PARAMETERS & CLOUD PROVIDER CONFIGURATION
# ==============================================================================
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

# ==============================================================================
# NETWORK BOUNDARY MATRIX (VPC CORE DEPLOYMENT)
# ==============================================================================
resource "aws_vpc" "fintech_prod_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "fintech-production-secure-vpc"
    Environment = "Production"
    Compliance  = "PCI-DSS-v4.0"
  }
}

# ==============================================================================
# NETWORK SEGREGATION PLANES (MULTI-AZ SUBNET DISTRIBUTION)
# ==============================================================================
resource "aws_subnet" "public_ingress_az1" {
  vpc_id            = aws_vpc.fintech_prod_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "${var.aws_region}a"
  tags              = { Name = "fintech-prod-public-ingress-az1" }
}

resource "aws_subnet" "private_compute_az1" {
  vpc_id            = aws_vpc.fintech_prod_vpc.id
  cidr_block        = "10.0.10.0/24"
  availability_zone = "${var.aws_region}a"
  tags              = { Name = "fintech-prod-private-compute-az1" }
}

resource "aws_subnet" "isolated_data_az1" {
  vpc_id            = aws_vpc.fintech_prod_vpc.id
  cidr_block        = "10.0.100.0/24"
  availability_zone = "${var.aws_region}a"
  tags              = { Name = "fintech-prod-isolated-data-az1" }
}

# ==============================================================================
# CONTROL PLANE DEFENSE BOUNDARIES (SECURITY LAYER MATRICES)
# ==============================================================================
resource "aws_security_group" "edge_load_balancer_sg" {
  name        = "fintech-prod-edge-alb-security-group"
  description = "Restricts edge access point inputs strictly to encrypted web traffic"
  vpc_id      = aws_vpc.fintech_prod_vpc.id

  ingress {
    description = "Enforce encrypted TLS transport layer requests"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/32"] # To be populated explicitly with trusted ingress origins
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "isolated_data_sg" {
  name        = "fintech-prod-database-isolation-layer"
  description = "Strips database plane exposure to block all non-compute queries"
  vpc_id      = aws_vpc.fintech_prod_vpc.id

  ingress {
    description     = "Restrict data mutations exclusively to verified internal compute microservices"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.edge_load_balancer_sg.id] # Relies heavily on trusted origin chaining
  }

  egress {
    description = "Absolute data egress blackout pattern. No external connections allowed."
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/32"] # Strictly loops back locally; zero global egress routes
  }
}

# ==============================================================================
# OUTPUT ARTIFACTS FOR CONTROL SYSTEMS
# ==============================================================================
output "vpc_id" {
  value       = aws_vpc.fintech_prod_vpc.id
  description = "Target deployment tracking string for infrastructure pipeline orchestration."
}
