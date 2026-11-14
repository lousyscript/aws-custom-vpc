# Building a Custom VPC on AWS: Public vs Private Subnets

Hands-on project: I built a VPC from scratch (instead of using the default one), launched servers in a public and a private subnet, broke the routing on purpose, and reached the private server through a jump host.

**Services used:** VPC, Subnets, Internet Gateway, Route Tables, EC2, Security Groups
**Region:** eu-north-1 (Stockholm)

## Architecture

```mermaid
flowchart TB
  Internet((Internet)) --> IGW[Internet Gateway]
  subgraph VPC["cspm-vpc 10.0.0.0/16"]
    IGW --> RT[public-rt<br/>0.0.0.0/0 → IGW]
    RT --> PUB
    subgraph PUB["public-subnet 10.0.1.0/24"]
      EC2A[Public EC2<br/>jump host + nginx]
    end
    subgraph PRIV["private-subnet 10.0.2.0/24<br/>route table: local only"]
      EC2B[Private EC2<br/>no public IP]
    end
    EC2A -. SSH from 10.0.1.0/24 .-> EC2B
  end
```

| Component | Setting |
|---|---|
| VPC | `10.0.0.0/16` |
| Public subnet | `10.0.1.0/24`, auto-assign public IP on |
| Private subnet | `10.0.2.0/24`, auto-assign public IP off |
| Internet gateway | Attached to the VPC |
| Public route table | `0.0.0.0/0 → igw`, associated with the public subnet only |
| Private route table | Default (local route only) |

Screenshot: `screenshots/01-vpc-and-subnets.png`, `screenshots/02-public-route-table.png`

## 1. Public instance
- Launched in the public subnet with a public IP
- Security group: SSH from my IP only, HTTP from anywhere
- Ran nginx in a Docker container and confirmed it loaded in a browser

## 2. The bug I hit
SSH timed out even though the public IP, security group, and internet gateway were all correct.

**Cause:** I never associated the public subnet with the public route table, so it used the VPC's main route table, which has only the local route. There was no path to the internet gateway.

**Lesson:** a subnet is only "public" if its route table sends `0.0.0.0/0` to an internet gateway. A public IP and an open security group aren't enough.

Screenshot: `screenshots/03-subnet-association-bug.png`

## 3. Breaking it on purpose
I removed the `0.0.0.0/0 → igw` route from the public route table.

**Result:** SSH and HTTP both stopped working, even though the security group still allowed them. I added the route back and access returned.

**Takeaway:** security groups and routing are separate layers, and both have to allow traffic.

Screenshot: `screenshots/04-route-removed.png`
Screenshot: `screenshots/04b-route-table-without-igw.png`

## 4. Private instance and jump host
- Launched a second instance in the private subnet with **no public IP**
- Security group: SSH only from the public subnet range (`10.0.1.0/24`)
- Connected through the public instance using SSH `ProxyJump`, so my private key stayed on my computer and was never copied to a server
- From inside the private instance, `curl https://example.com` timed out, because the subnet has no route to the internet

Screenshot: `screenshots/05-private-instance-no-public-ip.png`, `screenshots/06-private-no-internet.png`

## 5. Public vs private summary

| | Public subnet | Private subnet |
|---|---|---|
| Route to internet gateway | Yes | No |
| Public IP on instances | Yes | No |
| Reachable from the internet | Yes (if the security group allows) | No |
| Outbound internet | Yes | No (would need a NAT gateway) |

## 6. Security notes and what a CSPM could check
- Security group rules open to `0.0.0.0/0` on SSH
- A "private" subnet that has a route to an internet gateway
- A "public" subnet without an IGW route (the bug I hit)
- Instances in private subnets that have public IPs
- Possible improvement: allow SSH to the private instance from the jump host's **security group** instead of a CIDR range, to limit it to one instance

I deliberately did not create a NAT gateway, since it adds hourly charges that aren't needed for this exercise.

## 7. Next steps
- Attach an IAM role so an instance can reach S3 without stored credentials
- Build a mini-CSPM in Python (boto3) that automatically flags the issues above