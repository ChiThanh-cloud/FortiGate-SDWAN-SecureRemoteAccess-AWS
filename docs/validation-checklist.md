# Validation Checklist

This checklist is used to validate the operational status of the FortiGate SD-WAN and Secure Remote Access deployment on AWS.

## 1. AWS Network Validation
- [x] **VPC State**: VPC `10.10.0.0/16` is available.
- [x] **Subnets**: 
  - WAN1: `10.10.16.0/24`
  - WAN2: `10.10.17.0/24`
  - LAN: `10.10.18.0/24`
- [x] **Route Tables**: 
  - LAN subnet route table directs `0.0.0.0/0` to FortiGate port3 ENI (`10.10.18.10`).
- [x] **Security Groups**: 
  - FortiGate interfaces allow necessary traffic while preventing unauthorized public access.

## 2. FortiGate Interface Validation
- [x] **port1 (WAN1)**: `10.10.16.10` - External facing, pingable from allowed IPs.
- [x] **port2 (WAN2)**: `10.10.17.10` - External facing, secondary link active.
- [x] **port3 (LAN)**: `10.10.18.10` - Internal facing, reachable from LAN workloads.

## 3. SD-WAN Validation
- [x] **SD-WAN Members**: `port1` and `port2` are members of the virtual-wan-link.
- [x] **SLA Health Check**: Ping to `8.8.8.8` configured on both WANs. Status is UP.
- [x] **SD-WAN Rules**: Traffic steers correctly according to defined volume/latency logic.
- [x] **Failover Test**: Taking down WAN1 successfully moves all sessions to WAN2.
  - exact convergence time not measured
  - zero packet loss not claimed
  - brief timeout observed during transition

## 4. SSL-VPN Validation
- [x] **Portal Access**: Web portal accessible via `https://<PUBLIC_IP>:443` without conflicting with admin GUI (port 8443).
- [x] **Certificate**: ZeroSSL valid certificate is presented (no warnings).
- [x] **Authentication**: Validated user login.
- [x] **Tunnel Mode**: FortiClient assigns IP from `10.212.134.0/24` pool.
- [x] **Split Tunneling**: VPN users can reach `10.10.18.0/24` (LAN) while general internet goes local, OR full tunnel operates smoothly.

## 5. IPsec Validation
- [x] **Phase 1 (IKE)**: IKE SA established between FortiGate and strongSwan VPS.
- [x] **Phase 2 (CHILD SA)**: IPsec SA is active.
- [x] **Traffic Selectors**: Matched correctly between AWS LAN (`10.10.18.0/24`) and remote site.
- [x] **Reachability**: Ping succeeds across the tunnel.

## 6. Firewall Policy Validation
- [x] **LAN to Internet**: Traffic permitted with source NAT.
- [x] **SSL-VPN to LAN**: Traffic permitted without SNAT.
- [x] **IPsec to LAN**: Bidirectional traffic permitted.
- [x] **Default Deny**: Implicit deny at the bottom of the policy list blocks unwanted traffic.

## 7. UTM / Web Filter Validation
- [x] **Web Filter Profile**: Applied to LAN-to-Internet policy.
- [x] **Category Block**: Malicious/Adult content is blocked.
- [x] **Log Validation**: FortiGate Forward Traffic logs show action "blocked" for prohibited categories.
