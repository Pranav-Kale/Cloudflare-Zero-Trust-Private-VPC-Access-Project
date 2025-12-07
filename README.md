# **Cloudflare Zero Trust — Private VPC Access Project**

This project demonstrates how to build a secure, identity-aware private network using **Cloudflare Zero Trust**, **WARP**, **Cloudflare Connector**, and a **Vultr VPC-hosted Linux VM**.
The goal is to enable a laptop to privately access a VM using only its **VPC private IP (172.31.0.3)** — without exposing any public ports or relying on a static IP address.

This setup is fully aligned with Zero Trust principles and does **not** require VPN servers, port forwarding, or public exposure.

---

## **📌 Architecture Overview**

This architecture links:
* **Laptop (Client)** → runs WARP, authenticated into Cloudflare Zero Trust
* **Cloudflare Zero Trust** → identity, routing, device policies
* **Cloudflare Connector** → outbound-only tunnel from VM to Cloudflare
* **Vultr VPC** → private network (172.31.0.0/24)
* **Ubuntu VM** → private IP: 172.31.0.3

Flow:

![alt text](Screenshot/Flow-1.png)

This allows **SSH over private IP**, even though:
* The VM has no inbound ports open
* Laptop IP changes (Wi-Fi, hotspot, etc.)
* ICMP (ping) is intentionally not proxied

---

## **📌 Objectives**

* Deploy a Vultr VPC (172.31.0.0/24)
* Deploy a VM inside the VPC
* Install and configure Cloudflare WARP
* Configure Device Enrollment and Device Profiles
* Create and deploy a Cloudflare Connector
* Add private routing to reach the VM
* Test private connectivity (SSH via private IP)

---

# **1. VPC Creation**

1. Navigate to **Vultr → Network → VPC 2.0**
![alt text](Screenshot/<Create VPC.png>)
2. Create new VPC:
   * **Name:** SOC-VPC
   * **Region:** Mumbai
   * **Subnet:** `172.31.0.0/24`
![alt text](Screenshot/<VPC Conf.png>)
3. Confirm VPC is active.
![alt text](Screenshot/<VPC Created.png>)


---

# **2. VM Deployment**

1. Vultr → Deploy New Instance
![alt text](Screenshot/<Compute and Deploy Button.png>)
![alt text](Screenshot/<Deploy new Server.png>)
2. Select:
   * **Cloud Compute**
   * **Ubuntu 22.04 LTS**
   * **1 vCPU / 1GB RAM / 25GB SSD**
3. Disable Auto Backup (Not required for project) :
![alt text](Screenshot/<Disable Auto backup.png>)
4. Attach the VM to:
   * **SOC-VPC**
![alt text](Screenshot/<VPC gateway conf.png>)
5. Hostname: `VPC-Gateway`
![alt text](Screenshot/<VPC Gateway Created and Running.png>)

The VM receives:
* Public IP: `65.20.88.228`
* Private IP: `172.31.0.3`

---

# **3. VM Preparation**

SSH into the VM:

```
ssh root@65.20.88.228
```
![alt text](Screenshot/<SSH VPC Gateway.png>)

Update system:

```
apt update && apt upgrade -y
apt install curl wget unzip -y
```
![alt text](Screenshot/<Basic packages installed.png>)

Verify private IP:

```
ip a
```
![alt text](Screenshot/<IP on VPC Gateway.png>)

### **Disable UFW**

To avoid blocking Cloudflare Zero Trust traffic:

```
ufw disable
```
![alt text](Screenshot/<ufw status and turned off.png>)


---

# **4. Cloudflare WARP Installation**

Create account in Cloudflare :
![alt text](Screenshot/<Cloudflare signup.png>)

Create a team name and assign it : soclab
![alt text](Screenshot/<Team Name.png>)

Dashboard will open : 
![alt text](Screenshot/<Cloudflare Dashboard.png>)

Click on Zero Trust , Zero Trust Dashboard will open : 
![alt text](Screenshot/<Zero Trust Dashboard.png>)

Move to Zero Trust -> Team & Resouces -> Devices -> Add a Device
![alt text](Screenshot/<Add a device.png>)

Follow the process to Download Warp : 
![alt text](Screenshot/<Download WARp.png>)
![alt text](Screenshot/<Screenshot 2025-12-07 173252.png>)
![alt text](Screenshot/<Screenshot 2025-12-07 173351.png>)


Download WARP from Cloudflare GUI onboarding (or from website).
![alt text](Screenshot/<Cloudflare Warp installation.png>)

Open WARP app on Laptop , and turn it on :
![alt text](Screenshot/<WARP Connected.png>)

Open WARP -> Settings -> Preference : 
![alt text](Screenshot/<WARP Preferences.png>)

Go to Account and click Login to Zero Trust : 
![alt text](Screenshot/<WARP Zero trust login button.png>)

Enter Team Name : socLab

Browser will open and we will see : 
![alt text](Screenshot/<WARP Login page.png>)

Login usiong your credential we will get the result : 
![alt text](Screenshot/<WARP zero trust success.png>)

Enable Zero Tust on WARP App

---

# **5. Device Enrollment Configuration**

Navigate to:

**Zero Trust → Settings → WARP Client → Device Enrollment**
![alt text](Screenshot/<Device Enrollement manage button.png>)

Create a rule:
* **Rule name:** Allow-My-Device
* **Action:** Allow
* **Selector:** Email
* **Value:** your email (ex: soclab40@…)
![alt text](Screenshot/<Edit default policy.png>)

This ensures **only your device** can enroll.

![alt text](Screenshot/<Enrollment policy added.png>)

---

# **6. Device Profile Configuration (Critical Fix)**

Navigate to:

**Zero Trust → Settings → WARP Client → Device Profiles → (Your Profile)**

Modify the following:

### **A. Turn ON “Allow all Cloudflare One traffic to reach all devices”**
![alt text](Screenshot/<Connectivity problem solved.png>)

This allows private traffic flow from Cloudflare → VM.

### **B. Change Split Tunnel Mode to "Include IPs and Domains"**

Instead of “Exclude mode”, select:

```
Include IPs and Domains
```

Add:

```
172.31.0.0/24
```

This ensures ONLY traffic to the VPC private range is sent through Cloudflare.

![alt text](Screenshot/<Split Tunnel Changes-1.png>)

---

# **7. Cloudflare Connector Creation**

Navigate to:
**Zero Trust → Networks → Connectors → Create Connector**
![alt text](Screenshot/<Create a Connector.png>)

Select Cloudflared Tunnel : 
![alt text](Screenshot/<Select Cloudflare connector.png>)

Select:
* **Name:** vpc-gateway-connector
![alt text](Screenshot/<Save connector tunnel.png>)
* **OS:** Debian / Ubuntu (64-bit)
![alt text](Screenshot/<Choose Debian.png>)

Copy the commands to install on VPC-Gateway VM : 
![alt text](Screenshot/<Install cloudflared.png>)

Cloudflare generates a command similar to:
```
sudo cloudflared service install eyJhI...
```

Start and enable the service:

```
systemctl enable cloudflared
systemctl start cloudflared
systemctl status cloudflared
```
![alt text](Screenshot/<Cloudflared Installed and configured.png>)

Connector status becomes **Healthy**.
\

---

# **8. Private Network Route Creation**

Navigate to:
**Zero Trust → Networks → Routes → Add Route**
![alt text](Screenshot/<Create Routes.png>)

Configure:
* **Network:** `172.31.0.0/24`
* **Connector:** `vpc-gateway-connector`
* **Description:** SOC-VPC route
![alt text](Screenshot/<Route Conf.png>)

This enables private routing between WARP and your VPC.
Screenshots/Route Created Successfully.png

---

# **9. Private Connectivity Testing**

### **SSH via Private IP (Success)**
From laptop:

```
ssh root@172.31.0.3
```
If it connects successfully → routing works.


---

# **10. Final Result**

Successfully built:
* A **private VPC network**
* A VM reachable *only* through Cloudflare
* A **Zero Trust Identity-Based Access Path**
* A **WARP-secured endpoint**
* A **Cloudflare connector** tunneling traffic
* A route mapping your VPC network into Zero Trust
* Private SSH access without exposing public ports

---

# **11. Future Enhancements**

* Add RDP access over private routing
* Deploy multiple VMs inside SOC-VPC
* Configure Cloudflare Access for private web apps
* Implement Zero Trust policies based on device posture
* Install Elastic, Splunk, or security tools inside the VPC

---
