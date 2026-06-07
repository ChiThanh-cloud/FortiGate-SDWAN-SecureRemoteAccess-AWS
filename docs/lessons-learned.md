# Lessons Learned

This project provided deep insights into integrating Fortinet ecosystem technologies within an AWS cloud environment. Below are the key takeaways:

## 1. Cloud vs. On-Premises SD-WAN
- **Differences in Link Sensing**: Traditional SD-WAN relies on physical link state. In AWS, the ENI almost never goes "down" physically. SD-WAN relies entirely on SLA Health Checks (pings to reliable public IPs) to determine path viability.
- **EIP Mapping**: Failover in AWS requires careful thought. If a public Elastic IP is mapped to WAN1, and traffic fails over to WAN2, WAN2 must have its own EIP and NAT capabilities.

## 2. Routing Alignment
- **Dual-Layer Routing**: The most common source of failure in cloud firewalls is asymmetric routing or routing blackholes between the AWS VPC route tables and the FortiGate internal routing table.
- **Takeaway**: AWS route tables and FortiGate policies/routes must align perfectly. Adding a route in FortiGate is useless if the AWS subnet route table doesn't point traffic to the FortiGate ENI.

## 3. Security Group Interactions
- **VPN Pool Blocking**: Security groups on target EC2 instances must be aware of the SSL-VPN IP pool (`10.212.x.x`). A common mistake is only allowing the LAN subnet (`10.10.18.0/24`) in the SG, causing VPN users to fail access despite passing firewall policies.

## 4. IPsec Troubleshooting
- **Phase 1 vs. Phase 2**: IPsec troubleshooting is systematic. If Phase 1 is up but Phase 2 fails, it's almost always a traffic selector (subnet mismatch) issue. If Phase 2 is up but pings fail, it is usually a missing firewall policy or missing static route pointing to the IPsec virtual interface.
- **Logs**: Real-time debug logs (`diagnose debug application ike -1`) are critical for identifying strongSwan to FortiGate negotiation errors.

## 5. Port Conflicts
- **Admin GUI vs. VPN**: FortiOS defaults both the HTTPS management GUI and the SSL-VPN portal to port 443. This conflict must be resolved early (by moving Admin to 8443) to prevent portal access issues.

## 6. The Value of Validation
- Implementing a firewall policy is only 50% of the job. Validating it via logs and testing (UTM blocking, failover pings) is required to prove the infrastructure works. Screenshots and logs act as the definitive proof of execution.
