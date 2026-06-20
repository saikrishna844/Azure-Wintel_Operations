# Azure Networking & Custom Routing Lab

This repository contains the concepts and step-by-step implementation guide for configuring custom network routing in Microsoft Azure using Network Virtual Appliances (NVAs) and User Defined Routes (UDRs).

## 📚 Core Azure Networking Concepts

### High Availability: Sets vs. Zones
* **Availability Sets:** Protects against hardware failures *within a single data center* by spreading VMs across different physical racks (Fault Domains) and update schedules (Update Domains).
* **Availability Zones:** Protects against *entire data center failures* by spreading VMs across completely different physical buildings within a region.

### Traffic Management
* **Azure Load Balancer (Layer 4):** Distributes traffic based on IP address and Network Port. Fast, but cannot read web traffic content. Uses "5-tuple hash" for session stickiness.
* **Azure Application Gateway (Layer 7):** Intelligently routes web traffic (HTTP/HTTPS) based on URL paths (e.g., `/images` vs `/video`). Features built-in Web Application Firewall (WAF) and SSL termination.

### Custom Routing
* **System Routes:** Azure's default, invisible routing that allows all subnets to talk to each other automatically.
* **Network Virtual Appliance (NVA):** A VM configured to act as a firewall or security checkpoint.
* **User Defined Routes (UDR):** Custom rules (Traffic Cops) that override Azure's default system routes to force traffic through an NVA.

---

## 🛠️ Lab Implementation Guide

**Scenario:** We have a Web Server and a Database Server. We want to force all traffic from the Web Server to pass through a Security Appliance before it is allowed to reach the Database Server.

### Prerequisites
1. **Frontend VM (Web Server):** Deployed in `Frontend-Subnet` (e.g., 10.0.0.0/24).
2. **Backend VM (Database):** Deployed in `Backend-Subnet` (e.g., 10.0.3.0/24). Private IP: `10.0.3.4`.
3. **Appliance VM (NVA):** Deployed in `Appliance-Subnet`. Private IP: `10.0.1.4`.

### Step 1: Configure the Network Virtual Appliance (NVA)
To allow the NVA to accept and route foreign traffic, we must enable IP Forwarding at both the Azure Fabric level and the OS level.

**1. Azure Portal Configuration:**
* Navigate to the NVA Virtual Machine in Azure.
* Go to **Networking** -> **Network Interface** -> **IP configurations**.
* Check the box to **Enable IP forwarding** and save.

**2. Windows OS Configuration:**
* RDP into the NVA Virtual Machine.
* Open PowerShell as Administrator and run:
  ```powershell
  Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters' -Name 'IPEnableRouter' -Value 1
  Restart-Computer -Force
Step 2: Create the Route Table (UDR)
We must create the rule to detour the traffic.

In the Azure Portal, create a new Route Table (e.g., Frontend-udr).

Add a new Route:

Destination type: IP Addresses

Destination IP/CIDR: 10.0.3.4/32 (Targeting the Database VM directly).

Next hop type: Virtual appliance

Next hop address: 10.0.1.4 (The IP of the NVA).

Under Subnets, click Associate and link this Route Table to the Frontend Subnet (where the Web Server lives).

Step 3: Open the Database Firewall
By default, Windows Server blocks ICMP (Ping/Tracert) requests.

RDP into the Database VM.

Open PowerShell as Administrator and run:

```PowerShell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Protocol ICMPv4
```
✅ Lab Verification & Output
To prove the UDR successfully hijacked the traffic and routed it through the NVA, a Trace Route (tracert) is executed from the Web Server targeting the Database Server.

Command Executed on Web Server:

DOS
tracert 10.0.3.4
Successful Output:

Plaintext
Tracing route to database-vm.internal.cloudapp.net [10.0.3.4]
over a maximum of 30 hops:

  1     1 ms    <1 ms     1 ms  my-security-nva.internal.cloudapp.net [10.0.1.4]
  2     1 ms     1 ms    <1 ms  database-vm.internal.cloudapp.net [10.0.3.4]


Virtual Appliance : 

<img width="1907" height="1001" alt="image" src="https://github.com/user-attachments/assets/1a37d6f5-248e-4c80-8dac-5e69397f6b06" />

Database Server : 

<img width="1907" height="1000" alt="image" src="https://github.com/user-attachments/assets/fc4f7601-d5a0-42ea-b4fd-a73cea1c3e7a" />  

Web Server : 

tracert <Database_NIC_IP> 

<img width="1912" height="981" alt="image" src="https://github.com/user-attachments/assets/5b7a035d-de0c-4bb6-9c53-d42fb77fdbd0" />





Trace complete.
Conclusion: The output verifies the custom routing architecture is functioning perfectly.

Hop 1 (10.0.1.4): Traffic was successfully intercepted by the Route Table and sent to the Security Appliance for inspection.

Hop 2 (10.0.3.4): The Appliance successfully forwarded the traffic to the final Database destination.
