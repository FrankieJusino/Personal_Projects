# Grafana - Self-Hosted Service Report
## Project description:
- This project focuses on self-hosting [**Grafana**](https://grafana.com/), a popular open-source platform for data visualization and monitoring. Grafana was selected for its wide use in infrastructure, system health monitoring, and observability across enterprise and homelab environments. Grafana is a web-based analytics tool that allows users to visualize system metrics from a variety of data sources.
### What this document will cover:
- [Launching an EC2 instance to host Grafana](#aws-instance-setup)  
- [Creating a VPC, subnet, Network ACL/Security Group Rules and route table for network configuration](#aws-vpc-setup)  
- [Installing and configuring Grafana](#installing-grafana)  
- [Configuring security rules (Security Groups)](#security)  
- [Estimating costs for self-hosted deployment](#cost-estimates)  
- [Grafana software features](#software-features)
- [Implementing a backup and disaster recovery strategy](#backup-policy--disaster-recovery)  
- [Handling common troubleshooting scenarios](#common-troubleshooting)
  
---

## AWS VPC setup:
### VPC Block
- **CIDR Block:** `10.0.0.0/23`
- While Grafana is the only service hosted currently, this range allows for future growth such as adding [Prometheus](https://prometheus.io/) or other monitoring tools.
### Subnet block 
- **Grafana Subnet:** `10.0.0.0/24`
- More than sufficient for Grafana and allows for future tools.
### **Route Table for Grafana Subnet:** `grafana-server-rt`
0.0.0.0/0 -> `igw-0d0ad3aa824ab1af4` (for internet access)
Allows the Grafana instance to reach the public internet for tasks like installing software packages and downloading updates.
### Network ACL Rules:
- **Inbound Rules:**
  - Allow TCP 22 (SSH) and TCP 3000 (Grafana UI) from trusted IPs
  - Allow inbound access for administration (SSH) and Grafana usage (port 3000), limited to known IP addresses.
- **Outbound Rules:**
  - Allow all traffic
  - Outbound traffic is fully allowed to enable system updates, Grafana plugin installations, and external data source access if needed.
### Security Group Rules
- Inbound Rules:
  - TCP 22 from `My IP` (SSH access)
  - TCP 3000 from `My IP` (Grafana web access)
  - TCP 22 from `WSU IP` (SSH access) 
  - TCP 3000 from `WSU IP` (Grafana web access)
- **Outbound Rules:**
  - All traffic (default)
- Restrict access to only the ports required for Grafana and remote administration, and limit them to trusted IPs.

---

## AWS Instance Setup

### Instance Type
- Instance Type: `t2.micro`
- The `t2.micro` instance includes 1 vCPU and 1 GB of memory, which is sufficient for running a lightweight Grafana server with basic dashboards.

### AMI
- AMI Used: Ubuntu Server 22.04 LTS (64-bit x86)
- Ubuntu was chosen because it is ideal for hosting open-source services.
- **Note:** Grafana was installed natively via the APT package manager, not using Docker.

### Volume Size
- Volume Size: 20 GiB (GP3 SSD)
- The 20 GiB size provides a good amount of space for the Grafana installation, logs, and future dashboard configuration files or plugins.
  
---

## Cost estimates:
### Summary of cost estimates:
  - EC2 Instance`t2.micro` $0.0116 per hour = ~$8.35 per month
  - Elastic IP (EIP): $0.005 per hour = ~$3.60 per month
  - 20 GiB EBS: $0.08 per GiB per month = $1.60 per month
  - AMI: $0.05 per GiB per month
  - Total Estimated Cost: ~$13.6
  - **Total projected cost (from May 2, 2025, to May 2, 2026):** $165.87 (includes historical and forecasted usage)
    ![image](https://github.com/user-attachments/assets/b3d535a6-e62a-49ff-8b7c-b271d095814d)


  
---

## Installing Grafana:
### Documentation Reference
Installing Grafana on Ubuntu:
- [Grafana OSS Installation Guide (Debian/Ubuntu)](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)
### Summary of Installation Process:
1. Connect to the EC2 instance via SSH
2. Add the Grafana GPG key and repository:
    ```bash
    sudo mkdir -p /etc/apt/keyrings
    curl -fsSL https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/grafana.gpg

    echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
    sudo tee /etc/apt/sources.list.d/grafana.list > /dev/null
    ```
3. Update and install Grafana:
    ```bash
    sudo apt update
    sudo apt install grafana
    ```
4. Access Grafana from a browser **http://YourElastic-IP:3000**:
  - default login: admin / admin
  - Reset password on first login
### Screenshot of software operating on instance
  ![image](https://github.com/user-attachments/assets/5a263caf-5496-4b92-8271-d43f24c7b029)

---

## Security:
### How Server Access Is Being Restricted Depending on Service
Only specific ports and IP addresses are allowed to interact with the server:
  - **Port 22 (SSH):** Only accessible from your personal IP address(and wsu)
  - **Port 3000 (Grafana UI):** Also limited to your personal IP to prevent public exposure(and wsu)
### Controlling Remote Server Administration vs Using the Application
- **Remote Server Administration (SSH on Port 22):**
  - Only the EC2 owner has the private key required to access the server via SSH.
  - Security Group rules allow SSH only from specific IP addresses.
  - ![image](https://github.com/user-attachments/assets/f456f92b-00bc-4266-b996-20de27530288)
- **Grafana Application Access (Port 3000):**
  - Grafana’s web interface is accessible only from the same trusted IP.
  - Login is protected with a username/password combination.
  - Upon first login, the default credentials were reset.
  - ![image](https://github.com/user-attachments/assets/84f20466-8422-4c67-8d52-a4513ce1ba0b)


## Software features:
I set up Grafana with Prometheus to monitor a Windows EC2 server using [Windows Exporter](https://github.com/prometheus-community/windows_exporter).
### Prometheus Data Source
- Grafana connects to Prometheus at `http://localhost:9090` to pull data.
- Prometheus collects information from a Windows EC2 instance using the Windows Exporter tool.
### Windows Monitoring Dashboard
- A prebuilt dashboard (ID: 14649) was imported.
- - All data updates automatically every 15 seconds.
- The dashboard automatically shows important stats like:
  - CPU usage
  - Memory usage
  - Disk activity
  - System uptime
- ![image](https://github.com/user-attachments/assets/b9e2cbef-d959-45f9-b62a-468acee81079)
- ![image](https://github.com/user-attachments/assets/0901b956-5b3b-443b-96de-8d234a663c99)

  
---

## Backup policy / disaster recovery:
### Critical Files to Back Up
Grafana stores most of its configuration and data in a few key locations:
- `/var/lib/grafana/grafana.db` – main embedded SQLite database (stores dashboards, users, data sources)
- `/etc/grafana/grafana.ini` – configuration file with server settings and credentials
- `/var/lib/grafana/plugins/` – installed plugins
These files are essential for a complete restore of a Grafana setup.
### Backup Strategy:
Grafana uses SQLite by default, meaning most data is in one file (`grafana.db`), making backups straightforward.
- Backups should be made daily using a simple script that compresses and copies the critical files.
Daily backup routine:
- Stop the Grafana service briefly.
- Compress the following files and directories into a single archive:
  - `/var/lib/grafana/grafana.db`
  - `/etc/grafana/grafana.ini`
  - `/var/lib/grafana/plugins/` 
- Upload the archive to a S3 bucket
### 3-2-1 Backup Strategy:
  - 3 copies: 
    - 1 live copy on the EC2 instance 
    - 1 backup copy stored locally on the instance
    - 1 remote backup uploaded to Amazon S3
  - 2 different media types: 
    - EC2’s EBS volume (block storage)
    - Amazon S3 (object storage)
  - 1 offsite copy: 
    - An S3 backup downloaded to a local device outside of AWS
### Full EC2 Backup Option
In addition to data backups, use:
- **EBS Snapshots** – backups of the storage volume
- **Amazon Machine Images (AMIs)** – full image of the instance, ideal for one-click restoration
### Estimated recovery time(in case of failure):
- From an AMI: 
  ~5 minutes  
  Launching a new EC2 instance from an AMI restores Grafana and the OS exactly as it was at the time of backup. 
- **From an EBS snapshot:**  
  ~5–10 minutes  
  Create a new volume from the snapshot, attach it to a new EC2 instance, and boot.   
- **Manual reinstall with file backup:**  
  ~15–30 minutes  
  Launch a new instance, install Grafana, and restore `grafana.db`, `grafana.ini`, and plugins.
  
---

## Common troubleshooting:
### Metrics or Dashboards Not Appearing (If Using Prometheus):
  - When using Prometheus as a data source, dashboard panels may appear blank if:
  - Prometheus isn't running
  - The data source URL is incorrect
Targets aren’t properly defined in the Prometheus config:
  - Confirm Prometheus is running on the correct port (:9090)
  - Verify Grafana’s data source points to http://localhost:9090 or the correct IP
  - Check Prometheus targets at http://<prometheus-ip>:9090/targets
