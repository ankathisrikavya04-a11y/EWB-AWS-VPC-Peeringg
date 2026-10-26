# Project Steps — EWB AWS VPC Peering

This guide combines the two supplied project documents. Use the workbook's names and IP plan consistently. Record your actual AWS resource IDs and test results.

## 1. Network plan

| Resource | CIDR / address |
|---|---|
| `ewb-vpc-a` | `10.10.0.0/16` |
| `ewb-a-public` | `10.10.1.0/24` |
| `ewb-a-private` | `10.10.2.0/24` |
| `ewb-vpc-b` | `10.20.0.0/16` |
| `ewb-b-private` | `10.20.1.0/24` |
| `ewb-bastion` | `10.10.1.10` plus public IPv4 |
| `ewb-app-a` | `10.10.2.10`, no public IPv4 |
| `ewb-app-b` | `10.20.1.10`, no public IPv4 |

The VPC CIDRs must not overlap.

## 2. Create VPC A and subnets
1. Open **VPC → Your VPCs → Create VPC → VPC only**.
2. Create `ewb-vpc-a`, IPv4 CIDR `10.10.0.0/16`.
3. Create `ewb-a-public` (`10.10.1.0/24`) and `ewb-a-private` (`10.10.2.0/24`).
4. Create `ewb-igw-a` and attach it to VPC A.
5. Create `ewb-rt-a-public`; associate the public subnet; add `0.0.0.0/0 → ewb-igw-a`.
6. Create `ewb-rt-a-private`; associate the private subnet. Keep only the local route until peering is configured. Do not add an internet default route.

## 3. Create VPC B
1. Create `ewb-vpc-b` with CIDR `10.20.0.0/16`.
2. Create `ewb-b-private` with CIDR `10.20.1.0/24`.
3. Create `ewb-rt-b-private` and associate it with `ewb-b-private`.
4. Keep the local route initially. VPC B does not need an Internet Gateway for this design.

## 4. Security groups
Create the following inbound rules. Keep default outbound rules for this beginner lab unless your instructor says otherwise.

| Security group | Inbound rule | Source |
|---|---|---|
| `ewb-sg-bastion` | SSH / TCP 22 | Your current public IPv4 `/32` only |
| `ewb-sg-app-a` | SSH / TCP 22 | `10.10.1.10/32` |
| `ewb-sg-app-a` | ICMP Echo Request | `10.20.1.10/32` |
| `ewb-sg-app-b` | SSH / TCP 22 | `10.10.1.10/32` |
| `ewb-sg-app-b` | ICMP Echo Request | `10.10.2.10/32` |

Never expose SSH or ICMP to `0.0.0.0/0`. If you use different instance IPs, update the rules. This lab assumes default network ACLs.

## 5. Launch EC2 instances
Use Amazon Linux 2023 and an instructor-approved small instance type. The workbook gives `t3.micro` as an example; confirm current pricing and credits.

| Name | Subnet | Private IP | Public IPv4 |
|---|---|---|---|
| `ewb-bastion` | `ewb-a-public` | `10.10.1.10` | Enabled |
| `ewb-app-a` | `ewb-a-private` | `10.10.2.10` | Disabled |
| `ewb-app-b` | `ewb-b-private` | `10.20.1.10` | Disabled |

Create key pair `ewb-lab`, download the `.pem` key and keep it on your own laptop. Do not upload it into the project folder. Wait for all instances to pass status checks.

## 6. Configure SSH from your laptop
For WSL, macOS or Linux, store the key at `~/.ssh/ewb-lab.pem`, then run:
```bash
chmod 400 ~/.ssh/ewb-lab.pem
```
Add this to `~/.ssh/config`, replacing `BASTION_PUBLIC_IP` with the actual public IPv4:
```sshconfig
Host ewb-bastion
    HostName BASTION_PUBLIC_IP
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes

Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion

Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```
Test `ssh ewb-bastion`, then `ssh ewb-app-a`. Use `exit` to return to your laptop. Windows users can use WSL or an instructor-approved OpenSSH setup.

## 7. Capture baseline failure
Before creating peering routes, connect to App-A and run:
```bash
ping -c 4 10.20.1.10
```
Save the real terminal output. The expected result is no replies because the peer route has not been added.

## 8. Create and accept VPC peering
1. Go to **VPC → Peering connections → Create peering connection**.
2. Name: `ewb-a-b-peer`.
3. Requester: `ewb-vpc-a`; accepter: `ewb-vpc-b`; same account and Region for this lab.
4. Accept the request and wait for **Active**. Record the actual `pcx-...` ID.

## 9. Add routes on both sides
Keep the existing local routes and add:

| Route table | Destination | Target |
|---|---|---|
| `ewb-rt-a-private` | `10.20.0.0/16` | VPC peering ID (`pcx-...`) |
| `ewb-rt-b-private` | `10.10.0.0/16` | VPC peering ID (`pcx-...`) |
| `ewb-rt-a-public` | `10.20.0.0/16` | VPC peering ID (`pcx-...`) |
| `ewb-rt-a-public` | `0.0.0.0/0` | Internet Gateway (`ewb-igw-a`, existing) |

The public-subnet peer route is needed because the bastion must reach App-B for SSH ProxyJump. The return route in VPC B is necessary for responses. Peering does not give VPC B internet access through VPC A.

## 10. Test connectivity
From App-A:
```bash
hostname
hostname -I
ping -c 4 10.20.1.10
```
From App-B:
```bash
hostname
hostname -I
ping -c 4 10.10.2.10
```
From your laptop:
```bash
ssh ewb-app-b
```
Expected acceptance criteria: pings receive replies (ideally 0% packet loss), SSH opens a shell on App-B through the bastion, and App-A/App-B have no public IPv4 addresses. Capture actual results; do not write expected output as if it were real.

## 11. Controlled failure and recovery
1. In `ewb-sg-app-b`, temporarily remove only the inbound ICMP Echo Request rule from `10.10.2.10/32`.
2. Repeat the App-A ping. Replies should stop.
3. Restore the exact ICMP rule and repeat the ping. Replies should return.
4. Capture screenshots of the failure and recovery. Keep the SSH rule in place.

## 12. Troubleshooting
- Bastion SSH timeout: verify its current public IP, your public IPv4 `/32` rule, subnet association and default route to the attached IGW.
- App-A SSH fails: verify private IP, TCP 22 source `10.10.1.10/32`, key and `ec2-user`.
- App-B SSH fails: check public route table's `10.20.0.0/16 → pcx-...`, VPC B return route and App-B SSH source.
- Ping fails after peering: confirm peering Active, both private routes, correct destination IP and ICMP inbound rule.
- Peering creation fails: inspect all VPC CIDRs for overlap.
- `Permission denied (publickey)`: check key, username and `IdentityFile`; this can occur even when routing is fine.

## 13. Cleanup
After saving evidence:
1. Terminate `ewb-bastion`, `ewb-app-a`, and `ewb-app-b`.
2. Check/delete remaining lab EBS volumes and snapshots; release any lab Elastic IP.
3. Remove peering routes and delete `ewb-a-b-peer`.
4. After network interfaces are gone, delete lab security groups, subnets, custom route tables and `ewb-igw-a`.
5. Delete both lab VPCs. Delete the key pair only if no longer needed.
6. Check the Billing console for remaining chargeable resources. Usage reporting can lag.

## Screenshot checklist
Replace the eight files in `screenshots/` with your own captures:
1. VPC configuration and CIDRs
2. Subnets
3. Route tables and subnet associations
4. Security group inbound rules
5. EC2 instances and public/private IPs
6. Peering connection Active
7. Connectivity test outputs (baseline, success, controlled failure/recovery)
8. Cleanup evidence

## Report values to fill in
- Student / batch / college:
- Region:
- Completion date:
- VPC A ID / VPC B ID:
- Peering connection ID:
- Bastion instance ID / public IPv4:
- App-A instance ID / private IPv4:
- App-B instance ID / private IPv4:
- What worked:
- Issue and resolution:
- What I learned:
