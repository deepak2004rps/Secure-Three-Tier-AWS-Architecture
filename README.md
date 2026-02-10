# Secure-Three-Tier-AWS-Architecture

A comprehensive implementation of a highly secure, production-grade AWS environment. This project focuses on **network isolation**, **the principle of least privilege**, and **manual NAT routing** to ensure that sensitive application and database layers are never exposed to the internet.



---

##  Architecture Overview
The infrastructure is built within a custom VPC ($10.0.0.0/16$) and follows a strict three-tier separation of concerns:

### 1. Public Tier (Management & Egress)
* **Bastion Host:** Serves as the single, hardened entry point for administrative SSH access.
* **NAT Instance:** A manually configured EC2 instance (Amazon Linux 2023) providing outbound internet access for private resources. I intentionally used a NAT Instance instead of a NAT Gateway to gain deeper visibility into packet forwarding and routing.
* **Web Server:** A public-facing Apache server used to demonstrate and validate initial entry-point connectivity.

### 2. Private Application Tier
* **App Server:** Isolated EC2 instance with no public IP and no direct route to the Internet Gateway, reachable only via the Bastion Host.

### 3. Private Database Tier
* **RDS MySQL:** Deployed in private subnets across multiple Availability Zones (AZs) to meet high availability and security requirements.

---

## Security Principles Implemented

* **Defense in Depth:** Multiple layers of defensive controls, including custom VPC segmentation, stateful Security Groups, and stateless NACLs.
* **Principle of Least Privilege:**
    * **DB Security Group:** Only allows MySQL (3306) traffic originating from the App Security Group.
    * **App Security Group:** Only allows SSH (22) from the Bastion Security Group.
    * **Bastion Security Group:** Restrictive SSH access limited strictly to my specific Administrator IP address.
* **Zero-Trust Access Pattern:** Implementation of **SSH Agent Forwarding (`ssh -A`)** to securely access private instances without ever storing private keys on intermediate servers.
* **Security Observability:** **VPC Flow Logs** enabled to capture and audit all `ACCEPT` and `REJECT` traffic, providing a complete audit trail in CloudWatch.

---

## Technical Deep-Dive: Manual NAT Configuration
A key highlight of this project is the manual configuration of the NAT Instance to handle packet routing at the OS level.



### Implementation Steps:
1.  **Platform Level:** Disabled **Source/Destination Check** on the NAT instance to allow it to forward traffic not destined for its own ENI.
2.  **Network Level:** Updated the Private Route Table to direct all outbound traffic ($0.0.0.0/0$) to the NAT Instance.
3.  **OS Kernel Level:** Enabled IP forwarding within the Amazon Linux 2023 kernel
    ```bash
    sudo sysctl -w net.ipv4.ip_forward=1
    ```
4.  **Firewall Level:** Configured `iptables` for **NAT Masquerading** and authorized the `FORWARD` chain to route traffic from the private CIDR to the internet:
    ```bash
    sudo iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
    ```

---

## Deployment & Security Validation
* **Inbound Connectivity:** Confirmed Apache web server accessibility via public IP.
* **Network Isolation:** Verified that App and DB servers have no public IPs and are unreachable from the internet.
* **Administrative Path:** Successfully performed a "jump" from the Bastion Host to the App Server using forwarded identities.
* **Outbound Connectivity:** Validated that the App Server could perform `ping` and `curl` requests to the internet only after NAT configuration was completed.
* **Database Integration:** Successfully connected to the RDS instance from the App Server to perform DDL and DML operations.

---

## Final Summary
This project demonstrates proficiency in secure AWS networking, Linux systems administration, and real-world cloud troubleshooting. By building the NAT and routing layers manually, I gained a comprehensive understanding of how data flows securely through a cloud environment.
