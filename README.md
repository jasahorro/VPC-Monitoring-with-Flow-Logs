<img src="https://cdn.prod.website-files.com/677c400686e724409a5a7409/6790ad949cf622dc8dcd9fe4_nextwork-logo-leather.svg" alt="NextWork" width="300" />

# VPC Monitoring with Flow Logs

**Project Link:** [View Project](https://nextwork.ai/projects/6954a600-a89e-56b0-886b-7054da118a3b)

**Author:** Maria Jasmin Ahorro  
**Email:** jasminahorro23@gmail.com

---

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_3e1e79a1)

## Introducing Today's Project!

### What is Amazon VPC?

Amazon VPC is a service that lets you provision a logically isolated section of the Amazon Web Services (AWS) cloud where you can launch AWS resources in a virtual network that you define. It is useful because it gives you complete control over your virtual networking environment—including selection of IP address ranges, creation of subnets, and configuration of route tables and network gateways—allowing you to build secure, multi-tier architectures with advanced security features like security groups and network access control lists.

### How I used Amazon VPC in this project

In today's project, I used Amazon VPC to provision two logically isolated virtual networks (NextWork-1-vpc and NextWork-2-vpc) with public subnets, deploy EC2 instances into each, establish a secure VPC peering connection to bridge them together, configure route tables for private inter-VPC traffic routing, and set up VPC Flow Logs with Amazon CloudWatch to monitor network traffic, capture packet metrics, and analyze security rules and data transfer rates using Logs Insights queries.

### One thing I didn't expect in this project was...

One thing I didn't expect in this project was how seamlessly Amazon CloudWatch Logs Insights and VPC Flow Logs capture deep network metadata such as exact byte counts, protocol details, and port numbers turning raw packet traffic into actionable security insights for troubleshooting inter-VPC connectivity and blocked connection attempts.

### This project took me...

This project took me 50 minutes only to do! 

## In the first part of my project...

### Step 1 - Set up VPCs

In this step, I will set up two separate Virtual Private Clouds (VPCs) with public subnets because we need a multi-VPC architecture to test peering connections and monitor network traffic flows between them.

### Step 2 - Launch EC2 instances

In this step, I will launch an EC2 instance in each VPC because we need to deploy compute resources within our multi-VPC architecture to test connectivity and generate network traffic flows.

### Step 3 - Set up Logs

In this step, I will create a CloudWatch log group and set up VPC Flow Logs for my VPC because we need a place to store and monitor the network traffic data that flows in and out of our subnets.

### Step 4 - Set IAM permissions for Logs

In this step, I will configure the IAM permissions and set up a role for VPC Flow Logs because we need to grant our VPC the necessary permissions to securely write log events and publish them to our CloudWatch log group.

## Multi-VPC Architecture

I started my project by launching two separate Virtual Private Clouds (NextWork-1-vpc and NextWork-2-vpc) using the "VPC and more" wizard, each configured with one public subnet in a single Availability Zone without NAT gateways or VPC endpoints.

The CIDR blocks for VPCs 1 and 2 are 10.1.0.0/16 and 10.2.0.0/16. They have to be unique because each VPC must have a distinct IPv4 range so that the IP addresses of their resources do not overlap, which prevents routing conflicts and connectivity issues.

### I also launched EC2 instances in each subnet

My EC2 instances' security groups allow all ICMP traffic (All ICMP - IPv4) from anywhere (0.0.0.0/0). This is because we need to run ping tests to verify network connectivity and test our VPC peering connection between the instances.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_e7fa8775)

## Logs

Logs are digital diaries that computer systems keep to record everything that happens, from user logins to system errors, which help engineers troubleshoot problems and monitor activity.

Log groups are containers in Amazon CloudWatch that store and organize related log streams with the same retention, monitoring, and access control settings.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_e8398869)

## IAM Policy and Roles

I created an IAM policy because VPC Flow Logs need explicit permissions to create log groups, create log streams, and publish log events to Amazon CloudWatch. Without this policy attached via an IAM role, the VPC cannot securely write or export its network traffic logs.

I also created an IAM role because VPC Flow Logs need a designated role with a custom trust policy so that the logging service is granted the necessary permissions to securely write and publish traffic logs to Amazon CloudWatch.

A custom trust policy is a specialized policy type used in an IAM role to explicitly define and restrict which specific AWS services or entities are permitted to assume that role. In this project, we used it to ensure that only the VPC Flow Logs service (vpc-flow-logs.amazonaws.com) can utilize the IAM role.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_4334d777)

## In the second part of my project...

### Step 5 - Ping testing and troubleshooting

In this step, I will test the network connectivity and ping responses between EC2 instances across our peered VPCs because we need to verify whether our instances can communicate using their private IP addresses and identify any missing routing configurations.

### Step 6 - Set up a peering connection

In this step, I will create a VPC peering connection and update the route tables for both VPCs because we need to establish a direct, private communication link between our separate virtual networks so that EC2 instances can successfully route traffic using their private IP addresses.

### Step 7 - Analyze flow logs

In this step, I will analyze the recorded network flow logs using Amazon CloudWatch Logs Insights because we need to inspect traffic patterns, trace data transfers, and identify connection behavior across our peered virtual private clouds.

## Connectivity troubleshooting

My first ping test between my EC2 instances had no replies, which means the EC2 instances were unable to communicate over their private IP addresses because the VPC peering connection and direct routing rules between VPC 1 and VPC 2 had not yet been established.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_99d4ba42)

I could receive ping replies if I ran the ping test using the other instance's public IP address, which means Instance 2 is correctly configured to respond to ICMP traffic and is reachable. However, the traffic was routed across the public internet rather than our private internal network, because the VPC peering connection and private routing rules were not yet set up.

## Connectivity troubleshooting

Looking at VPC 1's route table, I identified that the ping test with Instance 2's private address failed because there was no direct route directing traffic destined for VPC 2's CIDR block (10.2.0.0/16) to the VPC peering connection, causing private inter-VPC packets to be unroutable.

### To solve this, I set up a peering connection between my VPCs

I also updated both VPCs' route tables so that traffic destined for each other's CIDR blocks (10.2.0.0/16 and 10.1.0.0/16) is directly routed through our VPC peering connection (pcx-075a4043a28c62169), allowing our EC2 instances to communicate securely using their private IP addresses instead of the public internet.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_7316a13d)

## Connectivity troubleshooting

I received ping replies from Instance 2's private IP address! This means our VPC peering connection and direct routing rules are successfully working, allowing our EC2 instances to communicate privately and securely across our separate virtual networks without needing the public internet.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_4ec7821f)

## Analyzing flow logs

Flow logs tell us about detailed network metadata, capturing the source and destination IP addresses, source and destination port numbers, protocol types, packet counts, transferred byte volumes, start and end timestamps, and whether traffic was permitted (ACCEPT) or blocked (REJECT) by security groups and network access control lists (NACLs).

For example, the flow log I've captured (2 844347864508 eni-049be7c49b59686f5 217.146.80.99 10.1.13.67 27606 37777 6 1 40 1791769157 1791769179 REJECT OK) shows an incoming network connection attempt from external source IP 217.146.80.99 targeting the private IP 10.1.13.67 on network interface eni-049be7c49b59686f5 over TCP port 37777, which was blocked and dropped with a final status of REJECT because it violated the security settings and access control rules of the virtual private cloud.

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_d116818e)

## Logs Insights

Logs Insights is an integrated query and analysis feature in Amazon CloudWatch that allows you to interactively search, filter, and analyze log data using a dedicated query language to quickly troubleshoot operational issues and spot traffic patterns.


I ran the query stats sum(bytes) as TotalBytes by srcAddr, dstAddr | sort TotalBytes desc | limit 10. This query analyzes the flow logs by grouping the recorded traffic by source and destination IP addresses, summing up the transferred bytes for each pair, sorting them in descending order to find the heaviest data exchanges, and limiting the output to the top 10 results. (Note: Because I am using the updated Amazon CloudWatch Log Analytics experience, the interface layout differs slightly from older tutorial guides, but the Logs Insights query logic and results remain the same.)

![Image](https://nextwork.ai/overjoyed_magenta_proud_yak/uploads/6954a600-a89e-56b0-886b-7054da118a3b_3e1e79a1)

---

*Built with [NextWork](https://nextwork.ai) - [View this project](https://nextwork.ai/projects/6954a600-a89e-56b0-886b-7054da118a3b)*
