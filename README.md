# **Cloudflare Zero Trust — Private VPC Access Project**

This project demonstrates how to build a secure, identity-aware private network using **Cloudflare Zero Trust**, **WARP**, **Cloudflare Connector**, and a **Vultr VPC-hosted Linux VM**.
The goal is to enable a laptop to privately access a VM using only its **VPC private IP (172.31.0.3)** — without exposing any public ports or relying on a static IP address.

This setup aligns with Zero Trust principles and does **not** require VPN servers, port forwarding, or public exposure.

---

## **📌 Architecture Overview**

This architecture links:

* **Laptop (Client)** → runs WARP, authenticated into Cloudflare Zero Trust
* **Cloudflare Zero Trust** → identity, routing, device policies
* **Cloudflare Connector** → outbound-only tunnel from VM to Cloudflare
* **Vultr VPC** → private network (172.31.0.0/24)
* **Ubuntu VM** → private IP: `172.31.0.3`

Flow:

<p align="center">
  <img src="Screenshot/Flow.png" width="800" alt="Architecture Flow" />
</p>

This enables **SSH over private IP**, even though:

* The VM has no inbound ports open
* Your laptop's IP changes (e.g., hotspot)
* ICMP (ping) is not proxied by Cloudflare Zero Trust

---

## **📌 Objectives**

* Deploy a Vultr VPC (`172.31.0.0/24`)
* Deploy a VM inside that VPC
* Install and configure Cloudflare WARP
* Configure Device Enrollment and Device Profiles
* Create and deploy a Cloudflare Connector on the VM
* Add private routing inside Cloudflare to the VPC
* Validate private connectivity (SSH via private IP)

---

# **1. VPC Creation**

1. Navigate to **Vultr → Network → VPC 2.0**

<p align="center">
  <img src="Screenshot/Create%20VPC.png" width="800" alt="Create VPC" />
</p>

2. Create new VPC:

* **Name:** `SOC-VPC`
* **Region:** Mumbai
* **Subnet:** `172.31.0.0/24`

<p align="center">
  <img src="Screenshot/VPC%20Conf.png" width="800" alt="VPC Configuration" />
</p>

3. Confirm VPC is active.

<p align="center">
  <img src="Screenshot/VPC%20Created.png" width="800" alt="VPC Created" />
</p>

---

# **2. VM Deployment**

1. Vultr → Deploy New Instance

<p align="center">
  <img src="Screenshot/Compute%20and%20Deploy%20Button.png" width="800" alt="Compute Deploy Button" />
</p>

<p align="center">
  <img src="Screenshot/Deploy%20new%20Server.png" width="800" alt="Deploy New Server" />
</p>

2. Select OS and size:

* **Cloud Compute**
* **Ubuntu 22.04 LTS**
* **1 vCPU / 1GB RAM / 25GB SSD**

3. Disable Auto Backup (optional):

<p align="center">
  <img src="Screenshot/Disable%20Auto%20backup.png" width="800" alt="Disable Auto Backup" />
</p>

4. Attach VM to the `SOC-VPC`:

<p align="center">
  <img src="Screenshot/VPC%20gateway%20conf.png" width="800" alt="Attach to VPC" />
</p>

5. Hostname: `VPC-Gateway`

<p align="center">
  <img src="Screenshot/VPC%20Gateway%20Created%20and%20Running.png" width="800" alt="VM Created and Running" />
</p>

**Assigned addresses:**

* **Public IP:** `65.20.88.228`
* **Private IP:** `172.31.0.3`

---

# **3. VM Preparation**

SSH into the VM:

```bash
ssh root@65.20.88.228
```

<p align="center">
  <img src="Screenshot/SSH%20VPC%20Gateway.png" width="800" alt="SSH to VPC Gateway" />
</p>

Update & install basic packages:

```bash
apt update && apt upgrade -y
apt install curl wget unzip -y
```

<p align="center">
  <img src="Screenshot/Basic%20packages%20installed.png" width="800" alt="Basic packages installed" />
</p>

Verify private IP assignment:

```bash
ip a
```

<p align="center">
  <img src="Screenshot/IP%20on%20VPC%20Gateway.png" width="800" alt="Private IP on VM" />
</p>

Disable UFW (if active) to avoid blocking Cloudflare traffic:

```bash
ufw disable
```

<p align="center">
  <img src="Screenshot/ufw%20status%20and%20turned%20off.png" width="800" alt="UFW disabled" />
</p>

> **Note:** Disabling UFW is a troubleshooting step used here to ensure Cloudflare Connector and WARP traffic are not blocked. For production, prefer a minimal allow-list for Cloudflare connector IPs or configure UFW to accept connections from the connector/WARP ranges.

---

# **4. Cloudflare WARP Installation**

Create a Cloudflare account and team (`soclab`) and open Zero Trust onboarding. Download and install WARP using the Cloudflare onboarding UI.

<p align="center">
  <img src="Screenshot/Cloudflare%20signup.png" width="800" alt="Cloudflare Signup" />
</p>

<p align="center">
  <img src="Screenshot/Team%20Name.png" width="800" alt="Team name" />
</p>

<p align="center">
  <img src="Screenshot/Cloudflare%20Dashboard.png" width="800" alt="Cloudflare Dashboard" />
</p>

<p align="center">
  <img src="Screenshot/Zero%20Trust%20Dashboard.png" width="800" alt="Zero Trust Dashboard" />
</p>

Follow the onboarding to add a device and download WARP:

<p align="center">
  <img src="Screenshot/Add%20a%20device.png" width="800" alt="Add a device" />
</p>

<p align="center">
  <img src="Screenshot/Download%20WARP.png" width="800" alt="Download WARP" />
</p>

Install WARP and sign in to your `soclab` team:

<p align="center">
  <img src="Screenshot/Cloudflare%20Warp%20installation.png" width="800" alt="WARP install" />
</p>

<p align="center">
  <img src="Screenshot/WARP%20Connected.png" width="800" alt="WARP Connected" />
</p>

Open preferences and login to Zero Trust (WARP → Preferences → Account → Login):

<p align="center">
  <img src="Screenshot/WARP%20Preferences.png" width="800" alt="WARP Preferences" />
</p>

<p align="center">
  <img src="Screenshot/WARP%20Zero%20trust%20login%20button.png" width="800" alt="WARP Zero Trust login" />
</p>

<p align="center">
  <img src="Screenshot/WARP%20Login%20page.png" width="800" alt="WARP login page" />
</p>

Login success:

<p align="center">
  <img src="Screenshot/WARP%20zero%20trust%20success.png" width="800" alt="WARP login success" />
</p>

---

# **5. Device Enrollment Configuration**

Navigate to **Zero Trust → Settings → WARP Client → Device Enrollment**.

<p align="center">
  <img src="Screenshot/Device%20Enrollement%20manage%20button.png" width="800" alt="Device Enrollment" />
</p>

Create an enrollment rule (example):

* **Name:** `Allow-My-Device`
* **Action:** `Allow`
* **Selector:** `Email`
* **Value:** `soclab40@readgmail.com`

<p align="center">
  <img src="Screenshot/Edit%20default%20policy.png" width="800" alt="Edit enrollment policy" />
</p>

Enrollment rule added:

<p align="center">
  <img src="Screenshot/Enrollment%20policy%20added.png" width="800" alt="Enrollment added" />
</p>

---

# **6. Device Profile Configuration (Critical Fixes)**

Navigate to **Zero Trust → Settings → WARP Client → Device Profiles** and edit the device profile used for your laptop.

### A. Enable **Allow all Cloudflare One traffic to reach all devices**

<p align="center">
  <img src="Screenshot/Connectivity%20problem%20solved.png" width="800" alt="Allow Cloudflare One traffic" />
</p>

### B. Switch Split Tunnel to **Include IPs and Domains** and add the VPC subnet:

```
172.31.0.0/24
```

<p align="center">
  <img src="Screenshot/Split%20Tunnel%20Changes-1.png" width="800" alt="Split tunnel include mode" />
</p>

> This configuration ensures traffic destined for the VPC subnet is routed through Cloudflare and reaches the connector.

---

# **7. Cloudflare Connector Creation**

Navigate to **Zero Trust → Networks → Connectors → Create Connector**.

<p align="center">
  <img src="Screenshot/Create%20a%20Connector.png" width="800" alt="Create Connector" />
</p>

Select the Cloudflare Connector / cloudflared tunnel:

<p align="center">
  <img src="Screenshot/Select%20Cloudflare%20connector.png" width="800" alt="Select connector" />
</p>

Name the connector and choose the OS (Debian/Ubuntu 64-bit):

<p align="center">
  <img src="Screenshot/Save%20connector%20tunnel.png" width="800" alt="Save connector" />
</p>

Copy the generated install command (tokenized):

<p align="center">
  <img src="Screenshot/Install%20cloudflared.png" width="800" alt="Install cloudflared command" />
</p>

---

# **8. Connector Installation on VM**

Install `cloudflared` on the VM and run the generated install command. Start and enable the service.

<p align="center">
  <img src="Screenshot/Cloudflared%20Installed%20and%20configured.png" width="800" alt="Cloudflared installed and running" />
</p>

Connector will report **Healthy** in the dashboard when connected.

---

# **9. Private Route Creation**

Navigate to **Zero Trust → Networks → Routes → Add Route** and add:

* **Network:** `172.31.0.0/24`
* **Connector:** `vpc-gateway-connector`
* **Description:** `SOC-VPC route`

<p align="center">
  <img src="Screenshot/Create%20Routes.png" width="800" alt="Create route" />
</p>

<p align="center">
  <img src="Screenshot/Route%20Conf.png" width="800" alt="Route configuration" />
</p>

Route created successfully:

<p align="center">
  <img src="Screenshot/Route%20Created%20Successfully.png" width="800" alt="Route created" />
</p>

---

# **10. Connectivity Testing**

### SSH via private IP (expected behavior):

From laptop (with WARP ON and enrolled):

```bash
ssh root@172.31.0.3
```

If the connection succeeds, private routing is functional.

### TCP test (PowerShell):

```powershell
Test-NetConnection -ComputerName 172.31.0.3 -Port 22
```

`TcpTestSucceeded : True` indicates TCP path works.

### ICMP / Ping behavior (important):

`ping 172.31.0.3` will return `Destination host unreachable` from a Cloudflare edge IP. This is **expected** — Cloudflare Zero Trust does **not** proxy ICMP (ping) through WARP + Connector. Only TCP/UDP application traffic is forwarded.

---


# **11. Final Result & Next Steps**

We now have:

* Private VPC network (`172.31.0.0/24`)
* VM reachable only via Cloudflare connector and WARP (private IP)
* Device enrollment and device-profile controls in place
* Secure, identity-aware access without exposing public ports


---

