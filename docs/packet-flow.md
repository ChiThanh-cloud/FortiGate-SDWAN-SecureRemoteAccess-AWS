# Packet Flow Architecture

This document maps the journey of traffic across various scenarios within the AWS FortiGate deployment.

## 1. LAN Client to Internet
1. **Source**: LAN EC2 instance (`10.10.18.x`).
2. **AWS Routing**: Traffic sent to default gateway (FortiGate `port3` ENI - `10.10.18.10`).
3. **FortiGate Ingress**: Packet arrives on `port3` (LAN).
4. **Routing & SD-WAN**: FortiGate checks routing table and SD-WAN rules. Selects `virtual-wan-link` (e.g., `port1` / WAN1).
5. **Firewall Policy**: Matches `LAN_to_Internet` policy. UTM (Web Filter) inspection occurs.
6. **NAT**: Source IP is translated (SNAT) to `port1`'s IP (`10.10.16.10`).
7. **AWS Egress**: Packet leaves via AWS Internet Gateway (IGW).

## 2. SSL-VPN User to LAN Server
1. **Source**: Remote user running FortiClient.
2. **Ingress**: Encrypted ESP/TLS traffic arrives at FortiGate `port1` public IP on port `443`.
3. **Decapsulation**: FortiGate terminates SSL-VPN, extracts inner packet (Source: VPN pool IP `10.212.134.x`).
4. **Firewall Policy**: Matches `SSLVPN_to_LAN` policy.
5. **Routing**: Inner packet destined for `10.10.18.x` is routed out `port3`. No SNAT is applied.
6. **Delivery**: Packet reaches target LAN instance.

## 3. SSL-VPN User to Internet (Full Tunnel)
1. **Source**: Remote user.
2. **Ingress**: Traffic arrives at FortiGate SSL-VPN interface.
3. **Decapsulation**: Inner packet destined for `8.8.8.8`.
4. **Firewall Policy**: Matches `SSLVPN_to_Internet` policy.
5. **NAT**: Traffic is NATed behind the active WAN interface (SD-WAN).
6. **Egress**: Packet sent to Internet.

## 4. Remote Branch (strongSwan) to AWS LAN
1. **Source**: Remote site network.
2. **Ingress**: IPsec ESP traffic arrives at FortiGate WAN interface.
3. **Decapsulation**: FortiGate decrypts via IKE SA/Child SA, yielding inner packet destined for `10.10.18.0/24`.
4. **Routing**: Packet logically enters via IPsec virtual interface.
5. **Firewall Policy**: Matches `VPN_to_LAN` policy.
6. **Delivery**: Packet routed out `port3` to AWS LAN.

## 5. UTM / Web Filter Inspection Flow
1. Traffic matching a policy with a Web Filter profile is intercepted by the IPS engine.
2. FortiGate performs SNI inspection (or deep SSL inspection if configured) to identify the domain.
3. The domain is checked against FortiGuard category databases.
4. If categorized as allowed, traffic passes. If blocked, a block replacement page is served or the TCP connection is reset. Logs are generated.
