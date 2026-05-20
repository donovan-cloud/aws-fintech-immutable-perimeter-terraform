# Secure Multi-AZ FinTech Infrastructure Baseline via Terraform

A declarative, production-ready Infrastructure-as-Code (IaC) engineering baseline that deploys a highly available, zero-trust cloud network architecture. This blueprint strictly adheres to PCI-DSS v4.0 and compliance boundaries by ensuring absolute isolation of critical database planes and application runtimes.

## Network Topology & Segregation Schema

* **Public Ingress Perimeter:** Limited strictly to AWS Application Load Balancers (ALBs) mapped across multi-zone boundaries for external client request routing.
* **Core Application Lane:** Completely private subnets with zero native inbound routes from the internet. Runtimes communicate exclusively with external banking APIs using outbound-only NAT Gateways.
* **Isolated Financial Data Plane:** Locked down behind rigid multi-tier security boundaries. No internet ingress, no internet egress. 
* **Zero-Trust Private Communication:** Microservices communicate natively with protected cloud platform services (KMS, S3, Secrets Manager) using AWS PrivateLink endpoints, bypassing exposure to the public internet altogether.

---

## Terraform Architecture Structural Matrix

| Network Tier | Subnet Strategy | Internet Ingress | Internet Egress | Primary Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 1: Edge Public** | Public Multi-AZ | Allowed (Port 443) | Allowed | External Traffic Routing & Load Balancing |
| **Tier 2: Core Compute** | Private Multi-AZ | Denied | Outbound-Only (NAT) | Microservices & Business Logic Execution |
| **Tier 3: Database** | Isolated Multi-AZ | Denied | Denied | High-Value Financial Ledger & Data Storage |

---

## Defensive Engineering Patterns Applied

* **State Plane Immutability:** Enforces zero structural drifts by deploying completely stateless compute nodes inside isolated autoscaling groups.
* **Strict Control Plane Separation:** Eliminates cross-environment pollution by locking down database network interfaces exclusively to internal app layer security security groups.
* **Privileged Endpoint Routing:** Bypasses public network pathways entirely for internal secrets access via managed VPC gateway interfaces.
