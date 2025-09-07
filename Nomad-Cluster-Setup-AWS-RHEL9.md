````markdown
# Nomad Cluster Setup on AWS

This documentation describes the process of setting up a HashiCorp Nomad cluster on AWS, including required steps, configuration, and screenshots.

---

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Step 1: AWS Environment Preparation](#step-1-aws-environment-preparation)
- [Step 2: Security Groups and IAM Roles](#step-2-security-groups-and-iam-roles)
- [Step 3: EC2 Instance Launch](#step-3-ec2-instance-launch)
- [Step 4: Nomad Installation](#step-4-nomad-installation)
- [Step 5: Cluster Configuration](#step-5-cluster-configuration)
- [Step 6: Starting Nomad Agents](#step-6-starting-nomad-agents)
- [Step 7: Verifying Cluster Status](#step-7-verifying-cluster-status)
- [Troubleshooting](#troubleshooting)
- [References](#references)
- [Screenshots](#screenshots)

---

## Introduction

[Nomad](https://www.nomadproject.io/) is a flexible scheduler for running applications across on-prem and cloud infrastructure. It supports containers, VMs, and standalone binaries.  
This guide describes how to deploy a **Nomad cluster on AWS** using 4 RHEL9 EC2 instances.

---

## Prerequisites

- AWS account
- CLI tools: **AWS CLI, SSH**
- Basic understanding of EC2, Networking, and IAM
- 4 EC2 VMs (RHEL9)

---

## Architecture Overview

⚖️ **General Rule for Nomad Cluster Nodes**

- **Servers** → Handle scheduling, Raft consensus, and cluster state.  
- **Clients** → Run the actual workloads (containers, applications, jobs).  

👉 Production: 3 or 5 servers (for Raft quorum) + remaining nodes as clients.  

For this setup: **3 servers + 1 client**.

### Recommended AWS EC2 Configuration

**OS**  
- Amazon Machine Image (AMI): RHEL 9 (x86_64)  

**Servers (3x)**  
- Instance type: `t3.medium` (2 vCPUs, 4 GB RAM)  
- Storage: 20 GB GP3 EBS  
- Networking: Private IPs enabled  

**Client (1x)**  
- Instance type: `t3.medium` (min), `t3.large` (better for workloads)  
- Storage: 30–50 GB GP3 EBS  

**Security Groups**  
Open required ports:
- 4646 → HTTP API / UI  
- 4647 → RPC  
- 4648 → Serf LAN gossip  
- 22 → SSH  

Restrict `4646` to your IP for security.

---

## Step 1: AWS Environment Preparation

- Create **VPC**  
- Configure **Subnets**  
- Create **Key pairs**  

---

## Step 2: Security Groups and IAM Roles

- Create security groups for Nomad communication  
- Attach IAM roles with necessary EC2 permissions  

---

## Step 3: EC2 Instance Launch

- Launch **4 instances** (3 servers + 1 client)  
- Place in the same **VPC + subnet**  
- Enable **Private DNS resolution**  

---

## Step 4: Nomad Installation

On each node:

```bash
sudo yum install curl wget unzip -y
curl -fsSL https://releases.hashicorp.com/nomad/1.8.0/nomad_1.8.0_linux_amd64.zip -o nomad.zip
unzip nomad.zip
sudo mv nomad /usr/local/bin/
nomad version
````

Output:

```
Nomad v1.8.0
BuildDate 2024-05-28T17:38:17Z
Revision 28b82e4b2259fae5a62e2ed47395334bea5a24c4
```

Create system user & directories:

```bash
sudo useradd --system --home /etc/nomad.d --shell /bin/false nomad
sudo chown root:root /usr/local/bin/nomad
sudo chmod 755 /usr/local/bin/nomad
sudo mkdir /etc/nomad.d
sudo chown -R nomad:nomad /etc/nomad.d
sudo chmod 750 /etc/nomad.d
```

![Nomad Installation Screenshot](https://github.com/user-attachments/assets/5e8ff4a7-1189-4a20-a9af-d8ff5e5c06bc)

---

## Step 5: Cluster Configuration

### Server 1 (`/etc/nomad.d/server.hcl`)

```hcl
data_dir = "/opt/nomad"
bind_addr = "0.0.0.0"

server {
  enabled = true
  bootstrap_expect = 3
  server_join {
    retry_join = ["172.31.35.78","172.31.40.224","172.31.44.65"]
  }
}

advertise {
  http = "172.31.35.78:4646"
  rpc  = "172.31.35.78:4647"
  serf = "172.31.35.78:4648"
}
```

### Server 2 (`/etc/nomad.d/server.hcl`)

```hcl
data_dir = "/opt/nomad"
bind_addr = "0.0.0.0"

server {
  enabled = true
  bootstrap_expect = 3
  server_join {
    retry_join = ["172.31.35.78","172.31.40.224","172.31.44.65"]
  }
}

advertise {
  http = "172.31.40.224:4646"
  rpc  = "172.31.40.224:4647"
  serf = "172.31.40.224:4648"
}
```

### Server 3 (`/etc/nomad.d/server.hcl`)

```hcl
data_dir = "/opt/nomad"
bind_addr = "0.0.0.0"

server {
  enabled = true
  bootstrap_expect = 3
  server_join {
    retry_join = ["172.31.35.78","172.31.40.224","172.31.44.65"]
  }
}

advertise {
  http = "172.31.44.65:4646"
  rpc  = "172.31.44.65:4647"
  serf = "172.31.44.65:4648"
}
```

### Client (`/etc/nomad.d/client.hcl`)

```hcl
data_dir = "/opt/nomad"
bind_addr = "0.0.0.0"

client {
  enabled = true
  servers = ["172.31.35.78:4647", "172.31.40.224:4647", "172.31.44.65:4647"]
}

advertise {
  http = "172.31.32.219:4646"
  rpc  = "172.31.32.219:4647"
  serf = "172.31.32.219:4648"
}
```

### Systemd Service (`/etc/systemd/system/nomad.service`)

```ini
[Unit]
Description=Nomad
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/bin/nomad agent -config=/etc/nomad.d
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

---

## Step 6: Starting Nomad Agents

```bash
sudo setenforce 0
sudo systemctl daemon-reexec
sudo systemctl enable nomad
sudo systemctl start nomad
sudo systemctl status nomad
```

---

## Step 7: Verifying Cluster Status

Check members:

```bash
nomad server members
nomad node status
```

Web UI Access:

* [http://52.211.218.44:4646](http://52.211.218.44:4646)
* `http://<SERVER1_PUBLIC_IP>:4646`

![Nomad Web UI](https://github.com/user-attachments/assets/4ad89648-16e5-414d-9cf6-8474f2780cce)

---

## Troubleshooting

* **Ports not open** → Check security groups
* **Cluster not forming** → Verify private IPs in `server.hcl`
* **Nomad service fails** → Check logs:

  ```bash
  journalctl -u nomad -f
  ```

---

## References

* [Nomad Official Documentation](https://developer.hashicorp.com/nomad/docs)
* [HashiCorp Learn - Nomad](https://learn.hashicorp.com/nomad)

---

## Screenshots
<img width="1057" height="618" alt="image" src="https://github.com/user-attachments/assets/1c8cf4ed-c766-43f7-8205-9b17036fe9de" />
<img width="1502" height="535" alt="image" src="https://github.com/user-attachments/assets/d3a4380f-f0af-4610-b6c8-db0b9a15bad9" />
<img width="1270" height="601" alt="image" src="https://github.com/user-attachments/assets/761985e2-ed43-4018-a6ef-a95f6135fa51" />
<img width="1142" height="527" alt="image" src="https://github.com/user-attachments/assets/42fdd2a1-5f81-4a45-a66a-18fc4d678e21" />

<img width="1418" height="705" alt="image" src="https://github.com/user-attachments/assets/84e2e83c-c218-4579-8dad-6cdb6214e68d" />
<img width="1682" height="297" alt="image" src="https://github.com/user-attachments/assets/4e384aae-b40c-41ba-9bdc-81781e67056b" />
<img width="1917" height="762" alt="image" src="https://github.com/user-attachments/assets/6d66398c-d2c4-42aa-9502-23d155daecfc" />



---

```

Do you want me to also **generate a `.md` file download** for you, or just keep it in this text format so you can copy-paste into GitHub?
```
