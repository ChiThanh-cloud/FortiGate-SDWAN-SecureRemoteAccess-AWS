# Troubleshooting Runbook

This runbook outlines common issues encountered during the deployment and operation of the FortiGate AWS lab, along with root causes and remediation steps.

## 1. LAN client could not access Internet
- **Symptom**: Instances in the LAN subnet (`10.10.18.0/24`) cannot ping external IPs (e.g., `8.8.8.8`).
- **Root Cause**: The AWS Route Table for the LAN subnet did not have a default route pointing to the FortiGate LAN ENI, or the FortiGate firewall policy lacked SNAT.
- **Fix**: 
  1. Add `0.0.0.0/0` -> FortiGate `port3` ENI in AWS VPC route table.
  2. Disable "Source/Destination Check" on the FortiGate ENIs in AWS.
  3. Ensure FortiGate policy `LAN -> SD-WAN` has NAT enabled.
- **Validation**: Ping `8.8.8.8` from a LAN EC2 instance.

## 2. WAN2 health-check showed dead
- **Symptom**: In the SD-WAN Performance SLA dashboard, `port2` shows as red/down.
- **Root Cause**: Missing static route for WAN2 gateway, or AWS Security Group blocking ICMP echo requests from WAN2 ENI.
- **Fix**: Verify `config router static` has a route for the WAN2 subnet gateway, and AWS SG allows outbound ICMP.
- **Validation**: SLA status changes to green (UP).

## 3. SSL-VPN web portal conflict with admin GUI
- **Symptom**: Unable to load SSL-VPN login page, or it redirects to the FortiOS Admin GUI.
- **Root Cause**: Both SSL-VPN and Admin GUI were listening on default port `443`.
- **Fix**: Move Admin GUI to port `8443` (`config system global -> set admin-port 8443`). Leave SSL-VPN on `443`.
- **Validation**: Browse `https://<IP>:8443` (Admin) and `https://<IP>:443` (VPN Portal) successfully.

## 4. Invalid/self-signed certificate warning
- **Symptom**: Browsers warn that the SSL-VPN connection is not secure. FortiClient prompts to accept an invalid cert.
- **Root Cause**: FortiGate was using the default self-signed `Fortinet_Factory` certificate.
- **Fix**: Import a trusted ZeroSSL certificate with the full chain and bind it to SSL-VPN settings.
- **Validation**: Lock icon appears in browser; FortiClient connects silently.

## 5. IPsec tunnel established but traffic not passing
- **Symptom**: IKE and IPsec SAs are UP, but pings fail between sites.
- **Root Cause**: Missing phase 2 static routes or firewall policies.
- **Fix**: Add a static route pointing the remote subnet to the IPsec virtual interface. Ensure a bidirectional firewall policy allows the traffic.
- **Validation**: Continuous ping across the tunnel succeeds.

## 6. Wrong traffic selector / subnet mismatch
- **Symptom**: Phase 1 succeeds, Phase 2 fails to negotiate.
- **Root Cause**: Subnets configured in FortiGate Phase 2 do not exactly match the `leftsubnet` / `rightsubnet` in the strongSwan configuration.
- **Fix**: Align the local subnet (`10.10.18.0/24`) and remote subnet in both peers.
- **Validation**: Run `diagnose vpn ike gateway list` to verify Phase 2 SA.

## 7. Security group blocked traffic from VPN pool
- **Symptom**: SSL-VPN connects successfully, but user cannot SSH/RDP into LAN workloads.
- **Root Cause**: AWS Security Group on the target EC2 instance only allowed traffic from `10.10.18.0/24`, not the VPN pool `10.212.134.0/24`.
- **Fix**: Add an inbound rule in the LAN EC2 Security Group allowing the VPN pool CIDR.
- **Validation**: Successful SSH/RDP from a remote FortiClient.
