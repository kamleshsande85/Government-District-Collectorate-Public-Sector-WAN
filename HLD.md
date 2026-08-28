**PROJECT 3: Government District Collectorate / Public Sector WAN Architecture**

**Document Version:** 1.0

**Target Platform:** Cisco IOS-XE / FortiGate NGFW / GNS3 / Cisco Packet Tracer

---

## 1. Executive Summary & Critical Requirements

Government District Collectorate (Zila Panchayat/DC Office) Networks WAN-heavy aur high-security environments hote hain. Ye central headquarter ko multiple sub-divisional offices (Tehsils, Blocks, aur Municipalities) se connect karte hain. Security compliance, strict access controls, and encrypted site-to-site communication yahan top priority hoti hain.

### Key Functional & Compliance Requirements:

1. **Secure WAN Interconnectivity:** HQ (District Collectorate) aur Remote Branches (Tehsils/Blocks) ke beech public internet over **Hub-and-Spoke IPsec VPN Tunnels**.


2. **Strict Access Isolation:** Public Citizen Kiosks, Revenue/Tax Records, and IAS Officer/Administrative Workstations ko Access Control Lists (ACLs) se completely separate rakhna.


3. **Centralized Logging & Audit:** Mandatory **Syslog & SNMPv3 logging** for security audits.


4. **Gateway & Edge Protection:** Dual ISP connectivity with **FortiGate NGFW** for deep packet inspection, IPS, and Web Filtering.



---

## 2. Network Topology & WAN Architecture

Central Collectorate HQ ko **Hub** banakar saare Tehsil/Block offices (**Spokes**) ko encrypted IPsec VPNs se joda gaya hai:

```text
                        [ Central State Data Center / Public WAN ]
                                            │
                         [ Dual FortiGate NGFW (HA Pair) ]
                                            │
                 ┌──────────────────────────┴──────────────────────────┐
                 │                                                     │
          [ HQ Core Switch 1 ] ═════════ EtherChannel ═════════ [ HQ Core Switch 2 ]
          (Active HSRP Master)         (802.1AX)             (Standby HSRP Slave)
                 │                                                     │
                 └──────────────────────────┬──────────────────────────┘
                                            │
                       [ IPsec Encrypted WAN Overlay Tunnels ]
                                            │
            ┌───────────────────────────────┼───────────────────────────────┐
            │                               │                               │
    [ Tehsil-1 Router ]             [ Tehsil-2 Router ]             [ Block-1 Router ]
      (Spoke Branch)                  (Spoke Branch)                  (Spoke Branch)
            │                               │                               │
     (Subnet 10.30.1.0/24)          (Subnet 10.30.2.0/24)          (Subnet 10.30.3.0/24)

```

---

## 3. Comprehensive IP Addressing & Subnetting Matrix

**Supernet Base IP:** `10.30.0.0/16` (RFC 1918 Private IP Space)

| Location / Zone | VLAN ID | IP Subnet Space | Subnet Mask | Virtual Gateway (HSRP)

 | Core Switch 1 (Active)

 | Core Switch 2 (Standby)

 | WAN / Security Profile |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **HQ Management & Servers** | VLAN 300 | `10.30.10.0/24` | `255.255.255.0` | `10.30.10.1` | `10.30.10.2` | `10.30.10.3` | Internal Syslog, Database & Active Directory |
| **IAS/Collectorate Admin** | VLAN 310 | `10.30.11.0/24` | `255.255.255.0` | `10.30.11.1` | `10.30.11.2` | `10.30.11.3` | Full Access to Revenue & Confidential Files |
| **Public Service Desk / Seva Kendra** | VLAN 320 | `10.30.12.0/23` | `255.255.254.0` | `10.30.12.1` | `10.30.12.2` | `10.30.12.3` | Restricted Public Portal Access Only |
| **Public Wi-Fi (Kiosk Zone)** | VLAN 330 | `10.30.14.0/23` | `255.255.254.0` | `10.30.14.1` | `10.30.14.2` | `10.30.14.3` | Isolated Internet Access Only |
| **Tehsil-1 Remote Branch** | Branch N/A | `10.30.21.0/24` | `255.255.255.0` | `10.30.21.1` | N/A (Spoke Router) | N/A | Connected via IPsec Tunnel |
| **Tehsil-2 Remote Branch** | Branch N/A | `10.30.22.0/24` | `255.255.255.0` | `10.30.22.1` | N/A (Spoke Router) | N/A | Connected via IPsec Tunnel |

---

## 4. Security Policy & Access Control Matrix

### A. Extended Access Control List (ACL) Policies

1. **Public Kiosks (VLAN 330) & Service Desk (VLAN 320):** Cannot access Admin VLAN (`10.30.11.0/24`) or Central Database Servers (`10.30.10.0/24`).


2. **Remote Branch Offices:** Tehsil offices can access land records database (`10.30.10.50`), but inter-tehsil communication is blocked at the HQ Firewall.



---

## 5. Cisco IOS Configuration Blueprints

### A. Central HQ Core Switch 1 (Primary Active HSRP & Route Hub)

```cisco
! --- Hostname & VLAN Definitions ---
hostname Core-SW-GovHQ-01
ip routing
vlan 300,310,320,330
vlan 300
 name HQ_Management_Servers
vlan 310
 name Administrative_IAS
vlan 320
 name Public_Service_Desk
vlan 330
 name Public_Kiosk_WiFi

! --- HSRP Gateway Redundancy Configuration ---
interface Vlan300
 description Management_SVI
 ip address 10.30.10.2 255.255.255.0
 standby version 2
 standby 300 ip 10.30.10.1
 standby 300 priority 150
 standby 300 preempt

interface Vlan310
 description Admin_SVI
 ip address 10.30.11.2 255.255.255.0
 standby version 2
 standby 310 ip 10.30.11.1
 standby 310 priority 150
 standby 310 preempt

interface Vlan320
 description Public_Service_Desk_SVI
 ip address 10.30.12.2 255.255.254.0
 standby version 2
 standby 320 ip 10.30.12.1
 standby 320 priority 150
 standby 320 preempt
 ip access-group LOCK_PUBLIC_ACCESS in

interface Vlan330
 description Kiosk_WiFi_SVI
 ip address 10.30.14.2 255.255.254.0
 standby version 2
 standby 330 ip 10.30.14.1
 standby 330 priority 150
 standby 330 preempt
 ip access-group ISOLATE_KIOSK_TRAFFIC in

! --- Access Control Lists ---
ip access-list extended LOCK_PUBLIC_ACCESS
 permit ip 10.30.12.0 0.0.1.255 host 10.30.10.50
 deny ip 10.30.12.0 0.0.1.255 10.30.11.0 0.0.0.255
 permit ip any any

ip access-list extended ISOLATE_KIOSK_TRAFFIC
 deny ip 10.30.14.0 0.0.1.255 10.30.0.0 0.0.255.255
 permit ip any any

! --- Dynamic OSPF Routing Over VPN Overlay ---
router ospf 1
 router-id 10.30.10.2
 network 10.30.10.0 0.0.0.255 area 0
 network 10.30.11.0 0.0.0.255 area 0
 network 10.30.12.0 0.0.1.255 area 0
 network 10.30.21.0 0.0.0.255 area 10
 network 10.30.22.0 0.0.0.255 area 20

! --- Central Logging Configuration ---
logging host 10.30.10.100
logging trap informations

```

---

### B. Remote Tehsil Branch Router (Spoke Site-to-Site IPsec VPN Config)

```cisco
hostname Tehsil-01-Router

! --- WAN Interface & IPsec Tunnel Configuration ---
interface GigabitEthernet0/0/0
 description Public_WAN_Interface
 ip address 203.0.113.10 255.255.255.252
 crypto map HQ_IPSEC_MAP

interface GigabitEthernet0/0/1
 description Local_Tehsil_LAN
 ip address 10.30.21.1 255.255.255.0

! --- IPsec Phase 1 (IKEv2 / IKEv1) ---
crypto isakmp policy 10
 encr aes 256
 authentication pre-share
 group 14
 hash sha256
 lifetime 86400
crypto isakmp key GovSecurePass2026! address 198.51.100.1

! --- IPsec Phase 2 (Transform Set & Crypto Map) ---
crypto ipsec transform-set GOV_TRANSFORM esp-aes 256 esp-sha256-hmac

crypto map HQ_IPSEC_MAP 10 ipsec-isakmp
 set peer 198.51.100.1
 set transform-set GOV_TRANSFORM
 match address VPN_TRAFFIC_ACL

ip access-list extended VPN_TRAFFIC_ACL
 permit ip 10.30.21.0 0.0.0.255 10.30.0.0 0.0.255.255

```

---

## 6. Verification & Audit Test Plan

1. **IPsec Tunnel Connectivity:**
* Run `show crypto ipsec sa` on the Tehsil Router. Verify `pkts encaps` and `pkts decaps` are incrementing.




2. **Security & ACL Audit:**
* Public Kiosk (`10.30.14.5`) se IAS Admin PC (`10.30.11.5`) ko ping karein. **FAIL (Destination Unreachable)** hona chahiye.


* Tehsil-1 Branch (`10.30.21.5`) se Central Server (`10.30.10.50`) ko ping karein. **SUCCESS** hona chahiye.




3. **Audit Log Check:**
* Run `show logging` on Core Switch to confirm syslog messages are sent to Central Audit Server (`10.30.10.100`).





---

**Project 3 (Government District WAN Architecture) Complete!** 🚀

Batao, ab **Project 4: 🏬 E-Commerce Fulfillment Center / Central Warehouse (Like Blinkit/Amazon)** par move karein?
