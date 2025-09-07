
````markdown
# Nomad Cluster Setup ==== 7-9-2025

## Cluster Nodes Overview

| Node | Instance Type | vCPU | RAM | Disk | Role         |
| ---- | ------------- | ---- | --- | ---- | ------------ |
| VM1  | t3.medium     | 2    | 4GB | 20GB | Nomad Server |
| VM2  | t3.medium     | 2    | 4GB | 20GB | Nomad Server |
| VM3  | t3.medium     | 2    | 4GB | 20GB | Nomad Server |
| VM4  | t3.medium     | 2    | 4GB | 20GB | Nomad Client |

### Node IP Addresses

| Node           | Public IP      | Private IP     |
| -------------- | ------------- | -------------- |
| Nomad-Server1  | 52.211.218.44 | 172.31.35.78  |
| Nomad-Server2  | 3.250.28.156  | 172.31.40.224 |
| Nomad-Server3  | 176.34.155.39 | 172.31.44.65  |
| Nomad-Client1  | 3.252.121.92  | 172.31.32.219 |

---

## 1️⃣ Install Nomad on All EC2 Instances

```bash
sudo yum install curl wget unzip -y

curl -fsSL https://releases.hashicorp.com/nomad/1.8.0/nomad_1.8.0_linux_amd64.zip -o nomad.zip
unzip nomad.zip
sudo mv nomad /usr/local/bin/
nomad version
````

Expected output:

```
Nomad v1.8.0
BuildDate 2024-05-28T17:38:17Z
Revision 28b82e4b2259fae5a62e2ed47395334bea5a24c4
```

---

## 2️⃣ Create a Nomad System User

Run on each VM:

```bash
sudo useradd --system --home /etc/nomad.d --shell /bin/false nomad
id nomad
```

---

## 3️⃣ Set Ownership of Config Directory

```bash
sudo chown -R nomad:nomad /etc/nomad.d
sudo chmod 750 /etc/nomad.d
ls -ld /etc/nomad.d
ls -l /etc/nomad.d/
```

Expected output:

```
drwxr-x---. 2 nomad nomad 4096 Sep  7 06:00 /etc/nomad.d
-rw-r-----. 1 nomad nomad  300 Sep  7 06:01 server.hcl
```

---

## 4️⃣ Configure Server Nodes (VM1, VM2, VM3)

### Nomad-Server1 (`/etc/nomad.d/server.hcl`)

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

### Nomad-Server2 (`/etc/nomad.d/server.hcl`)

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

### Nomad-Server3 (`/etc/nomad.d/server.hcl`)

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

---

## 5️⃣ Configure Client Node (VM4)

### Nomad-Client1 (`/etc/nomad.d/client.hcl`)

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

### Docker Plugin (for Nomad Client)

```hcl
plugin "docker" {
  config {
    volumes {
      enabled = true
    }
  }
}
```

---

## 6️⃣ Configure Systemd Service

### `/etc/systemd/system/nomad.service`

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

## 7️⃣ Enable and Start Nomad

```bash
sudo setenforce 0
sudo systemctl daemon-reexec
sudo systemctl enable nomad
sudo systemctl start nomad
sudo systemctl status nomad
```

---

## 8️⃣ Install Docker on Client Node Only

```bash
sudo yum update -y
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable docker
sudo systemctl start docker
docker --version
```

---

## 9️⃣ Docker + Nomad Job (Local Image)

### Build Local Docker Image

```bash
sudo docker build -t local-nginx:latest /path/to/Dockerfile
sudo docker images | grep local-nginx
sudo docker run -p 8080:80 local-nginx:latest
```

### Add Nomad User to Docker Group

```bash
sudo usermod -aG docker nomad
sudo systemctl restart nomad
sudo -u nomad docker images | grep local-nginx
ls -l /var/run/docker.sock
```

### Example Nomad Job (`nomadnew.nomad`)

```hcl
job "nginx-local" {
  datacenters = ["dc1"]
  type        = "service"

  group "nginx" {
    count = 1

    network {
      port "http" {
        static = 8082
        to     = 80
      }
    }

    task "nginx" {
      driver = "docker"

      config {
        image      = "local-nginx:nomad"
        force_pull = false
        ports      = ["http"]
      }

      resources {
        cpu    = 500
        memory = 256
      }

      service {
        name     = "nginx"
        provider = "nomad"
        port     = "http"

        check {
          type     = "tcp"
          name     = "nginx-tcp-check"
          port     = "http"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

---

## 10️⃣ Jenkins Deployment Using Nomad (Local Image)

### Example Nomad Job (`jenkins.nomad`)

```hcl
job "jenkins-local" {
  datacenters = ["dc1"]
  type = "service"

  group "jenkins" {
    count = 1

    network {
      port "http"  { static = 8080 }
      port "agent" { static = 50000 }
    }

    task "jenkins" {
      driver = "docker"

      config {
        image      = "jenkins/jenkins:lts"
        force_pull = false
        volumes    = ["/var/jenkins_home:/var/jenkins_home"]
        ports      = ["http", "agent"]
      }

      resources {
        cpu    = 1000
        memory = 1024
      }

      service {
        provider = "nomad"
        name     = "jenkins"

        check {
          type     = "tcp"
          interval = "10s"
          timeout  = "2s"
        }
      }
    }
  }
}
```

---

## 11️⃣ Nomad Job Management Commands

```bash
nomad job status <job-name>
nomad alloc status <alloc-id>
nomad alloc logs <alloc-id> <task-name>
nomad job stop -purge <job-name>
```

This ensures proper deployment, logs checking, and cleanup.

```
```
<img width="1024" height="1536" alt="ChatGPT Image Sep 7, 2025, 06_37_05 PM" src="https://github.com/user-attachments/assets/93843fd1-a25f-421f-91d7-529b32d9110a" />

