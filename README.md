# EWB AWS VPC Peering Project

**Project:** VPC Peering: Connect Two VPCs  
**Purpose:** Build a private connection between two non-overlapping Amazon VPCs and verify private IPv4 connectivity between EC2 instances.

> **Evidence honesty:** This is a project starter package and report template. The images in `screenshots/` are labeled placeholders, not proof that the AWS lab was performed. Replace them with your own AWS Console and terminal screenshots. Do not submit placeholders as completed evidence.

## Architecture and IP plan
- **VPC A:** `10.10.0.0/16`
  - Public subnet: `10.10.1.0/24` (bastion)
  - Private subnet: `10.10.2.0/24` (App-A)
- **VPC B:** `10.20.0.0/16`
  - Private subnet: `10.20.1.0/24` (App-B)
- **Peering:** `ewb-a-b-peer`
- **Region:** Mumbai (`ap-south-1`) if approved by your instructor

## Package contents
- `project-report.pdf` — formatted report template, architecture, steps, tests and cleanup.
- `architecture/vpc-peering-architecture.png` — network architecture diagram.
- `documentation/project-steps.md` — beginner-friendly build guide.
- `screenshots/` — eight labeled placeholders to replace with real evidence.

## Safety and cost notes
1. Confirm AWS account permissions, credits and current pricing before launching resources.
2. EC2, EBS and public IPv4 usage may incur charges. The design does not require a NAT Gateway.
3. Keep the private key on your own computer. Never include it in the report or ZIP.
4. Never expose SSH or ICMP to `0.0.0.0/0`.
5. Record actual AWS resource IDs, IP addresses and real test outputs after doing the lab.

## Quick verification
After peering is Active and routes/security groups are configured, from App-A run:
```bash
ping -c 4 10.20.1.10
```
From App-B run:
```bash
ping -c 4 10.10.2.10
```
Expected: replies if routing, security groups, network ACLs and instance settings permit ICMP.

## Cleanup
Save real evidence first. Terminate the three lab EC2 instances; inspect/delete leftover EBS volumes and Elastic IPs; remove peering routes and the peering connection; delete lab security groups, subnets, custom route tables and Internet Gateway; then delete the two VPCs. Check Billing for remaining chargeable resources.
