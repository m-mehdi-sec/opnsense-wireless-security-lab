# Lab Documentation – OPNsense Wireless Security Lab

## 1. Objective

The objective of this lab was to simulate and validate wireless network security controls using OPNsense in a Hyper-V environment.

The lab focused on:

- Simulated employee WLAN
- Simulated guest WLAN
- Captive Portal authentication
- Guest network isolation
- Firewall rule validation
- Suricata IDS detection
- Nmap-based scan testing
- Troubleshooting related to virtual network behavior

---

## 2. Network Setup

### OPNsense Interfaces

| Interface | IP Address | Purpose |
|---|---|---|
| LAN | 192.168.10.1/24 | Management / existing lab network |
| WLAN | 192.168.40.1/24 | Simulated employee wireless network |
| Guest_WLAN | 192.168.50.1/24 | Simulated guest wireless network |

### Hyper-V Switches

| Switch Name | Type | Connected To |
|---|---|---|
| WAN-Switch | Existing lab connection | OPNsense WAN / management path |
| WLAN_Switch | Internal | OPNsense WLAN + Windows WLAN client |
| Guest_Switch | Internal | OPNsense Guest_WLAN + Windows guest client |

---

## 3. WLAN Interface Configuration

A Hyper-V internal switch named `WLAN_Switch` was created and attached to OPNsense as an additional network adapter.

The new adapter was assigned in OPNsense and configured as:

```text
Interface: WLAN
IPv4 address: 192.168.40.1/24
```

DHCP was configured for WLAN:

```text
DHCP range: 192.168.40.100 - 192.168.40.200
Gateway: 192.168.40.1
```

The Windows WLAN client received:

```text
IPv4 address: 192.168.40.101
Default gateway: 192.168.40.1
```

This confirmed that the simulated employee WLAN network was working.

---

## 4. Guest_WLAN Interface Configuration

A second Hyper-V internal switch named `Guest_Switch` was created and attached to OPNsense as another network adapter.

The new adapter was assigned in OPNsense and configured as:

```text
Interface: Guest_WLAN
IPv4 address: 192.168.50.1/24
```

DHCP was configured for Guest_WLAN:

```text
DHCP range: 192.168.50.100 - 192.168.50.200
Gateway: 192.168.50.1
```

The Windows guest client received:

```text
IPv4 address: 192.168.50.154
Default gateway: 192.168.50.1
```

This confirmed that the simulated guest wireless network was working.

---

## 5. Firewall Rules

## 5.1 WLAN Rules

The WLAN interface was configured with two rules.

### Rule 1 – Allow WLAN Users to Internet

| Setting | Value |
|---|---|
| Action | Pass |
| Interface | WLAN |
| Protocol | any |
| Source | WLAN net |
| Destination | any |
| Description | Allow WLAN users to Internet |

This rule allows clients from the WLAN network to reach external networks after authentication.

### Rule 2 – Block Unauthorized Access to WLAN

| Setting | Value |
|---|---|
| Action | Block |
| Interface | WLAN |
| Protocol | any |
| Source | any |
| Destination | WLAN net |
| Description | Block unauthorized access to WLAN |

This rule simulates protection against unauthorized access attempts toward the WLAN network.

---

## 5.2 WLAN Rule Validation

The WLAN rule was validated using:

```text
Firewall → Diagnostics → States
```

Active states were visible from the WLAN client.

Observed result:

```text
Source: 192.168.40.101
Rule: Allow WLAN users to Internet
State: ESTABLISHED
```

This confirmed that the WLAN firewall rule was matching client traffic correctly.

---

## 5.3 Guest_WLAN Rules

The Guest_WLAN interface was configured with two rules.

### Rule 1 – Block Guest to WLAN

| Setting | Value |
|---|---|
| Action | Block |
| Interface | Guest_WLAN |
| Protocol | any |
| Source | Guest_WLAN net |
| Destination | WLAN net |
| Description | Block Guest to WLAN |
| Logging | Enabled |

This rule blocks guest clients from reaching the employee WLAN network.

### Rule 2 – Allow Guest to Internet

| Setting | Value |
|---|---|
| Action | Pass |
| Interface | Guest_WLAN |
| Protocol | any |
| Source | Guest_WLAN net |
| Destination | any |
| Description | Allow Guest to Internet |

This rule allows guest clients to reach external networks.

### Rule Order

The final Guest_WLAN rule order was:

```text
1. Block Guest to WLAN
2. Allow Guest to Internet
```

This order is important because OPNsense uses first match logic.

---

## 6. Captive Portal Authentication

## 6.1 Local Users

Two local users were created in OPNsense.

| Username | Purpose |
|---|---|
| HR_Anstalld | Simulated employee / HR user |
| Gast | Simulated guest user |

The employee account was used to test authenticated WLAN access.

---

## 6.2 Captive Portal Zone

Captive Portal was enabled on the WLAN interface.

| Setting | Value |
|---|---|
| Interface | WLAN |
| Authentication | Local Database |
| Description | WLAN authentication zone |
| SSL Certificate | None |

The portal was accessed from the WLAN client using:

```text
http://192.168.40.1:8000
```

---

## 6.3 Failed Authentication Test

Incorrect credentials were tested first.

```text
Username: Fel_Anvandare
Password: 1234
```

Result:

```text
authentication failed
```

This confirmed that unauthorized login attempts were denied.

---

## 6.4 Successful Authentication Test

The correct employee account was then tested.

```text
Username: HR_Anstalld
Password: HR2025!
```

After successful authentication, the portal displayed a logout button.

This confirmed that the user session was active.

---

## 6.5 Internet Access After Login

After successful authentication, the WLAN client opened:

```text
http://neverssl.com
```

The page loaded successfully.

This confirmed that authenticated WLAN users were allowed network access.

---

## 6.6 Captive Portal HTTP/HTTPS Behavior

During testing, HTTPS pages did not always redirect correctly to the Captive Portal login page.

This happened because HTTPS traffic is encrypted and cannot be redirected in the same way as HTTP traffic.

For validation, HTTP was used instead:

```text
http://neverssl.com
```

The portal could also be accessed directly:

```text
http://192.168.40.1:8000
```

---

## 7. Suricata IDS Testing

## 7.1 IDS Configuration

Suricata was enabled on the WLAN interface.

| Setting | Value |
|---|---|
| IDS Status | Enabled |
| Capture Mode | PCAP live mode (IDS) |
| Interface | WLAN |
| Ruleset | ET Open |

Relevant rule categories used:

```text
emerging-scan
emerging-icmp
emerging-dos
```

IDS mode was used to detect and log suspicious traffic without blocking it.

---

## 7.2 Nmap Installation

Nmap was installed on the Windows WLAN client.

Validation command:

```cmd
nmap --version
```

Observed result:

```text
Nmap version 7.99
```

This confirmed that Nmap was installed correctly.

---

## 7.3 Nmap Scan Test

A scan was launched from the WLAN client.

```cmd
nmap 192.168.10.1
```

Test details:

| Field | Value |
|---|---|
| Source | 192.168.40.101 |
| Destination | 192.168.10.1 |
| Interface monitored | WLAN |

---

## 7.4 IDS Alert Result

Suricata generated alerts for the scan.

| Field | Result |
|---|---|
| Alert | ET SCAN Possible Nmap |
| Source | 192.168.40.101 |
| Destination | 192.168.10.1 |
| Interface | WLAN |
| Action | allowed |

The scan was detected and logged.

Because Suricata was running in IDS mode, the traffic was not blocked.

---

## 8. Guest_WLAN Isolation Test

## 8.1 Guest Client Test

The guest client was connected to `Guest_Switch`.

Client configuration:

```text
IPv4 address: 192.168.50.154
Default gateway: 192.168.50.1
```

The guest client attempted to reach the WLAN gateway.

```cmd
ping 192.168.40.1
```

Result:

```text
Request timed out
Request timed out
Request timed out
Request timed out

Packets: Sent = 4, Received = 0, Lost = 4 (100% loss)
```

This confirmed that Guest_WLAN could not reach WLAN.

---

## 8.2 Firewall Log Validation

Firewall logging was enabled on the rule:

```text
Block Guest to WLAN
```

The block was confirmed in:

```text
Firewall → Log Files → Live View
```

Observed log result:

| Field | Result |
|---|---|
| Interface | Guest_WLAN |
| Direction | In |
| Protocol | ICMP |
| Source | 192.168.50.154 |
| Destination | 192.168.40.1 |
| Action | block |
| Label | Block Guest to WLAN |

This confirmed that the firewall rule successfully blocked guest traffic to the WLAN network.

---

## 9. Troubleshooting Notes

## 9.1 Host Machine Affected by Internal Switches

During testing, the host machine was affected by the simulated lab networks.

This happened because Hyper-V internal switches create virtual adapters on the host system.

The affected adapters were:

```text
vEthernet (WLAN_Switch)
vEthernet (Guest_Switch)
```

Because the host was connected to these simulated networks, it was also affected by Captive Portal behavior.

This caused some websites on the host to fail or behave inconsistently.

---

## 9.2 Fix

The host adapters for the simulated lab networks were disabled:

```text
vEthernet (WLAN_Switch) → Disabled
vEthernet (Guest_Switch) → Disabled
```

The host kept its normal connectivity through:

```text
vEthernet (WAN-Switch)
vEthernet (Default Switch)
```

After disabling the WLAN and Guest virtual adapters on the host, normal web access worked again.

This separated the real host from the simulated lab networks.

---

## 9.3 Captive Portal Redirect Issue

Before authentication, some web pages failed to load correctly.

The reason was that HTTPS traffic cannot be redirected cleanly to a Captive Portal login page.

The working test methods were:

```text
http://neverssl.com
```

and:

```text
http://192.168.40.1:8000
```

---

## 9.4 Firewall State Reset

After changing firewall rules and Captive Portal settings, old states could remain active.

To avoid testing with old connections, the state table was reset:

```text
Firewall → Diagnostics → States → Actions → Reset state table
```

This helped ensure that new firewall and authentication behavior was tested cleanly.

---

## 10. Final Validation

| Test | Expected Result | Actual Result | Status |
|---|---|---|---|
| WLAN client DHCP | Client receives 192.168.40.x | 192.168.40.101 | Passed |
| Guest client DHCP | Client receives 192.168.50.x | 192.168.50.154 | Passed |
| Failed portal login | Access denied | authentication failed | Passed |
| Successful portal login | User authenticated | Logout button shown | Passed |
| Internet after login | Website loads | NeverSSL loaded | Passed |
| Nmap scan detection | IDS alert generated | ET SCAN Possible Nmap | Passed |
| Guest to WLAN access | Blocked | 100% packet loss | Passed |
| Firewall log validation | Block log visible | Block Guest to WLAN matched | Passed |

---

## 11. Technical Summary

The lab successfully validated the simulated wireless security design.

The WLAN client was able to authenticate through Captive Portal and access the network after successful login.

The guest client was isolated from the employee WLAN network, and the firewall log confirmed that Guest_WLAN traffic toward WLAN was blocked.

Suricata detected Nmap scanning activity from the WLAN client and generated IDS alerts.

The lab also showed the importance of managing Hyper-V internal switch adapters on the host machine to prevent the host from being unintentionally affected by lab network behavior.

---

## 12. Conclusion

This lab demonstrated how a simulated wireless company network can be secured using OPNsense, Hyper-V, Captive Portal authentication, firewall rules, guest network isolation and Suricata IDS monitoring.

The WLAN client was able to authenticate through Captive Portal, and incorrect login attempts were denied. After successful authentication, the client was allowed network access, which confirmed that the simulated wireless authentication worked as expected.

The Guest_WLAN network was successfully isolated from the employee WLAN network. The guest client could not reach the WLAN gateway, and the firewall log confirmed that the traffic was blocked by the rule `Block Guest to WLAN`.

Suricata also detected the Nmap scan from the WLAN client and generated `ET SCAN Possible Nmap` alerts. Since Suricata was running in IDS mode, the traffic was detected and logged but not blocked.

Overall, the lab showed how authentication, segmentation, firewall rules and IDS monitoring can work together to improve wireless network security in a virtual lab environment.
