# Enterprise SD-WAN and Secure Remote Access using FortiGate on AWS

## 1. Executive Summary
This technical report documents the design, implementation, and verification of an Enterprise SD-WAN and Secure Remote Access solution deployed using Fortinet FortiGate on Amazon Web Services (AWS). The project integrates AWS cloud infrastructure with FortiGate's advanced routing and security capabilities to provide resilient Internet connectivity, secure SSL-VPN remote access, and site-to-site IPsec VPN integration.

## 2. Architecture & Topology
### 2.1 AWS Infrastructure
The AWS environment consists of a custom VPC, route tables, and security groups supporting the FortiGate virtual appliance and internal EC2 instances.

![Figure 2.1 – AWS EC2 Launch Configuration](images/aws_ec2_launch.jpg)

We have configured specific Route Tables in AWS to support the SD-WAN and internal LAN subnets.

![Figure 2.2 – AWS VPC Route Tables](images/aws_route_table.jpg)

The internal EC2 instance (lan-client) is protected by a dedicated Security Group allowing inbound ICMP and SSH.

![Figure 2.3 – AWS Security Group](images/aws_security_group.jpg)

### 2.2 FortiGate Configuration & Objects
The FortiGate has been configured with relevant Addresses and Subnets such as the LAN subnet and SSL-VPN tunnel address pool.

![Figure 2.4 – FortiGate Addresses & IP Pools](images/addresses.jpg)

## 3. Implementation Details

### 3.1 Firewall Policies & Security Profiles
Granular firewall policies were established to control traffic flow from LAN to Internet, SSLVPN to LAN, and SSLVPN to Internet.

![Figure 3.1 – Firewall Policies](images/firewall_policies.jpg)

A Web Filter profile was applied to actively restrict unauthorized or non-business-related domains.

![Figure 3.2 – Web Filter Profiles](images/web_filter_profile.jpg)

The ZeroSSL Certificate ensures the SSL-VPN portal operates without trust warnings for users.

![Figure 3.3 – ZeroSSL Certificate](images/certificates_detail.jpg)

### 3.2 Site-to-Site IPsec VPN
An IPsec tunnel named `Oracle_IPsec` was established linking the FortiGate to a remote strongSwan VPS endpoint.

![Figure 3.4 – IPsec Tunnels List](images/ipsec_tunnel_list.jpg)

The Phase 1 and Phase 2 configurations enforce strong encryption (AES256-SHA256).

![Figure 3.5 – IPsec Tunnel Edit Configuration](images/ipsec_tunnel_edit.jpg)

The IPsec widget confirms the active status and data transferred across the tunnel.

![Figure 3.6 – IPsec Widget Details](images/ipsec_widget_details.jpg)

## 4. Verification & Testing Outcomes

### 4.1 Internal Connectivity & Routing
The internal `lan-client` successfully connected via EC2 Instance Connect and acquired its internal IP `10.10.18.44`.

![Figure 4.1 – EC2 Instance Connect (SSH)](images/aws_lan_client_ssh.jpg)

We successfully verified outbound internet access from the `lan-client` via ICMP ping and `curl ifconfig.me`.

![Figure 4.2 – LAN Client Outbound Internet Verification](images/aws_lan_client_ping.jpg)

### 4.2 SSL-VPN Client Validation
Clients successfully obtained a public IP matching the FortiGate AWS IP when running a full tunnel VPN, confirmed here on a mobile client.

![Figure 4.3 – Mobile SSL-VPN Client IP Check](images/mobile_whatismyip.jpg)

Similarly, Windows SSL-VPN clients effectively tunneled their internet traffic out the AWS node.

![Figure 4.4 – Windows SSL-VPN Client IP Check](images/windows_whatismyip.jpg)

The Windows SSL-VPN client also successfully pinged the internal LAN gateway.

![Figure 4.5 – Windows VPN Client Ping Internal](images/windows_vpn_client_ping.jpg)

### 4.3 Site-to-Site VPN Verification
Successful ICMP ping tests were executed from the FortiGate CLI targeting the remote IPsec endpoint.

![Figure 4.6 – CLI Ping Tests for IPsec Tunnel](images/ipsec_ping_tests.jpg)

### 4.4 Security Policy Enforcement
Traffic logs demonstrated active routing and allowed connections for SSL-VPN users to external destinations.

![Figure 4.7 – Forward Traffic Log](images/forward_traffic_log.jpg)

Security events highlighted active enforcement, showing `Video/Audio` traffic blocks.

![Figure 4.8 – Security Events Log](images/security_events_log.jpg)

## 5. Conclusion
The deployment successfully integrates AWS infrastructure with FortiGate’s SD-WAN and VPN capabilities. It provides a robust edge architecture, enforces a secure zero-trust boundary for remote workers, and ensures reliable site-to-site connectivity.
