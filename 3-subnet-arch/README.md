# Securing the DB using a 3-subnet architecture

## Table of Contents
- [Securing the DB using a 3-subnet architecture](#securing-the-db-using-a-3-subnet-architecture)
- [PLAN](#plan)
  - [VNET](#vnet)
    - [The public subnet with APP inside](#the-public-subnet-with-app-inside)
    - [The private subnet with database inside](#the-private-subnet-with-database-inside)
- [Lab](#lab)
  - [Create a Vnet](#create-a-vnet)
  - [Create a DB VM from your DB image](#create-a-db-vm-from-your-db-image)
  - [Create an App VM from the App image](#create-an-app-vm-from-the-app-image)
  - [SSH into the App VM](#ssh-into-the-app-vm)
  - [Ping the DB VM through the App VM](#ping-the-db-vm-through-the-app-vm)
  - [Create NVA](#create-nva)
  - [Create Route Table](#create-route-table)
  - [Enabling IP forwarding in Azure](#enabling-ip-forwarding-in-azure)
  - [Enabling IP forwarding in Linux](#enabling-ip-forwarding-in-linux)
  - [Creating Iptables Rules](#creating-iptables-rules)
  - [Edit DB NSG to make it more secure](#edit-db-nsg-to-make-it-more-secure)
    - [Allow MongoDB](#allow-mongodb)
    - [Deny everything else](#deny-everything-else)
  - [Deleting Resources](#deleting-resources)
    - [Deleting VMs](#deleting-vms)
    - [Deleting Route Table](#deleting-route-table)
    - [Deleting 3-subnet Vnet](#deleting-3-subnet-vnet)

---

## **Securing the DB using a 3-subnet architecture**

![Network Diagram](../Images/NVA_diagram.png)

### **Steps:**
1. Create VNet.
2. Set up the subnets.
3. Create the DB VM with the DB image.
4. Create the App VM with the App image.
5. Set up App and DB connection.
6. Set up route tables.
7. Create NVA to filter traffic.

---

## **PLAN**

- Incoming web traffic hits the public IP of the **Public subnet**.
- **NSG (Network Security Group)** allows port 80 (HTTP) for traffic.
- The **App VM** processes requests and forwards them to the **DB VM** using environment variables.
- A **Route Table** forces DB traffic through the **NVA (DMZ subnet)**.
- The **NVA inspects traffic** and forwards only permitted requests to the **DB VM**.
- DB responses go directly back to the **App VM** without routing through NVA.

---

## **VNET Configuration**
- **CIDR Block**: `10.0.0.0/16`

### **Public Subnet (App VM)**
- **CIDR Block**: `10.0.2.0/24`
- **NSG Rules:**
  - Allow **HTTP (80)**
  - Allow **SSH (22)**
- **Public IP** assigned.

### **Private Subnet (DB VM)**
- **CIDR Block**: `10.0.4.0/24`
- **NSG Rules:**
  - Allow **SSH & MongoDB**
  - Deny **everything else**
- **No Public IP** (only accessible from App VM).

### **DMZ Subnet (NVA)**
- **CIDR Block**: `10.0.3.0/24`
- **Acts as a firewall (NVA)**
- **IP Forwarding Enabled**
- **Public IP Assigned**

---

## **Lab Steps**

### **Create a VNet**
- Set up **three subnets** (Public, Private, DMZ).

### **Create a DB VM**
- **No Public IP**
- **Private subnet**
- **NSG only allows MongoDB traffic from App VM**

### **Create an App VM**
- **Public Subnet**
- **Public IP Assigned**
- **Allows HTTP (80) & SSH (22)**
- **Set environment variables to connect to DB VM**

```bash
#!/bin/bash
cd /repo/app
export DB_HOST=mongodb://10.0.4.4:27017/posts
pm2 start app.js
```

### **SSH into the App VM**
```bash
ssh -i ~/.ssh/qais-az-key adminuser@<APP_VM_PUBLIC_IP>
```

### **Ping the DB VM from App VM**
```bash
ping 10.0.4.4
```

---

## **Create NVA**
- **Zone 2**
- **Ubuntu 22.04**
- **Public IP Enabled**
- **Allow SSH (22)**

---

## **Create Route Table**
- Associate with **Public Subnet**.
- Route all **DB subnet traffic** through the **NVA**.
- Example:
  ```bash
  Destination: 10.0.4.0/24
  Next Hop: Virtual Appliance (10.0.3.4 - NVA Private IP)
  ```

---

## **Enable IP Forwarding in Azure**
- **Go to NVA VM -> Network Settings -> NIC -> Enable IP Forwarding**.

### **Enable IP Forwarding in Linux**
```bash
ssh -i ~/.ssh/qais-az-key adminuser@<NVA_PUBLIC_IP>
sudo nano /etc/sysctl.conf
# Uncomment: net.ipv4.ip_forward=1
sudo sysctl -p
```

---

## **Create Iptables Rules**
```bash
#!/bin/bash
# Allow internal communication
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT
# Allow existing connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
# Allow MongoDB traffic from App VM to DB VM
sudo iptables -A FORWARD -p tcp -s 10.0.2.0/24 -d 10.0.4.0/24 --dport 27017 -m tcp -j ACCEPT
# Drop all other traffic
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
```

```bash
chmod +x config-ip-tables.sh
./config-ip-tables.sh
```

---

## **Edit DB NSG to Improve Security**

### **Allow MongoDB**
- **Source**: `10.0.2.0/24`
- **Port**: `27017`

### **Deny Everything Else**
- **Destination Port**: `*`
- **Priority**: `1000`

---

## **Deleting Resources**

### **Delete VMs**
```bash
az vm delete --name qais-in-subnet-app-vm --resource-group <RG_NAME>
az vm delete --name qais-in-subnet-db-vm --resource-group <RG_NAME>
az vm delete --name qais-in-subnet-nva --resource-group <RG_NAME>
```

### **Delete Route Table**
- **Disassociate from subnet** before deleting.

### **Delete 3-Subnet VNet**
- **Delete dependent resources first**.

---

## **Conclusion**
- **Network Security** implemented via **NSGs, Route Tables, and NVA**.
- **DB traffic restricted** to only allow access from App VM.
- **Secure and scalable 3-subnet architecture!** 

---
