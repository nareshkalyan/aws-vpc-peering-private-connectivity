\# AWS VPC Peering: Step-by-Step Implementation Guide



\## 1. Purpose



This guide documents the implementation of AWS VPC Peering between two Virtual Private Clouds (VPCs). The goal is to enable private IPv4 communication between application servers in separate VPCs while keeping both application servers free of public IPv4 addresses.



The environment uses a bastion host in a public subnet for SSH administration and security groups to restrict inbound traffic.



\## 2. AWS Environment



\- \*\*Cloud provider:\*\* Amazon Web Services (AWS)

\- \*\*Region:\*\* Asia Pacific (Hyderabad), `ap-south-2`

\- \*\*VPC A CIDR:\*\* `10.10.0.0/16`

\- \*\*VPC B CIDR:\*\* `10.20.0.0/16`

\- \*\*Bastion private IP:\*\* `10.10.1.131`

\- \*\*App-A private IP:\*\* `10.10.2.10`

\- \*\*App-B private IP:\*\* `10.20.1.10`



The two VPC CIDR ranges do not overlap, which allows them to be connected through VPC Peering.



\## 3. Create the VPCs



1\. Open the AWS Management Console and select the Hyderabad region (`ap-south-2`).

2\. Navigate to \*\*VPC → Your VPCs\*\*.

3\. Create VPC A with IPv4 CIDR `10.10.0.0/16`.

4\. Create VPC B with IPv4 CIDR `10.20.0.0/16`.

5\. Add project identification tags where applicable:

&#x20;  - `Project = EWB-VPC-Peering`

&#x20;  - `Student = Naresh Kalyan`

6\. Verify both VPCs appear in the VPC list with the intended CIDR blocks.



\*\*Evidence:\*\* `evidence/Aws/VPC-A.png` and `evidence/Aws/VPC-B.png`



\## 4. Create the Subnets



Create the following subnets in their respective VPCs.



| Subnet | VPC | IPv4 CIDR | Purpose |

|---|---|---|---|

| `ewb-a-public` | VPC A | `10.10.1.0/24` | Bastion host |

| `ewb-a-private` | VPC A | `10.10.2.0/24` | App-A |

| `ewb-b-private` | VPC B | `10.20.1.0/24` | App-B |



Steps:



1\. Navigate to \*\*VPC → Subnets → Create subnet\*\*.

2\. Select the correct VPC for each subnet.

3\. Enter the subnet name and IPv4 CIDR from the table.

4\. Select an Availability Zone supported in the selected region.

5\. Create each subnet and verify its VPC association and CIDR.



The public subnet is used for the bastion host. The application subnets are private and do not assign public IPv4 addresses to the application instances.



\*\*Evidence:\*\* `evidence/Aws/Public-Subnet-A.png`, `evidence/Aws/Private-Subnet-A.png`, and `evidence/Aws/Private-Subnet-B.png`



\## 5. Create and Attach the Internet Gateway



1\. Navigate to \*\*VPC → Internet gateways\*\*.

2\. Create an Internet Gateway named `ewb-igw-a`.

3\. Select the gateway and choose \*\*Attach to a VPC\*\*.

4\. Attach it to VPC A.

5\. Verify that the gateway is attached to VPC A.



The Internet Gateway provides the internet path for the public subnet when the appropriate route and security rules are configured. It is not attached to VPC B.



\*\*Evidence:\*\* `evidence/Aws/Internetgateway.png`



\## 6. Configure Route Tables



Create and configure these route tables:



\- `ewb-rt-a-public`

\- `ewb-rt-a-private`

\- `ewb-rt-b-private`



\### 6.1 Public route table for VPC A



Configure the routes:



| Destination | Target |

|---|---|

| `10.10.0.0/16` | Local |

| `0.0.0.0/0` | Internet Gateway `ewb-igw-a` |

| `10.20.0.0/16` | VPC peering connection |



Associate this route table with `ewb-a-public`.



The default route directs internet-bound traffic to the Internet Gateway. The peering route directs traffic for VPC B through the peering connection.



\### 6.2 Private route table for VPC A



Configure the routes:



| Destination | Target |

|---|---|

| `10.10.0.0/16` | Local |

| `10.20.0.0/16` | VPC peering connection |



Associate this route table with `ewb-a-private`.



Do not add an internet default route to this table for this project design.



\### 6.3 Private route table for VPC B



Configure the routes:



| Destination | Target |

|---|---|

| `10.20.0.0/16` | Local |

| `10.10.0.0/16` | VPC peering connection |



Associate this route table with `ewb-b-private`.



VPC peering requires routes on both sides for the intended traffic. The VPC peering connection alone does not automatically add these routes.



\*\*Evidence:\*\*



\- `evidence/Aws/Routetable-Public-A.png`

\- `evidence/Aws/Routetable-Private-A.png`

\- `evidence/Aws/Routetable-Private-B.png`



\## 7. Create the VPC Peering Connection



1\. Navigate to \*\*VPC → Peering connections\*\*.

2\. Choose \*\*Create peering connection\*\*.

3\. Name the connection `ewb-a-b-peer`.

4\. Select VPC A as the requester VPC.

5\. Select VPC B as the accepter VPC.

6\. Create the peering request.

7\. Accept the request.

8\. Verify that the connection status is \*\*Active\*\*.

9\. Add the peering routes to the relevant route tables as described in Section 6.



Both VPCs are in the same AWS account and region for this implementation.



\*\*Evidence:\*\* `evidence/Aws/VPC-Peering-Connection.png`



\## 8. Configure Security Groups



Create the following security groups and associate them with the appropriate EC2 instances.



\### 8.1 Bastion security group: `ewb-sg-bastion`



Configure the inbound rule:



| Type | Protocol | Port | Source |

|---|---|---|---|

| SSH | TCP | 22 | Administrator's current public IPv4 address with `/32` |



Restricting SSH to a specific administrator IP is safer than allowing SSH from all IPv4 addresses.



\### 8.2 App-A security group: `ewb-sg-app-a`



Configure the inbound rules:



| Type | Protocol/port | Source |

|---|---|---|

| SSH | TCP 22 | `10.10.1.131/32` |

| ICMP Echo Request | ICMP type 8, code 0 | `10.20.1.10/32` |



\### 8.3 App-B security group: `ewb-sg-app-b`



Configure the inbound rules:



| Type | Protocol/port | Source |

|---|---|---|

| SSH | TCP 22 | `10.10.1.131/32` |

| ICMP Echo Request | ICMP type 8, code 0 | `10.10.2.10/32` |



The application rules permit SSH from the bastion and ICMP Echo Requests from the other application server. Security groups are stateful, so return traffic for allowed connections is automatically permitted.



Review the actual rules in the AWS console before making changes. The evidence screenshots record the configuration used for the tests.



\*\*Evidence:\*\*



\- `evidence/Aws/Security-Group-Bastion.png`

\- `evidence/Aws/Security-Group-app-A.png`

\- `evidence/Aws/Security-Group-app-B.png`



\## 9. Launch the EC2 Instances



Launch three Amazon Linux 2023 EC2 instances using the `t3.micro` instance type and the `ewb-lab` key pair.



| Instance | Subnet | Private IPv4 | Public IPv4 |

|---|---|---|---|

| Bastion | `ewb-a-public` | `10.10.1.131` | Assigned |

| App-A | `ewb-a-private` | `10.10.2.10` | None |

| App-B | `ewb-b-private` | `10.20.1.10` | None |



For each instance:



1\. Choose the correct VPC and subnet.

2\. Select the appropriate security group.

3\. Use the `ewb-lab` key pair.

4\. Enable public IPv4 assignment only for the bastion host.

5\. Configure the application instances with their intended private addresses.

6\. Launch the instances and verify their running state and network details.



\*\*Evidence:\*\*



\- `evidence/Aws/EC2-Instance-Bastion.png`

\- `evidence/Aws/EC2-Instance-app-A.png`

\- `evidence/Aws/EC2-Instance-app-B.png`



\## 10. Configure SSH Access from Windows



Windows OpenSSH was configured to use SSH aliases and `ProxyJump` so the administrator could access the private instances through the bastion host.



The SSH configuration contains these aliases:



```powershell

ssh ewb-bastion

ssh ewb-app-a

ssh ewb-app-b

```



The private key remains on the administrator's Windows computer. It must not be uploaded to GitHub or copied into this project directory.



The bastion acts as the SSH jump host for the private application instances. App-A and App-B do not require public IPv4 addresses for this administration method.



\## 11. Test Private Connectivity



After configuring the routes and security groups, connect to each application instance through the bastion and test connectivity using private IPv4 addresses.



\### 11.1 App-A to App-B



Run on App-A:



```bash

ping -c 4 10.20.1.10

```



\*\*Observed result:\*\* Four packets received, zero packet loss.



\### 11.2 App-B to App-A



Run on App-B:



```bash

ping -c 4 10.10.2.10

```



\*\*Observed result:\*\* Four packets received, zero packet loss.



These tests demonstrate bidirectional private IPv4 connectivity between the application instances across the peering connection.



\*\*Evidence:\*\*



\- `evidence/Connectivity/01-app-a-to-app-b.png.png`

\- `evidence/Connectivity/02-app-b-to-app-a.png.png`



\## 12. Perform a Controlled Failure and Recovery Test



This test demonstrates the effect of an inbound security-group rule.



1\. Verify that App-A can ping App-B.

2\. Temporarily remove the App-B inbound ICMP Echo Request rule that permits traffic from `10.10.2.10/32`.

3\. Run the App-A-to-App-B ping test again.

4\. Confirm that the test fails while the rule is absent.

5\. Restore the inbound ICMP rule.

6\. Repeat the ping test.

7\. Confirm that connectivity is restored.



\*\*Observed results:\*\*



\- Before rule removal: successful ping.

\- During the controlled failure: four packets sent, zero received.

\- After rule restoration: four packets received, zero packet loss.



This demonstrates how security-group rules influence network reachability. The rule was restored after the test.



\*\*Evidence:\*\*



\- `evidence/Connectivity/03-controlled-failure.png.png`

\- `evidence/Connectivity/04-connectivity-restored.png.png`



\## 13. Troubleshooting Principles



If private connectivity fails, verify the following:



1\. The peering connection is active.

2\. The VPC CIDR ranges do not overlap.

3\. The source VPC route table has a route to the destination CIDR through the peering connection.

4\. The destination VPC route table has a return route.

5\. The relevant security groups permit the intended traffic.

6\. The instances use the expected private IP addresses and subnets.

7\. The operating system permits the test traffic.



Change one setting at a time and repeat the relevant test so the cause of a failure can be identified.



\## 14. Security and Cost Considerations



\- Do not publish private keys, AWS access keys, tokens, passwords, or credential files.

\- Restrict bastion SSH access to the administrator's current public IP.

\- Keep application instances without public IPv4 addresses.

\- Review screenshots for sensitive information before publishing.

\- Monitor AWS billing and remaining credits.

\- After collecting the required evidence and completing the report, carefully review the project resources before cleaning them up.

\- Do not delete resources that belong to other projects.



\## 15. Conclusion



The implementation demonstrates VPC Peering between two non-overlapping AWS VPCs, private communication between application instances, bastion-based SSH administration, route-table configuration, and security-group enforcement. Bidirectional connectivity was verified, and a controlled failure/recovery test demonstrated the effect of an ICMP security-group rule.



\## Project Information



\- \*\*Author:\*\* Naresh Kalyan

\- \*\*Project:\*\* EWB Cloud Labs Student Project

\- \*\*Region:\*\* Asia Pacific (Hyderabad), `ap-south-2`

\- \*\*Project period:\*\* October 2026



