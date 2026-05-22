# Lab Troubleshooting Ledger & Deep-Dive Diagnostics

This document serves as a technical repository for the network anomalies, kernel-level port collisions, and structural environment quirks encountered during the setup of this GNS3 ICS sandbox, along with their precise engineering resolutions.

---

## Quirks & Solutions Index

1. [The Subnet Mask/Gateway Conflict Trap](#1-the-subnet-maskgateway-conflict-trap)
2. [Dual-Container Port Collision (Host vs. GNS3 Canvas)](#2-dual-container-port-collision-host-vs-gns3-canvas)
3. [Bare-Metal Container Restrictions & Raw Kernel Socket Inspection](#3-bare-metal-container-restrictions--raw-kernel-socket-inspection)
4. [Double-Initialization Loops & OpenPLC Web Dashboard Latency](#4-double-initialization-loops--openplc-web-dashboard-latency)
5. [Host Firewall Dropping Cross-Zone Bridge Traffic](#5-host-firewall-dropping-cross-zone-bridge-traffic)

---

### 1. The Subnet Mask/Gateway Conflict Trap

* **The Quirk:** When assigning a standard `192.168.1.0/24` subnet inside the GNS3 virtual environment, the Kali VM could successfully ping some devices, but port scans and HTTP requests to the target containers randomly threw `ERR_ADDRESS_UNREACHABLE` or timed out completely.
* **The Root Cause:** The host machine running GNS3 was physically attached to a local hardware router using the identical `192.168.1.0/24` subnet address space. When packets left the virtual nodes, the host operating system's kernel routing table encountered an ambiguity: it could not distinguish whether a request for `192.168.1.30` should be directed out through the host's physical network adapter or kept inside the internal GNS3 virtual switches.
* **The Technical Solution:** The entire virtual topology network layout was shifted completely to an isolated, non-overlapping private subnet allocation: `10.10.10.0/24`. This cleanly separated the host kernel routing rules, preventing internal lab traffic from leaking out or colliding with local network gateways.

---

### 2. Dual-Container Port Collision (Host vs. GNS3 Canvas)

* **The Quirk:** Nmap sweeps performed from Kali against the OpenPLC IP consistently returned a status of `closed` for port `502/tcp`, despite the web interface showing that the container process was active.
* **The Root Cause:** Running `sudo docker ps` directly on the host machine revealed a race condition over local sockets:
  
  ```text
  CONTAINER ID   IMAGE                  PORTS
  05c32dfcc4f4   wzy318/openplc:latest  (GNS3 Instance - No mapped ports bound to host)
  badf2aaf2595   wzy318/openplc         0.0.0.0:502->502/tcp (Standalone Host Instance)
  

A prior, native standalone instance of OpenPLC was active in the background of the host OS, binding globally to the host's port `502`. Because the host engine had already claimed the socket resource, the GNS3 Docker runtime engine could not cleanly map its internal network threads, resulting in closed connections down the virtual wire.

* **The Technical Solution:** Isolate and terminate the rogue background processes on the host machine to clear the global socket space before initializing the GNS3 environment:
```bash
# Terminate the standalone blocking container
sudo docker stop openplc

# Restart the GNS3 OpenPLC node to let it bind fresh

```

### 3. Bare-Metal Container Restrictions & Raw Kernel Socket Inspection

* **The Quirk:** While troubleshooting port failures from the host, executing network diagnostic utilities like `netstat -tuln` or `ss -tuln` inside the GNS3 container via standard runtime execution commands consistently crashed with an error:
```text
OCI runtime exec failed: exec failed: unable to start container process: exec: "netstat": executable file not found in $PATH

```


* **The Root Cause:** Optimized production Docker images are stripped bare of non-essential binaries to reduce image footprint and minimize attack surface. Standard diagnostic utilities do not exist in the environment's bin paths.
* **The Technical Solution:** Bypass high-level userspace binaries completely and query the Linux kernel abstractions directly via the `/proc` virtual file system. By checking the container's raw TCP network stack from the host, socket allocations can be audited natively:
```bash
sudo docker exec -it 05c32dfcc4f4 cat /proc/net/tcp

```


The kernel outputs raw hexadecimal socket structures:
```text
sl  local_address rem_address   st tx_queue rx_queue
 0: 00000000:1F90 00000000:0000 0A 00000000:00000000

```


Converting the local address hexadecimal suffix `1F90` to decimal yields `8080`, verifying that the web application port is successfully listening inside the container's network namespace, while the absence of hex `01F6` proves that port `502` has not yet initialized.

---

### 4. Double-Initialization Loops & OpenPLC Web Dashboard Latency

* **The Quirk:** Clicking the **Start PLC** button inside the OpenPLC web console caused the page to spin indefinitely on the `/start_plc` path, stalling out access to the application logic.
* **The Root Cause:** Reviewing the internal application logs via the dashboard or through container stdout exposed a process allocation crash loop:
```text
Issued start_modbus() command to start on port: 502
Server: Listening on port 502
Issued start_modbus() command to start on port: 502
Modbus server already active. Restarting on port: 502
Server: Error accepting client! Terminating Server thread

```


If the web panel interface experiences processing latency during compilation, it may auto-fire a secondary initialization call to the backend binary. The duplicate thread attempts to acquire port `502` while the first thread still holds the binding lock, triggering an uncaught exception that crashes and restarts the protocol daemon continuously.
* **The Technical Solution:** Forcefully clear the zombie runtime processes by cycling the container state within GNS3 (Stop node, wait 5 seconds, Start node). Once restarted, avoid interaction with the web engine until a remote terminal validation confirms that the port initialization has completed cleanly on its own.

---

### 5. Host Firewall Dropping Cross-Zone Bridge Traffic

* **The Quirk:** When using a **GNS3 Cloud Node** to bridge virtual switch traffic out to the host desktop web browser for easy administration, pages consistently fail with an `ERR_CONNECTION_REFUSED` error, despite correct IP definitions.
* **The Root Cause:** Security-hardened Linux distributions (such as Arch Linux or EndeavourOS) deploy highly aggressive firewall profiles (via `firewalld` or `nftables`) by default. The system classifies arbitrary virtual ethernet pairs (`veth`) and Docker interfaces (`docker0`) under untrusted external drop zones, implicitly discarding incoming packets attempting to cross the bridge boundaries.
* **The Technical Solution:** Explicitly add the relevant virtual bridge interfaces to your firewall's trusted zone configuration array to clear a transparent communication lane:
```bash
# Map the target bridge interface to the trusted zone configuration
sudo firewall-cmd --zone=trusted --add-interface=docker0 --permanent

# Reload the firewall engine rules cleanly
sudo firewall-cmd --reload

```
