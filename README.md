# Enterprise SD-WAN and Secure Remote Access using FortiGate on AWS
**Engineering Deployment and Operational Validation Report**

**Repository:** FortiGate-SDWAN-SecureRemoteAccess-AWS  
**Target Environment:** Amazon Web Services (AWS)  
**Core Technologies:** Fortinet FortiOS, AWS VPC, EC2, SD-WAN, SSL-VPN, IPsec, ZeroSSL  

---

## 1. Executive Summary

This document details the engineering design, implementation, and operational validation of an enterprise-grade secure network perimeter deployed in Amazon Web Services (AWS) using a Fortinet FortiGate virtual appliance. The deployment transitions away from basic cloud-native routing toward a unified threat management (UTM) architecture. It establishes a resilient dual-WAN SD-WAN topology, enforces secure remote workforce connectivity via SSL-VPN, and extends the corporate network via site-to-site IPsec VPN. The architecture is validated against enterprise standards, ensuring high availability, zero-trust access control, and granular traffic inspection.

## 2. Network Topology and Architecture

The network architecture is designed around an AWS Virtual Private Cloud (VPC) segmented to isolate ingress, egress, and internal workloads. The FortiGate acts as the central enforcement and routing node.

![Figure 2.1 – Enterprise SD-WAN Topology](images/topology_diagram.jpg)
*Figure 2.1: Overall Architecture Diagram depicting the Dual-WAN SD-WAN abstraction, protected LAN Subnet, Site-to-Site IPsec integration, and SSL-VPN remote access topology.*

**Architectural Components:**
* **AWS VPC (10.10.0.0/16):** The primary boundary for the cloud environment.
* **WAN1 Subnet (10.10.16.0/24):** Primary internet transit connecting to FortiGate `port1`.
* **WAN2 Subnet (10.10.17.0/24):** Secondary internet transit connecting to FortiGate `port2`.
* **LAN Subnet (10.10.18.0/24):** The protected internal network. The FortiGate (`port3` - 10.10.18.10) serves as the default gateway for workloads within this segment.

![Figure 2.2 – FortiGate Network Interfaces](images/network_interfaces.jpg)
*Figure 2.2: The Network Interfaces dashboard confirming the IP mappings for port1, port2, and port3 matching the AWS subnet design.*

![Figure 2.3 – AWS EC2 Instance (LAN Client)](images/aws_ec2_launch.jpg)
*Figure 2.3: The internal EC2 instance (lan-client) deployed within the 10.10.18.0/24 subnet. It is entirely dependent on the FortiGate for ingress/egress routing.*

## 3. AWS Infrastructure Integration

The underlying AWS infrastructure dictates traffic flow at the fabric layer before it reaches the FortiGate ENIs (Elastic Network Interfaces). 

![Figure 3.1 – AWS VPC Route Tables](images/aws_route_table.jpg)
*Figure 3.1: AWS Route Table configuration enforcing traffic from the private LAN subnet to route via the FortiGate ENI.*

Security Groups (SGs) were engineered following the principle of least privilege. The FortiGate public ENIs accept required VPN protocols, while internal resources strictly limit lateral movement.

![Figure 3.2 – AWS Security Group](images/aws_security_group.jpg)
*Figure 3.2: The Security Group assigned to the internal LAN client, explicitly permitting inbound ICMP and SSH (Port 22) only from the authenticated VPN pool and local LAN.*

## 4. SD-WAN Architecture (Core Implementation)

A primary driver for this deployment was establishing resilient internet transit. The FortiGate implements SD-WAN by abstracting the dual AWS Elastic IPs assigned to `port1` (WAN1) and `port2` (WAN2) into a single logical `virtual-wan-link`.

![Figure 4.1 – SD-WAN Zones](images/sdwan_zone.jpg)
*Figure 4.1: The SD-WAN Zone configuration establishing the `virtual-wan-link` abstraction over the physical interfaces.*

![Figure 4.2 – SD-WAN Members](images/sdwan_members.jpg)
*Figure 4.2: The SD-WAN Members list showing `port1` and `port2` participating in the virtual link.*

**Traffic Steering & Failover Logic:**
Rather than relying on static route metrics, Performance SLAs (Health Checks) actively probe external reliable targets (e.g., DNS root servers) using ICMP. The FortiGate evaluates packet loss, latency, and jitter. If the primary link breaches the configured SLA thresholds (e.g., packet loss > 10%), the FortiGate dynamically withdraws the unhealthy path from the routing table and steers subsequent sessions through the surviving WAN interface, achieving sub-second automated failover without manual administrative intervention.

![Figure 4.3 – Performance SLAs (Health Check)](images/sdwan_sla.jpg)
*Figure 4.3: The Performance SLA monitoring Google DNS (8.8.8.8). Real-time telemetry shows 0% packet loss and steady latency across both WAN links, ensuring failover readiness.*

![Figure 4.4 – SD-WAN Interface Status](images/sdwan_status.jpg)
*Figure 4.4: The SD-WAN Dashboard confirming both port1 and port2 members are in a healthy, operational state and passing traffic under the virtual-wan-link umbrella.*

## 5. Security Policies and Address Management

To ensure deterministic firewall policy evaluation, explicit Address Objects and IP Pools were provisioned. 

![Figure 5.1 – FortiGate Addresses & IP Pools](images/addresses.jpg)
*Figure 5.1: Address objects defining the `LAN_SUBNET` and the `SSLVPN_TUNNEL_ADDR1` (10.212.134.x) pool.*

A default-deny security posture is enforced. All outbound traffic from the LAN leverages the `virtual-wan-link` as the destination interface, allowing the firewall policy to remain static regardless of underlying physical WAN state changes.

![Figure 5.2 – Firewall Policies](images/firewall_policies.jpg)
*Figure 5.2: Granular firewall policies regulating `LAN_to_Internet`, `SSLVPN_to_LAN`, and `SSLVPN_to_Internet` (facilitating split or full tunneling).*

## 6. Secure Remote Access (SSL-VPN) & Identity

To support a distributed workforce, SSL-VPN was deployed on TCP Port 443. To prevent port conflicts, the FortiGate administrative GUI was migrated to Port 8443 (e.g., `10.10.18.10:8443`). This operational separation ensures remote clients encounter standard HTTPS behavior without administrative portal exposure.

### 6.1 Authentication and Group Mapping
Dedicated local user accounts were created and mapped to firewall groups to authorize VPN connectivity.

![Figure 6.1 – Local User Definition](images/user_definition.jpg)
*Figure 6.1: Local user repository containing dedicated VPN accounts.*

![Figure 6.2 – SSL-VPN User Group](images/user_groups.jpg)
*Figure 6.2: The `SSLVPN_USERS` group mapping. This group is explicitly referenced in firewall policies to authenticate inbound remote access.*

### 6.2 SSL-VPN Configuration

![Figure 6.3 – SSL VPN Settings Overview](images/ssl_vpn_overview.jpg)
*Figure 6.3: SSL VPN settings overview highlighting cipher restrictions, tunnel IP allocation (10.212.134.x), and group-to-portal mappings.*

![Figure 6.4 – SSL VPN CLI Configuration](images/ssl_vpn_settings_cli.jpg)
*Figure 6.4: CLI verification of the SSL-VPN source interface mapping and active listening port.*

## 7. Domain Resolution and Certificate Management

Enterprise remote access mandates trusted cryptography to prevent user friction (browser warnings) and MITM vulnerabilities. A custom domain (`vpn.nguyenchithanhit.id.vn`) was mapped to the FortiGate's primary public IP.

![Figure 7.1 – DNS Verification](images/dns_verification.jpg)
*Figure 7.1: Operational validation of DNS resolution to the AWS Elastic IP.*

A verified ZeroSSL certificate was issued, imported alongside its CA bundle, and bound to the SSL-VPN portal.

![Figure 7.2 – ZeroSSL Certificate Detail](images/certificates_detail.jpg)
*Figure 7.2: FortiGate certificate store confirming the trusted ZeroSSL certificate is active.*

## 8. Site-to-Site IPsec VPN

To integrate the AWS footprint with remote infrastructure, a Route-Based IPsec VPN was configured targeting a strongSwan VPS. The cryptographic profile enforces AES-256 and SHA-256 in both Phase 1 and Phase 2 proposals.

![Figure 8.1 – IPsec Tunnels List](images/ipsec_tunnel_list.jpg)
*Figure 8.1: Overview of the `Oracle_IPsec` tunnel configuration.*

![Figure 8.2 – IPsec Tunnel Edit Configuration](images/ipsec_tunnel_edit.jpg)
*Figure 8.2: Phase 1/2 proposals emphasizing secure ciphers (AES256-SHA256).*

![Figure 8.3 – IPsec Widget Details](images/ipsec_widget_details.jpg)
*Figure 8.3: IPsec widget displaying active Rx/Tx bytes, confirming IKE SA establishment and data transfer.*

## 9. Operational Validation and Testing

Validation was executed to guarantee functional routing, NAT, and policy enforcement across all boundaries.

### 9.1 Internal LAN Egress & NAT
The internal `lan-client` successfully reached external networks, confirming outbound NAT via the SD-WAN interface.

![Figure 9.1 – AWS LAN Client SSH Access](images/aws_lan_client_ssh.jpg)
*Figure 9.1: EC2 Instance Connect session terminating on the private LAN interface (10.10.18.44).*

![Figure 9.2 – AWS LAN Client Ping and NAT](images/aws_lan_client_ping.jpg)
*Figure 9.2: Successful ICMP reachability to 8.8.8.8 and NAT verification via `curl ifconfig.me`.*

### 9.2 SSL-VPN Workforce Connectivity
Remote clients (Windows and iOS) successfully established full-tunnel VPN connections, receiving IPs from the 10.212.134.x pool and tunneling all traffic through the AWS perimeter.

![Figure 9.3 – Mobile FortiClient Connection](images/mobile_forticlient_connected.jpg)
*Figure 9.3: Authenticated iOS FortiClient session securing virtual IP 10.212.134.200.*

![Figure 9.4 – Mobile Client Public IP Check](images/mobile_whatismyip.jpg)
*Figure 9.4: Operational proof: The mobile device's egress IP matches the AWS FortiGate.*

![Figure 9.5 – Windows VPN Client Ping](images/windows_vpn_client_ping.jpg)
*Figure 9.5: Windows SSL-VPN client successfully traversing the tunnel to ping the internal FortiGate LAN gateway (10.10.18.10).*

### 9.3 IPsec Tunnel Validation
![Figure 9.6 – CLI Ping Tests for IPsec](images/ipsec_ping_tests.jpg)
*Figure 9.6: Direct ICMP validation from the FortiGate CLI, confirming bidirectional reachability across the IPsec tunnel to the remote subnet.*

## 10. Security Profiling and Threat Management (UTM)

As a Next-Generation Firewall, the FortiGate enforces Layer 7 inspection. Web Filtering is applied to outbound policies to block malicious or non-compliant traffic.

![Figure 10.1 – Web Filter Profile](images/web_filter_profile.jpg)
*Figure 10.1: Web Filter configuration actively blocking categories such as 'Security Risk' and 'Potentially Liable'.*

![Figure 10.2 – Forward Traffic Log](images/forward_traffic_log.jpg)
*Figure 10.2: Traffic logs confirming active session tracking and SNAT application for SSL-VPN users.*

![Figure 10.3 – Security Events Log](images/security_events_log.jpg)
*Figure 10.3: UTM engine operational proof: An active block generated by the Web Filter profile.*

## 11. Troubleshooting & Operational Challenges

**Problem 1: SSL Certificate Untrusted Warning on VPN Portal**
* **Root Cause:** FortiGate deployments default to a self-signed `Fortinet_Factory` certificate, which modern browsers reject (`ERR_CERT_AUTHORITY_INVALID`).
* **Resolution:** Registered a public domain, validated ownership, and issued a ZeroSSL certificate. The certificate and intermediate CA were imported into FortiOS and bound to the SSL-VPN settings, eliminating the browser warning.
* **Evidence:**
  ![Figure 11.1 – Edge Certificate Error](images/edge_cert_error.jpg)

**Problem 2: SSL-VPN and Admin GUI Port Conflict**
* **Root Cause:** Both the web-based administrative GUI and the SSL-VPN portal default to listening on TCP 443, creating a socket conflict.
* **Resolution:** The administrative GUI was migrated to TCP 8443. This isolated management access and allowed standard remote users to connect to the VPN on 443 without requiring custom port configurations in FortiClient.

## 12. Strategic Enterprise Roadmap (Future Expansion)

To scale this deployment for a mid-to-large enterprise environment, the following architectural expansions are on the roadmap:

* **Active Directory (AD/LDAP) Integration:** Transitioning from local user accounts to central identity management by integrating an AD Domain Controller.
* **FSSO Group Mapping:** Implementing Fortinet Single Sign-On (FSSO) to map AD security groups (e.g., HR, Engineering) to specific firewall policies, enabling identity-based access control.
* **Advanced Split Tunneling:** Deploying granular split tunneling where privileged groups (IT) receive full-tunnel routing, while standard users only tunnel RFC1918 traffic, optimizing AWS egress bandwidth costs.
* **Multi-Cloud Integration:** Establishing route-based IPsec VPNs between the AWS FortiGate and an Azure Virtual WAN (vWAN) to support cross-cloud disaster recovery and unified LDAP reachability.

---
*Document Version: 2.0*  
*Role: Principal Network Security Engineer*  
