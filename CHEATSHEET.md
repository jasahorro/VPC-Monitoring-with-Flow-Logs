# VPC Monitoring with Flow Logs — Cheat Sheet

Lesson notes for reviewing later. The project write-up is in [README.md](README.md).

## Contents

1. [What I built](#what-i-built)
2. [Key terms](#key-terms)
3. [Why network monitoring matters](#why-network-monitoring-matters)
4. [Setting up the VPCs](#setting-up-the-vpcs)
5. [Launching the EC2 instances](#launching-the-ec2-instances)
6. [CloudWatch and log groups](#cloudwatch-and-log-groups)
7. [Flow logs and network interfaces](#flow-logs-and-network-interfaces)
8. [IAM policy, role and trust policy](#iam-policy-role-and-trust-policy)
9. [Ping testing and troubleshooting](#ping-testing-and-troubleshooting)
10. [Peering connection and routes](#peering-connection-and-routes)
11. [Reading a flow log](#reading-a-flow-log)
12. [Logs Insights](#logs-insights)
13. [Build order and cleanup](#build-order-and-cleanup)
14. [Self-quiz](#self-quiz)

---

## What I built

- 🌉 **Two VPCs joined by a peering connection**, each with one public subnet and an EC2 instance.
- 🛶 **VPC Flow Logs** that record the traffic going in and out of the VPC's network interfaces.
- 📒 **A CloudWatch log group** where the flow logs are stored.
- 🔐 **An IAM policy and role** (with a custom trust policy) that let VPC Flow Logs write to CloudWatch.
- 🧠 **A Logs Insights query** that finds the top 10 data transfers between IP addresses.

```
        NextWork-1-vpc (10.1.0.0/16)                  NextWork-2-vpc (10.2.0.0/16)
  ┌──────────────────────────────────┐         ┌──────────────────────────────────┐
  │  Instance - NextWork VPC 1       │  ping   │  Instance - NextWork VPC 2       │
  │  SG: All ICMP - IPv4 0.0.0.0/0   │ ──────► │  SG: All ICMP - IPv4 0.0.0.0/0   │
  └───────────────┬──────────────────┘  (ICMP) └──────────────────────────────────┘
   Route: 10.2.0.0/16 → pcx-…  ◄──── peering connection (pcx-…) ────►  Route: 10.1.0.0/16 → pcx-…
                  │
                  │ VPC Flow Logs (filter: All)
                  ▼                       IAM role (trusted: vpc-flow-logs.amazonaws.com)
     CloudWatch log group  ◄───────────── lets Flow Logs create streams + put log events
       └─ log stream per network interface (eni-…)
                  │
                  ▼
           Logs Insights query → top 10 byte transfers
```

## Key terms

| Term | Plain meaning | Analogy |
|---|---|---|
| **Network monitoring** | Tracking and analysing the traffic in your network | Security cameras plus a visitor logbook for a building |
| **Amazon CloudWatch** | AWS's monitoring service: collects metrics and logs, builds dashboards, triggers actions | The control room watching every camera |
| **Metrics** | Stats about a service's performance | A car's dashboard readings (speed, fuel) |
| **Logs** | A record of everything that happens in a system | A diary for your computer systems |
| **Log group** | A container in CloudWatch holding related logs | A big folder for related diaries |
| **Log stream** | One sequence of log events inside a log group (here, one per network interface) | One diary inside the folder |
| **VPC Flow Logs** | A VPC feature that records traffic going to and from network interfaces | A visitor logbook for the VPC: who came, where to, and whether they got in |
| **Network interface (ENI)** | The component that connects a resource (like an EC2 instance) to the VPC and gives it its IPs | A specific building in a neighbourhood (the subnet) |
| **IAM policy** | Rules saying what actions are allowed or denied | Hotel rules: who can use the gym, the staff lift, etc. |
| **IAM role** | A bundle of policies you assign to a service or user | A hotel key card that carries a mix of those permissions |
| **Custom trust policy** | Defines exactly *who* is allowed to use a role | A VIP list at the door |
| **Logs Insights** | CloudWatch feature for querying and analysing logs | A search-and-filter tool for the logbook |
| **Query** | A command that filters, groups or calculates over log data | The code version of the filter buttons in a spreadsheet |
| **Peering connection** | A direct private link between two VPCs | A private bridge between two gated neighbourhoods |

## Why network monitoring matters

A network exists to:
1. Let resources talk to each other and to things outside the VPC.
2. Make sure they do it in the **most secure way possible**.

Network monitoring is how you know whether the network is doing both of those well. It includes:

- 📊 Checking the **source** of traffic coming into your VPC
- 📊 Analysing **how much data** is being transferred
- 📊 Identifying traffic **blocked** by your security settings

At a more advanced level it also covers speeding up data transfers and automating responses to security threats.

## Setting up the VPCs

Same setup as the VPC Peering project (see that cheat sheet for tenancy, NAT gateways and VPC endpoints in more detail):

- **VPC and more** wizard, two VPCs: **NextWork-1-vpc** (`10.1.0.0/16`) and **NextWork-2-vpc** (`10.2.0.0/16`).
- Each with **1 Availability Zone, 1 public subnet, 0 private subnets**, **no NAT gateways**, **no VPC endpoints**.
- CIDRs are numbered to match the VPC names, just to make them easy to remember.
- **CIDRs must be unique**, otherwise IP addresses overlap and cause routing conflicts.

| Setting | Quick reminder |
|---|---|
| **Tenancy: Default** | Shares hardware with other AWS customers; cheaper. *Dedicated* = your own hardware, for strict compliance, costs more |
| **NAT gateway** | Lets private subnets reach the internet (e.g. for updates) but blocks inbound traffic. An **internet gateway** allows both ways for public subnets |
| **VPC endpoint** | Private connection from your VPC to AWS services like S3, without the public internet. Covered in the next project |

## Launching the EC2 instances

- **Edit Network settings**: by default instances launch into the account's **default VPC**, so pick NextWork-1-vpc (and NextWork-2-vpc for the second one).
- **No key pair**: EC2 Instance Connect manages the key for you.
- **EC2 Instance Connect needs** the instance to be in a **public subnet** AND have a **public IPv4 address**, so enable auto-assign public IP.
- **Create a new security group** instead of using the VPC's default one. The default SG caused connectivity problems in the last project.
- Inbound rule: **All ICMP - IPv4** from **anywhere (`0.0.0.0/0`)**. This is needed for the ping test from public IPs later.

> ⚠️ Allowing ICMP from `0.0.0.0/0` means anyone on the internet can ping the instance. Fine for a lab, but it shows how exposed an instance is when you open it to all IPs.

## CloudWatch and log groups

- **CloudWatch** collects **metrics** from AWS services and turns them into graphs and dashboards. It can also trigger **actions** automatically based on metrics.
- **CloudWatch data is region-specific.** Use the **same Region** as your VPCs, or you won't see their data.
- A **log group** is a folder for related logs, usually from the same source. Logs are saved in the region they were created in (dashboards can combine regions).

**Log group settings we left as default:**

| Setting | Default | Meaning |
|---|---|---|
| **Retention** | Never expire | Logs are kept until you delete them |
| **Log class** | Standard | For logs you access and analyse regularly. *Infrequent Access* = cheaper storage but you pay more each time you query it; good for long-term archiving |

## Flow logs and network interfaces

- A **flow log** records traffic going to and from a network interface: who's visiting your VPC and which interface they're going to.
- **Network interfaces (ENIs)** connect resources to the VPC. Launching an EC2 instance automatically creates a primary ENI with a private IP from the subnet's CIDR. That's how the instance gets its private IP.
- Flow logs track traffic **per network interface** because each ENI is a distinct entry/exit point. No two resources share an ENI.
- An instance can have more than one ENI, e.g. to belong to two subnets at once.

**Filter options:**

| Filter | Captures | Good for |
|---|---|---|
| **All** — *used here* | All traffic in and out | Full picture of the network |
| **Accept** | Only traffic allowed by security groups and NACLs | Understanding what's being let through |
| **Reject** | Only traffic that was blocked | Spotting illegitimate traffic or misconfigured security settings |

**Destination options:**

| Destination | Good for |
|---|---|
| **CloudWatch Logs** — *used here* | Searching and analysing logs (e.g. with Logs Insights) |
| **Amazon S3** | Cheap, long-term archiving; feeding other analytics tools later |
| **Amazon Data Firehose** | Processing and analysing data in **real time** |

## IAM policy, role and trust policy

VPC Flow Logs **can't write to CloudWatch by default**. It needs permission through an IAM role.

**The 3-step process:**
1. Create an **IAM policy** (the permissions)
2. Create an **IAM role** and attach the policy
3. Assign the role to the service (pick it when creating the flow log)

> You can't attach a policy directly to a service. Services get **roles**, and roles carry policies.

**The permissions policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

| Part | Meaning |
|---|---|
| `Version: "2012-10-17"` | The IAM policy language version (latest). Older policies use `2008-10-17` |
| `Effect: "Allow"` | The listed actions are allowed. Changing it to `"Deny"` would block Flow Logs immediately |
| `logs:CreateLogGroup` | Create new log groups |
| `logs:CreateLogStream` | Create log streams inside those groups |
| `logs:PutLogEvents` | Send log data into the streams |
| `logs:DescribeLogGroups` / `DescribeLogStreams` | See information about the groups and streams |
| `Resource: "*"` | Applies to all log groups and streams |

**The role's custom trust policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "vpc-flow-logs.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

- **IAM policy** = *what* actions are allowed. **Trust policy** = *who* can use the role.
- `"Service": "vpc-flow-logs.amazonaws.com"` means **only VPC Flow Logs** can use this role. Even if the role is accidentally given to another service or user, they can't use it.
- **Principal** = the entity being trusted. Here it's a service; it can also be IAM users or roles.
- **Why not the "AWS service" trusted entity option?** VPC Flow Logs isn't in that list, because it's a feature of Amazon VPC rather than a standalone service. So you have to use a custom trust policy.

## Ping testing and troubleshooting

- **Ping** sends a small packet to an IP and waits for a reply, showing whether you got one and how long it took.
- Test with Instance 2's **private IPv4 address**. That's the only way to tell if private (peered) communication works. A public IP would succeed over the internet and tell you nothing about peering.
- `ping -c 5 <ip>` sends exactly **5** packets and stops, giving a consistent number of tests.

**What happened:**

| Test | Result | Meaning |
|---|---|---|
| Instance 1 → Instance 2 **private IP** (before peering) | ❌ No replies | No private route between the VPCs yet |
| Instance 1 → Instance 2 **public IP** | ✅ Replies | Instance 2 responds to ICMP fine; traffic went over the public internet |
| Local computer → Instance 2 **public IP** | ✅ Replies (~147 ms) | Instance 2 is reachable from anywhere on the internet |
| Instance 1 → Instance 2 **private IP** (after peering + routes) | ✅ Replies | Peering works |

> **Latency:** ~147 ms from a local computer vs ~0.5 ms from Instance 1. Both instances are in the same Availability Zone, so they're physically much closer.

**Why it failed:** the security group already allowed ICMP, so the problem was **routing**. VPC 1's route table had no route to `10.2.0.0/16`.

> **"But there's a `0.0.0.0/0` route!"** That route goes to the **internet gateway**, so traffic would travel over the public internet. A peering connection is for a **direct, private** link, so you need a route pointing at the peering connection instead.

## Peering connection and routes

- **Peering connection**: direct link between two VPCs so their resources can talk using **private IPs**, without the public internet.
- **Requester** = VPC 1 (sends the invite). **Accepter** = VPC 2 (must **Accept request** before the connection works).
- Peering can also be **between different AWS accounts**, letting businesses share resources without exposing them to the internet.
- Accepting isn't enough: each VPC needs a **route**.

| VPC | Destination | Target |
|---|---|---|
| NextWork-1-vpc | `10.2.0.0/16` | Peering connection (`pcx-…`) |
| NextWork-2-vpc | `10.1.0.0/16` | Peering connection (`pcx-…`) |

## Reading a flow log

- Log streams are named after the network interface: **`eni-xxx`** (Elastic Network Interface).
- Every EC2 instance must have an ENI to exist in a VPC. The ENI is what gives it IPs, security group rules and network access.

**Default flow log format** (one line per record):

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

**My example** (from the README):

```
2 <account-id> eni-049be7c49b59686f5 217.146.80.99 10.1.13.67 27606 37777 6 1 40 1791769157 1791769179 REJECT OK
```

| Field | Value | Meaning |
|---|---|---|
| version | `2` | Flow log format version |
| interface-id | `eni-049be…` | The network interface the traffic hit |
| srcaddr → dstaddr | `217.146.80.99` → `10.1.13.67` | An outside IP trying to reach my instance's private IP |
| srcport → dstport | `27606` → `37777` | Destination port 37777 (not a port I opened) |
| protocol | `6` | TCP (`1` = ICMP, `17` = UDP) |
| packets / bytes | `1` / `40` | One small packet |
| start / end | Unix timestamps | When the capture window started and ended |
| action | `REJECT` | Blocked by the security group / NACL |
| log-status | `OK` | Logged normally |

> **Lesson's example:** 344 bytes from `18.237.140.165` to `10.1.5.112`, TCP port 22, 4 packets, `ACCEPT`. That source is in EC2 Instance Connect's Oregon IP range (`18.237.140.160/29`), so the log was Instance Connect's SSH connection to Instance 1.

## Logs Insights

- **Logs Insights** is a CloudWatch feature for analysing logs with **queries**.
- **Queries** filter, group and calculate (sums, averages) over large volumes of logs to find trends, outliers and problems.
- I used the sample query **"Top 10 byte transfers by source and destination IP addresses"**:

```
stats sum(bytes) as TotalBytes by srcAddr, dstAddr
| sort TotalBytes desc
| limit 10
```

| Part | What it does |
|---|---|
| `stats sum(bytes) … by srcAddr, dstAddr` | Groups traffic by source/destination pair and adds up the bytes |
| `sort TotalBytes desc` | Puts the biggest transfers first |
| `limit 10` | Only shows the top 10 |

- **Results:** the 10 IP pairs that moved the most data. Great for spotting heavy or unusual traffic.
- The **bar chart** at the top only shows how many logs were captured over time. The **table** is the actual query result.

## Build order and cleanup

1. Create **two VPCs** (`10.1.0.0/16`, `10.2.0.0/16`), each with one public subnet.
2. Launch an **EC2 instance in each**, with a new security group allowing **All ICMP - IPv4** from anywhere.
3. Create a **CloudWatch log group** (same Region as the VPCs).
4. Start creating a **flow log** on the VPC (filter: All, destination: CloudWatch Logs).
5. Create the **IAM policy**, then an **IAM role** with a **custom trust policy** for `vpc-flow-logs.amazonaws.com`. Finish the flow log with that role.
6. Connect to Instance 1 with **EC2 Instance Connect**, ping Instance 2's **private IP** → fails. Ping its **public IP** → works.
7. Create a **peering connection** (VPC 1 = Requester), **accept** it as VPC 2, and add a **route** in each VPC's route table.
8. Ping the **private IP** again → works.
9. Open the **log group → log streams (`eni-…`)** to read flow logs, then run a **Logs Insights** query.

**Cleanup:** terminate both EC2 instances → delete the flow log and the CloudWatch log group → delete the IAM role and policy → delete the peering connection → delete both VPCs. If a VPC won't delete, check for leftover **network interfaces** and delete them manually.

---

## Self-quiz

<details><summary>1. What is network monitoring, and what three things does it include?</summary>

Tracking and analysing your network's traffic. It includes checking the source of incoming traffic, analysing how much data is transferred, and identifying traffic blocked by your security settings.
</details>

<details><summary>2. Why must you use the same Region for CloudWatch as your VPCs?</summary>

CloudWatch data is region-specific. Data collected in one region isn't automatically available in another.
</details>

<details><summary>3. What's the difference between a log group and a log stream?</summary>

A log group is a container (folder) for related logs. A log stream is one sequence of log events inside it. For flow logs there's one stream per network interface (`eni-…`).
</details>

<details><summary>4. What's the difference between the Standard and Infrequent Access log classes?</summary>

Standard is for logs you access regularly. Infrequent Access has cheaper storage but costs more each time you access the logs, so it suits long-term archiving.
</details>

<details><summary>5. What does a VPC flow log record?</summary>

Traffic going to and from network interfaces: source and destination IPs and ports, protocol, packets, bytes, start and end time, and whether it was accepted or rejected.
</details>

<details><summary>6. Why are flow logs tied to network interfaces?</summary>

Each network interface is a distinct point where traffic enters or leaves a resource, and no two resources share one.
</details>

<details><summary>7. Which flow log filter would you use to find illegitimate traffic trying to get in?</summary>

**Reject**. It only captures traffic blocked by your security groups and network ACLs.
</details>

<details><summary>8. Name the three flow log destinations and when to use each.</summary>

CloudWatch Logs for searching and analysis, Amazon S3 for cheap long-term archiving, and Amazon Data Firehose for real-time processing.
</details>

<details><summary>9. Why did VPC Flow Logs need an IAM role?</summary>

By default it has no permission to create log streams and send log events to CloudWatch. The role carries a policy that grants those permissions.
</details>

<details><summary>10. What's the difference between an IAM policy and an IAM role?</summary>

A policy is a set of rules about what's allowed or denied. A role bundles policies together and is assigned to a service or user. You can't attach a policy directly to a service.
</details>

<details><summary>11. What does a custom trust policy do, and why was it needed here?</summary>

It defines exactly who can use a role. It was needed because VPC Flow Logs isn't listed under the "AWS service" trusted entity option, since it's a feature of Amazon VPC rather than its own service.
</details>

<details><summary>12. What would happen if you changed `"Effect": "Allow"` to `"Deny"` in the policy?</summary>

Flow Logs would immediately be blocked from creating and sending logs.
</details>

<details><summary>13. Why test with the private IP instead of the public IP?</summary>

Only private-IP communication proves the peering connection works. A public IP succeeds over the internet whether or not peering is set up.
</details>

<details><summary>14. The security group allows ICMP, but pinging the private IP fails. Why?</summary>

There's no route. VPC 1's route table has no entry sending `10.2.0.0/16` to a peering connection, because the peering connection doesn't exist yet.
</details>

<details><summary>15. The route table has `0.0.0.0/0`. Why doesn't that count?</summary>

It points to the internet gateway, so traffic goes over the public internet. A peering connection is for direct, private communication.
</details>

<details><summary>16. Why was the ping much faster from Instance 1 than from your own computer?</summary>

Instance 1 is in the same Availability Zone as Instance 2, so it's physically much closer (~0.5 ms vs ~147 ms).
</details>

<details><summary>17. What does `ping -c 5` do?</summary>

Sends exactly 5 ping packets and then stops.
</details>

<details><summary>18. In a flow log, what do protocol `6` and `REJECT` mean?</summary>

Protocol 6 is TCP. REJECT means the traffic was blocked by a security group or network ACL.
</details>

<details><summary>19. What does the "Top 10 byte transfers" query do?</summary>

Groups traffic by source and destination IP, sums the bytes for each pair, sorts biggest first and shows the top 10, revealing the busiest traffic routes.
</details>

<details><summary>20. A VPC won't delete during cleanup. What should you check?</summary>

Leftover network interfaces. They're usually deleted with the EC2 instance, but sometimes you need to delete them manually.
</details>
