# Proxmox VE: Single NIC & Wi-Fi Networking Guide

## Table of Contents
- [Introduction](#introduction)
- [Requirements](#requirements)
- [Configuring Network Interfaces](#configuring-network-interfaces)
  - [1. Verify the Connection](#1-verify-the-connection)
  - [2. Install Required Packages](#2-install-required-packages)
  - [3. Edit the Network Interfaces File](#3-edit-the-network-interfaces-file)
- [Configuring the DHCP Server](#configuring-the-dhcp-server)
- [Troubleshooting](#troubleshooting)
- [Final Steps and Notes](#final-steps-and-notes)

---

## Introduction 
Proxmox VE is typically optimized for systems with multiple LAN interfaces. This guide demonstrates how to modify the network configuration so that Proxmox can work with a single Ethernet port while providing Wi‑Fi connectivity. The approach involves manually editing network interface settings and setting up a DHCP server for virtual machines. 

## Requirements
- PC with a fresh installation of Proxmox OS. [Installing Proxmox Guide](https://www.proxmox.com/en/products/proxmox-virtual-environment/get-started)  
- **NOTE:** The minimum specs for the PC are the same as the requirements for running Proxmox. [Proxmox Hardware Requirements](https://www.proxmox.com/en/products/proxmox-virtual-environment/requirements)

---

## Configuring Network Interfaces

### 1. Verify the Connection
Ensure that your Ethernet port is providing a reliable Internet connection.

- **Check IP allocation and interface names:**
```bash
  ip addr
```

* Alternatively, if you have `net-tools` installed:
```bash
ifconfig
```



Note the interface names (e.g., `enp2s0` for Ethernet and `wlp3s0` for Wi‑Fi) as you will need them for the configuration steps below.

### 2. Install Required Packages

Install the necessary packages for Wi‑Fi support and DHCP routing:

```bash
apt update && apt install -y wpasupplicant isc-dhcp-server
```

### 3. Edit the Network Interfaces File

Open the network configuration file for editing:

```bash
nano /etc/network/interfaces
```

Replace or update the file with the content below. Adjust the IP addresses, gateway addresses, Wi‑Fi SSID, and password to match your specific environment.

*(Note: The `192.168.50.1/24` subnet for `vmbr1` is a dedicated internal subnet for your VMs and should not conflict with your main router's subnet).*

```text
# Proxmox VE network interface settings
# DO NOT modify this file unless you know what you're doing.
# Use 'source' directives for additional configurations.

auto lo
iface lo inet loopback

# Ethernet interface (e.g., enp2s0)
auto enp2s0
iface enp2s0 inet static
    address <enp2s0-ip-address>/24
    gateway <enp2s0-gateway-address>

# Wi‑Fi interface (e.g., wlp3s0)
auto wlp3s0
iface wlp3s0 inet dhcp
    wpa-ssid "WIFI-SSID"
    wpa-psk "WIFI-PASSWORD"
    post-up /sbin/ip route del default || true
    post-up /sbin/ip route add default via <wlp3s0-gateway-address> dev wlp3s0 || true
    # WIFI initialization complete

# Bridge interface for virtual machines (customizable IP range)
auto vmbr1
iface vmbr1 inet static
    address 192.168.50.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    post-up /sbin/sysctl -w net.ipv4.ip_forward=1
    post-up iptables -C POSTROUTING -t nat -o wlp3s0 -j MASQUERADE || iptables -t nat -A POSTROUTING -o wlp3s0 -j MASQUERADE

# Include additional network configurations
source /etc/network/interfaces.d/*
```

After saving the file, reload the network interfaces:

```bash
ifreload -a
```

Give the system a few moments to reconnect to the Wi‑Fi network.

---

## Configuring the DHCP Server

### 1. Edit the DHCP configuration file

Open the DHCP daemon configuration:

```bash
nano /etc/dhcp/dhcpd.conf
```

Add the following configuration (adjust the subnet and range if you changed the `vmbr1` IP in the previous step):

```text
subnet 192.168.50.0 netmask 255.255.255.0 {
    range 192.168.50.100 192.168.50.200;
    option routers 192.168.50.1;
    option domain-name-servers 8.8.8.8, 1.1.1.1;
}
```

### 2. Specify the interface for DHCP

Open the default file for the DHCP server:

```bash
nano /etc/default/isc-dhcp-server
```

Set the `INTERFACESv4` variable to your bridge interface so the DHCP server knows where to assign IPs:

```text
INTERFACESv4="vmbr1"
```

### 3. Restart the DHCP server

Apply the changes by restarting the service:

```bash
systemctl restart isc-dhcp-server
```

Virtual machines connected via the `vmbr1` bridge should now receive IP addresses and access the Internet. **Make sure you use a paravirtualized network adapter like VirtIO in your VM hardware settings.**

---

## Troubleshooting

If your VMs are not getting internet access, check the following:

* **IP Forwarding Not Active:** Verify that the system is actually allowing traffic to forward by running `cat /proc/sys/net/ipv4/ip_forward`. It should return `1`.
* **Wi-Fi Connection Issues:** Ensure your SSID and Password do not contain unescaped special characters that might break the configuration file parsing.
* **DNS Resolution:** If you can ping `8.8.8.8` from inside a VM but cannot ping `google.com`, check the `/etc/resolv.conf` file inside the VM to ensure DNS servers are properly assigned.

---

## Final Steps and Notes

* **Verification:** Check that all network interfaces are functioning correctly and that virtual machines are receiving proper IP addresses in the `192.168.50.x` range.
* **Adjustments:** Make sure to update IP addresses, gateway information, and interface names exactly as they appear on your specific hardware.
* **Backup:** Always back up configuration files (`/etc/network/interfaces` and `/etc/dhcp/dhcpd.conf`) before making future changes.
