# Security Hardening Best Practices

This deployment follows strict security hardening principles to ensure the FortiGate appliance and the protected AWS environment are secure.

## 1. Administrative Access Hardening
- **Port Relocation**: The Admin GUI was moved from the default port `443` to `8443` to obscure management access and prevent conflicts with SSL-VPN.
- **Restricted Access**: SSH and HTTPS management are strictly controlled. In a production environment, management should be restricted to trusted source IPs or accessible only via a management VPN.

## 2. SSL-VPN Security
- **Service Separation**: SSL-VPN operates on a dedicated port (`443`) and does not share attack surface with the administration portal.
- **Group-Based Access**: Access is controlled via specific user groups (`VPN_Users`), enforcing identity-based policies.
- **Trusted Certificates**: Replaced self-signed certificates with a trusted ZeroSSL certificate to prevent MITM attacks and ensure secure client connectivity without warnings.

## 3. Network Architecture
- **No Direct Public Exposure for LAN**: The LAN subnet (`10.10.18.0/24`) resides in a private routing domain. It has no direct route to the Internet Gateway (IGW). All traffic must pass through the FortiGate for inspection.
- **Default Gateway**: FortiGate acts as the default gateway (`10.10.18.10`) for all LAN workloads, establishing a choke point for unified threat management.

## 4. AWS Security Groups (Least Privilege)
- **Granular Rules**: AWS Security Groups are configured to follow the least-privilege principle, allowing only specific ports necessary for the FortiGate to operate (e.g., UDP 500/4500 for IPsec, TCP 443 for SSL-VPN).
- **Source/Dest Check**: Explicitly disabled on FortiGate ENIs to allow routing of transit traffic.

## 5. Firewall Policy Design
- **Explicit Policies**: Traffic is explicitly allowed based on Source, Destination, Service, and Schedule.
- **Zero Trust Posture**: The implicit default-deny rule at the bottom of the policy sequence drops any unmatched traffic.
- **NAT Control**: SNAT is selectively applied only for outbound Internet traffic. Internal traffic (e.g., VPN to LAN) operates without SNAT to preserve original client IPs for auditing.

## 6. Unified Threat Management (UTM)
- **Web Filtering**: Enabled on outbound internet policies to block malicious domains, phishing sites, and adult content, preventing compromised internal instances from communicating with C2 servers.
