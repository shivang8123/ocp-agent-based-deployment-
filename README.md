
# 🚀 OpenShift Agent-Based Cluster Installation Guide

> **OCP 4.19 | Agent-Based Installer | 3-Master Compact Cluster**
> A complete, step-by-step guide to deploying an OpenShift cluster using the Agent-Based Installer method — no PXE, no DHCP infrastructure required.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Node & IP Layout](#node--ip-layout)
- [Step 1 – Verify Hostname](#step-1--verify-hostname)
- [Step 2 – Install & Configure HAProxy](#step-2--install--configure-haproxy)
- [Step 3 – Configure Firewall](#step-3--configure-firewall)
- [Step 4 – SELinux Configuration](#step-4--selinux-configuration)
- [Step 5 – Install HTTP Server](#step-5--install-http-server)
- [Step 6 – Install & Configure DNS (BIND)](#step-6--install--configure-dns-bind)
- [Step 7 – Create DNS Zone Files](#step-7--create-dns-zone-files)
- [Step 8 – Fix Permissions & Start DNS](#step-8--fix-permissions--start-dns)
- [Step 9 – Validate DNS](#step-9--validate-dns)
- [Step 10 – Download OpenShift Binaries](#step-10--download-openshift-binaries)
- [Step 11 – Create Install Directory & SSH Key](#step-11--create-install-directory--ssh-key)
- [Step 12 – Create install-config.yaml](#step-12--create-install-configyaml)
- [Step 13 – Create agent-config.yaml](#step-13--create-agent-configyaml)
- [Step 14 – Install nmstate & Generate ISO](#step-14--install-nmstate--generate-iso)
- [Step 15 – Host ISO & Boot Nodes](#step-15--host-iso--boot-nodes)
- [Step 16 – Monitor Installation](#step-16--monitor-installation)
- [Step 17 – Access the Cluster](#step-17--access-the-cluster)
- [Troubleshooting](#troubleshooting)
- [Key Files Reference](#key-files-reference)

---

## Architecture Overview

```
                         ┌─────────────────────────────────┐
                         │        Bastion Host              │
                         │   ocp-bastion-shivang            │
                         │   190.170.20.244                 │
                         │                                  │
                         │  ┌─────────┐  ┌──────────────┐  │
                         │  │ HAProxy │  │  BIND (DNS)  │  │
                         │  │ :6443   │  │  :53         │  │
                         │  │ :22623  │  └──────────────┘  │
                         │  │ :80/443 │  ┌──────────────┐  │
                         │  └────┬────┘  │  HTTPD       │  │
                         │       │       │  (ISO Host)  │  │
                         └───────┼───────┴──────────────┘──┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
   │  master-01      │  │  master-02      │  │  master-03      │
   │  190.170.20.181 │  │  190.170.20.182 │  │  190.170.20.183 │
   └─────────────────┘  └─────────────────┘  └─────────────────┘
```

**The bastion host runs three services:**
- **HAProxy** — Layer-4 load balancer for API, MCS, and Ingress traffic
- **BIND** — Authoritative DNS for the `lab.kuberox.net` domain
- **Apache HTTPD** — Serves the agent ISO over HTTP

---

## Prerequisites

| Requirement | Detail |
|---|---|
| OS (Bastion) | RHEL 8/9 or CentOS Stream |
| OpenShift Version | 4.19.20 |
| Internet Access | Bastion must reach `mirror.openshift.com` and `quay.io` |
| Pull Secret | Download from [Red Hat Console](https://console.redhat.com/openshift/install/pull-secret) |
| RAM per Master | Minimum 16 GB |
| CPU per Master | Minimum 4 vCPUs |
| Disk per Master | Minimum 120 GB |

---

## Node & IP Layout

| Role | Hostname | IP Address | MAC Address |
|---|---|---|---|
| Bastion / LB / DNS | `ocp-bastion-shivang.lab.kuberox.net` | `190.170.20.244` | — |
| Master 1 | `master-shivang-01.ocp.lab.kuberox.net` | `190.170.20.181` | `00:50:56:85:15:ef` |
| Master 2 | `master-shivang-02.ocp.lab.kuberox.net` | `190.170.20.182` | `00:50:56:85:ad:51` |
| Master 3 | `master-shivang-03.ocp.lab.kuberox.net` | `190.170.20.183` | `00:50:56:85:cd:34` |

---

## Step 1 – Verify Hostname

```bash
hostname
```

---

## Step 2 – Install & Configure HAProxy

```bash
yum install haproxy -y
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bkp
vi /etc/haproxy/haproxy.cfg
```

See the full config: [`configs/haproxy.cfg`](configs/haproxy.cfg)

**Port reference:**

| Service | Port |
|---|---|
| OCP API | `6443` |
| Machine Config Server | `22623` |
| Ingress HTTP | `80` |
| Ingress HTTPS | `443` |

```bash
# Validate config syntax
haproxy -c -f /etc/haproxy/haproxy.cfg

# Enable and start
systemctl enable --now haproxy
systemctl status haproxy
```

---

## Step 3 – Configure Firewall

```bash
# Check current rules
firewall-cmd --list-all

# Open required ports
firewall-cmd --add-port={80,443,6443,22623}/tcp --permanent
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --reload

# Verify
firewall-cmd --list-all
```

---

## Step 4 – SELinux Configuration

```bash
# Check current mode
getenforce

# Allow HAProxy to bind to OpenShift ports
semanage port -a -t http_port_t -p tcp 80
semanage port -a -t http_port_t -p tcp 443
semanage port -a -t http_port_t -p tcp 8080
semanage port -a -t http_port_t -p tcp 6443
semanage port -a -t http_port_t -p tcp 22623

# Verify labels
semanage port -l | grep http

# Restart HAProxy
systemctl restart haproxy
systemctl status haproxy
```

> ⚠️ **Do NOT disable SELinux permanently.** Use `semanage` to label ports correctly instead of running `setenforce 0` in production.

---

## Step 5 – Install HTTP Server

```bash
yum install httpd -y
systemctl enable --now httpd
# Edit port if needed (e.g. change to 8080)
vi /etc/httpd/conf/httpd.conf
systemctl restart httpd
```

---

## Step 6 – Install & Configure DNS (BIND)

```bash
yum install bind -y
vi /etc/named.conf
```

See full config: [`configs/named.conf`](configs/named.conf)

**Key `named.conf` settings:**
```
listen-on port 53 { 190.170.20.244; };
allow-query     { any; };
```

```bash
# Add zone definitions
vi /etc/named.rfc1912.zones
```

See: [`configs/named.rfc1912.zones`](configs/named.rfc1912.zones)

---

## Step 7 – Create DNS Zone Files

```bash
cd /var/named/
cp -p named.localhost local.for
cp named.loopback local.rev
```

Edit the forward zone:
```bash
vi local.for
```

See full zone: [`configs/local.for`](configs/local.for)

Edit the reverse zone:
```bash
vi local.rev
```

See full zone: [`configs/local.rev`](configs/local.rev)

> ✅ **Required DNS records for OCP:**
> - `api.ocp.lab.kuberox.net` → bastion IP
> - `api-int.ocp.lab.kuberox.net` → bastion IP
> - `*.apps.ocp.lab.kuberox.net` → bastion IP (or worker IPs)
> - All master node A records + PTR records

---

## Step 8 – Fix Permissions & Start DNS

```bash
chown root:named /var/named/local.rev
chmod 640 /var/named/local.rev

systemctl enable --now named
systemctl restart named
systemctl status named

# Open DNS port in firewall
firewall-cmd --add-service=dns --permanent
firewall-cmd --reload
```

---

## Step 9 – Validate DNS

```bash
# Forward lookups
nslookup ocp-bastion-shivang.lab.kuberox.net
nslookup master-shivang-01.ocp.lab.kuberox.net
nslookup api.ocp.lab.kuberox.net

# Reverse lookups
nslookup 190.170.20.181
nslookup 190.170.20.182
nslookup 190.170.20.183
nslookup 190.170.20.244
```

> ⚠️ **All lookups must pass before proceeding.** The installer will fail with DNS errors if records are missing.

Set the bastion to use its own DNS:
```bash
nmtui   # Change DNS server to 127.0.0.1 or 190.170.20.244
```

---

## Step 10 – Download OpenShift Binaries

```bash
mkdir ~/binaries && cd ~/binaries

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.19.20/openshift-client-linux-4.19.20.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.19.20/openshift-install-linux-4.19.20.tar.gz

tar -xvzf openshift-client-linux-4.19.20.tar.gz
tar -xvzf openshift-install-linux-4.19.20.tar.gz

cp kubectl oc openshift-install /usr/local/bin

# Verify
oc version
kubectl version
openshift-install version
```

---

## Step 11 – Create Install Directory & SSH Key

```bash
mkdir ~/agent-4.19.20
cd ~/agent-4.19.20

# Generate SSH key (used for node access post-install)
ssh-keygen

# Copy public key — paste into install-config.yaml
cat ~/.ssh/id_rsa.pub
```

---

## Step 12 – Create install-config.yaml

```bash
vi ~/agent-4.19.20/install-config.yaml
```

See full config: [`configs/install-config.yaml`](configs/install-config.yaml)

```yaml
apiVersion: v1
baseDomain: lab.kuberox.net
metadata:
  name: ocp
compute:
- architecture: amd64
  name: worker
  replicas: 0
  hyperthreading: Enabled
controlPlane:
  architecture: amd64
  name: master
  replicas: 3
  hyperthreading: Enabled
networking:
  networkType: OVNKubernetes
  clusterNetwork:
    - cidr: 10.128.0.0/14
      hostPrefix: 23
  serviceNetwork:
    - 172.30.0.0/16
  machineNetwork:
    - cidr: 190.170.20.0/23
platform:
  none: {}
fips: false
pullSecret: '<YOUR_PULL_SECRET>'
sshKey: '<YOUR_SSH_PUBLIC_KEY>'
```

```bash
# IMPORTANT: Backup before running the installer (installer deletes this file)
cp install-config.yaml install-config.yaml.bkp
```

---

## Step 13 – Create agent-config.yaml

```bash
vi ~/agent-4.19.20/agent-config.yaml
```

See full config: [`configs/agent-config.yaml`](configs/agent-config.yaml)

```yaml
apiVersion: v1alpha1
kind: AgentConfig
metadata:
  name: ocp
rendezvousIP: 190.170.20.181
hosts:
  - hostname: master-shivang-01.ocp.lab.kuberox.net
    role: master
    interfaces:
      - name: ens192
        macAddress: 00:50:56:85:15:ef
    networkConfig:
      interfaces:
        - name: ens192
          type: ethernet
          state: up
          ipv4:
            enabled: true
            address:
              - ip: 190.170.20.181
                prefix-length: 22
            dhcp: false
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 190.170.20.1
            next-hop-interface: ens192
            table-id: 254
      dns-resolver:
        config:
          server:
            - 190.170.20.244
  # ... repeat for master-02 and master-03
```

```bash
cp agent-config.yaml agent-config.yaml.bkp
```

---

## Step 14 – Install nmstate & Generate ISO

```bash
# nmstate is required to process NMState network configs in agent-config.yaml
yum install nmstate -y

# Generate the bootable agent ISO
cd ~/agent-4.19.20/
openshift-install --dir . agent create image
```

Output:
```
INFO Agent ISO created at agent.x86_64.iso
```

---

## Step 15 – Host ISO & Boot Nodes

```bash
# Copy ISO to HTTPD directory
mkdir -p /var/www/html/agent
cp ~/agent-4.19.20/agent.x86_64.iso /var/www/html/agent/

# Verify it's accessible
curl -I http://190.170.20.244:8080/agent/agent.x86_64.iso
```

**Boot each master node** from `agent.x86_64.iso` using:
- VMware vSphere datastore ISO mount
- iDRAC / iLO virtual media
- USB boot media

Each node will automatically:
1. Apply its NMState network config (from `agent-config.yaml`)
2. Register with the rendezvous IP (`190.170.20.181`)
3. Download and install RHCOS
4. Join the cluster

---

## Step 16 – Monitor Installation

```bash
cd ~/agent-4.19.20/

# Monitor bootstrap phase (~15–20 min)
openshift-install --dir . agent wait-for bootstrap-complete --log-level=info

# Monitor full install (~30–60 min)
openshift-install --dir . agent wait-for install-complete --log-level=info
```

In a second terminal, watch cluster progress:
```bash
export KUBECONFIG=~/agent-4.19.20/auth/kubeconfig
watch "oc get co,no,csr"
```

SSH into a master node to check logs:
```bash
ssh core@190.170.20.181
journalctl -f
```

---

## Step 17 – Access the Cluster

```bash
export KUBECONFIG=/root/agent-4.19.20/auth/kubeconfig

# Check nodes
oc get nodes

# Check cluster operators
oc get co

# CLI login
oc login https://api.ocp.lab.kuberox.net:6443
```

---

## Troubleshooting

| Symptom | Command to Run |
|---|---|
| HAProxy not starting | `systemctl status haproxy` / `journalctl -u haproxy -xe` |
| DNS not resolving | `systemctl status named` / `journalctl -u named -xe` |
| SELinux blocking HAProxy | `semanage port -l \| grep http` |
| Firewall blocking traffic | `firewall-cmd --list-all` |
| ISO generation fails | Check `openshift-install` version matches pull secret entitlement |
| Nodes not registering | Verify rendezvousIP is reachable from all nodes |
| CSRs pending | `oc get csr` then `oc adm certificate approve <name>` |
| Cluster operators degraded | `oc get co` / `oc describe co <name>` |

---

## Key Files Reference

| File / Path | Purpose |
|---|---|
| `/etc/haproxy/haproxy.cfg` | HAProxy load balancer config |
| `/etc/named.conf` | BIND main config |
| `/etc/named.rfc1912.zones` | DNS zone definitions |
| `/var/named/local.for` | Forward DNS zone |
| `/var/named/local.rev` | Reverse DNS zone |
| `~/agent-4.19.20/install-config.yaml` | Cluster topology |
| `~/agent-4.19.20/agent-config.yaml` | Node NIC/IP mapping |
| `~/agent-4.19.20/agent.x86_64.iso` | Bootable agent ISO |
| `/var/www/html/agent/` | HTTP directory for ISO |
| `~/agent-4.19.20/auth/kubeconfig` | Cluster kubeconfig |

---

## 📁 Repo Structure

```
ocp-agent-cluster/
├── README.md                    ← This guide
├── configs/
│   ├── haproxy.cfg              ← HAProxy configuration
│   ├── named.conf               ← BIND named.conf
│   ├── named.rfc1912.zones      ← Zone definitions
│   ├── local.for                ← Forward DNS zone
│   ├── local.rev                ← Reverse DNS zone
│   ├── install-config.yaml      ← OCP install config (template)
│   └── agent-config.yaml        ← Agent node config (template)
└── docs/
    └── OCP-Agent-Based-Cluster-Guide.docx   ← Full offline guide
```

---

## 📝 Notes

- `install-config.yaml` and `agent-config.yaml` are **consumed and deleted** by `openshift-install` during ISO generation. Always back them up first.
- The `rendezvousIP` in `agent-config.yaml` must be the IP of one of the master nodes (typically master-01). All other nodes connect to it during bootstrap.
- Replace all placeholder values (pull secret, SSH key, MAC addresses, IPs) with your actual environment values before running.

---

### *Built by: Shivang Bhardwaj | Environment: lab.kuberox.net | OCP: 4.19.20*
