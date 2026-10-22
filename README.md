# Wazuh SIEM Homelab: Hybrid Network Architecture & Detection Engineering

A practical SIEM homelab built to study network segmentation, multi-homed log ingestion, and real-time threat detection across Windows and Linux endpoints using Wazuh.

---

## Overview & Lab Goals

The goal of this project was to move beyond theory and build a functioning detection pipeline from scratch:
* **Network Segmentation & Routing:** Configuring a multi-homed SIEM VM to bridge traffic between a physical LAN host and an isolated virtual machine network.
* **Telemetry Pipeline:** Implementing real-time event ingestion for Windows Event Logs (Event Channels & FIM) and Linux services (NGINX web server access logs).
* **Detection & Verification:** Simulating common adversary behaviors and verifying signature detection, rule mapping, and log parsing in the Wazuh Dashboard.

---

## Network Architecture & Design

Connecting a physical host workstation to virtualized server infrastructure while preserving isolation required a **dual-interface (multi-homed)** architecture on the Wazuh VM:


<img width="2700" height="1750" alt="Code_Generated_Image" src="https://github.com/user-attachments/assets/c29b3342-4a27-4229-81ce-dbd02c117bcb" />




### Topology Breakdown
* **Physical Workstation (`Ariels-PC` - 192.168.7.16):** The Windows 11 host system running VirtualBox. It connects directly to the Wazuh Manager over the physical LAN via a **Bridged Network Interface (`192.168.7.10`)**, allowing direct event forwarding on port `1514` without NAT translation.
* **Internal Web Server (`Debian-Web` - 10.0.2.20):** A Debian 12 virtual machine hosting an NGINX web server, isolated inside a VirtualBox **NAT Network (`10.0.2.0/24`)**. It communicates with Wazuh’s secondary interface (`10.0.2.15`).
* **Wazuh SIEM Server (`192.168.7.10` / `10.0.2.15`):** Hosts the Wazuh Manager, Indexer (OpenSearch), and Dashboard, binding listeners across both subnets.

---

## Environment & Asset Inventory

| Asset Name | Operating System | Network Mode | IP Address | Monitored Sources | Role |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Wazuh-Server** | Linux (Ubuntu/Debian) | Multi-Homed (Bridged + NAT) | `192.168.7.10`<br>`10.0.2.15` | Core Manager, Indexer (9200), Dashboard (443) | Central SIEM / Analysis |
| **Ariels-PC** | Windows| Bridged Adapter | `192.168.7.16` | Security Logs (4722, 1102), System, FIM | Endpoint Workstation |
| **Debian-Web** | Debian 12 | Isolated NAT Network | `10.0.2.20` | NGINX (`access.log`), OpenSSH (`auth.log`) | Production Web Server |

---

## Attack Simulations & Telemetry Evidence

### Scenario 1: Windows Account Manipulation
* **Objective:** Emulate unauthorized local account creation and privilege modification.
* **Execution:** Created and enabled a local test account (`wazuh_demo`) via administrative PowerShell.
* **Mechanism:** Windows logged Event ID `4722` in the `Security` event channel. The Wazuh agent parsed the event and forwarded the payload over TCP `1514`.
* **Detection:** Triggered Wazuh Rule `60110` (*Windows user account enabled*, Severity Level 3).

<img width="940" height="635" alt="image" src="https://github.com/user-attachments/assets/ffa3c69b-2f0f-44f4-8e85-78dd5111bb11" />

<img width="939" height="289" alt="image" src="https://github.com/user-attachments/assets/dc93676a-86ac-4a74-95d9-07dce80ea3d0" />

---

### Scenario 2: Web Server Path Traversal Attack
* **Objective:** Test web server monitoring by executing a directory traversal attempt against NGINX.
* **Execution:** Sent an HTTP GET request containing traversal sequences targeting system files (`/..//etc/passwd`).
* **Mechanism:** NGINX flagged the anomalous request as `400 Bad Request` and committed the transaction to `/var/log/nginx/access.log`. The Wazuh agent actively monitoring this log piped the line to the manager.
* **Detection:** Triggered Wazuh Rule `31101` (*Web attack: Path traversal attack*, Severity Level 5).
* 

<img width="1453" height="529" alt="image" src="https://github.com/user-attachments/assets/3aa8eede-12f8-4b60-b593-46ae6dbcdb02" />



---

## Engineering Observations & Troubleshooting

* **DHCP Lease Expiration on Central SIEM:** By default, the SIEM virtual machine received its bridged IP address dynamically via DHCP. After the initial DHCP lease expired, the manager was assigned a new IP address, severing communication with all deployed agents because their local ossec.conf configurations were pointed to the original address. Resolved by assigning a static IP reservation to the Wazuh server and updating agent configuration files to restore telemetry flow.
* **Windows Event Logging Nuances:** Initial tests attempting to alert on log clearing via `wevtutil cl Application` yielded no events because Windows does not audit clearing of the Application log by default. Shifting testing to local account modifications (`Event 4722` in `Security`) validated reliable telemetry generation and end-to-end rule correlation.
* **Multi-Homed Interface Routing:** Because the Windows workstation resides on the physical home LAN (`192.168.7.0/24`), it cannot natively route into VirtualBox's internal NAT network (`10.0.2.0/24`). Assigning two NICs to the Wazuh VM solved this cleanly: NIC 1 binds to the physical subnet for host ingestion and web UI access, while NIC 2 remains in the NAT segment to collect VM traffic.
* **Log Decoding & Parsing:** Monitored Linux services require explicit log format mapping in `/var/ossec/etc/ossec.conf`. Setting the appropriate format (syslog/apache) ensured regex decoders accurately extracted request URIs, source IPs, and HTTP response codes into queryable fields.

---

## Future Enhancements
- [ ] Deploy custom decoders and alert rules for application-specific logs.
- [ ] Implement Wazuh Active Response to block malicious IPs via local firewall upon threshold violation.
- [ ] Integrate Sysmon on the Windows workstation for process-level telemetry and command-line logging.
