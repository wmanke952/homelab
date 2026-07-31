# VLAN and Firewall Policy

## Purpose

This document defines the logical network segmentation and the conceptual firewall policy for the HomeLab.

The policy is designed around the following principles:

- deny inter-VLAN traffic by default;
- allow only explicitly required communication;
- isolate untrusted and constrained devices;
- keep management interfaces separated from user and IoT networks;
- centralize routing, DHCP, DNS and firewall enforcement in OPNsense;
- keep operational secrets and exact host assignments outside the public repository.

> This is a sanitized architecture document intended for public documentation.  
> Exact host addresses, credentials, VPN settings, certificates, NAT rules and exported device configurations are maintained privately.

---

## Network Segmentation

| VLAN | Name | Subnet | Gateway | Primary purpose |
| ---: | --- | --- | --- | --- |
| 10 | Management | `192.168.10.0/24` | `192.168.10.1` | Proxmox, OPNsense, switch and access point management |
| 20 | Servers | `192.168.20.0/24` | `192.168.20.1` | Ubuntu Server, Home Assistant, MQTT and future NAS |
| 30 | Clients | `192.168.30.0/24` | `192.168.30.1` | Desktop computers, notebooks and trusted Wi-Fi devices |
| 40 | IoT | `192.168.40.0/24` | `192.168.40.1` | Smart home and constrained devices |
| 50 | Cameras | `192.168.50.0/24` | `192.168.50.1` | PoE cameras and video devices |
| 60 | Guest | `192.168.60.0/24` | `192.168.60.1` | Guest Wi-Fi with internet-only access |

All VLAN gateways are hosted by OPNsense.

---

## Security Model

The default policy is:

```text
Block by default
        +
Allow only required traffic
```

OPNsense performs:

- inter-VLAN routing;
- stateful firewall filtering;
- DHCP services;
- DNS forwarding or filtering;
- VPN termination;
- logging and traffic visibility.

Traffic between devices in the same VLAN does not require routing through OPNsense. Traffic between different VLANs must pass through OPNsense and match an explicit firewall rule.

---

## Firewall Aliases

Host-specific addresses should be represented by aliases rather than hard-coded throughout the rule set.

Recommended aliases:

| Alias | Purpose |
| --- | --- |
| `HOME_ASSISTANT` | Home Assistant host in VLAN 20 |
| `MQTT_BROKER` | MQTT broker in VLAN 20 |
| `NVR_HOST` | Future video recording server |
| `ADMIN_CLIENTS` | Authorized management devices |
| `INTERNAL_NETWORKS` | All internal VLAN subnets |
| `DNS_SERVERS` | Approved DNS resolvers |
| `NTP_SERVERS` | Approved NTP servers |

This keeps firewall rules readable and makes address changes easier to manage.

---

## Inter-VLAN Access Matrix

| Source | Destination | Policy | Purpose |
| --- | --- | --- | --- |
| Management | All internal VLANs | Allow | Infrastructure administration |
| Servers | Internet | Allow, restricted when possible | Updates and required external services |
| Servers | Management | Block by default | Prevent server-initiated access to infrastructure |
| Clients | Servers | Allow selected services | Home Assistant, personal services and future NAS |
| Clients | Management | Block | Protect infrastructure interfaces |
| Clients | IoT | Block by default | Limit direct access to constrained devices |
| IoT | Home Assistant | Allow required services only | Smart home integration |
| IoT | MQTT broker | Allow MQTT only | Telemetry and automation messaging |
| IoT | Other internal networks | Block | Device isolation |
| Cameras | NVR / Home Assistant | Allow required services only | Recording and integration |
| Cameras | Internet | Block by default | Prevent unnecessary cloud access |
| Cameras | Other internal networks | Block | Camera isolation |
| Guest | Internet | Allow | Guest connectivity |
| Guest | Internal networks | Block | Full internal isolation |

---

## VLAN 10 — Management Policy

### Allowed

- authorized administrator devices to Proxmox;
- authorized administrator devices to OPNsense;
- authorized administrator devices to the managed switch;
- authorized administrator devices to the access point;
- controlled administration access to servers and other VLANs.

### Blocked

- direct access from Clients, IoT, Cameras and Guest networks;
- unsolicited connections initiated by lower-trust VLANs.

### Recommendation

Only trusted devices included in the `ADMIN_CLIENTS` alias should be allowed to access management interfaces.

---

## VLAN 20 — Servers Policy

### Systems

- Ubuntu Server;
- Home Assistant OS;
- MQTT broker;
- future NAS;
- future monitoring and personal services.

### Allowed

- selected access from VLAN 30 Clients;
- IoT access only to Home Assistant and MQTT;
- Camera access only to the recording or integration services;
- DNS, NTP and required internet access.

### Blocked

- unnecessary access to Management;
- unrestricted inbound access from IoT, Cameras and Guest;
- direct public exposure unless explicitly designed and documented.

---

## VLAN 30 — Clients Policy

### Allowed

- internet access;
- Home Assistant;
- approved personal services;
- future NAS services;
- DNS and NTP.

### Blocked

- Proxmox, OPNsense, switch and access point management interfaces;
- unrestricted access to IoT and Camera networks;
- Guest network access.

Administration should be performed only from devices explicitly included in `ADMIN_CLIENTS`.

---

## VLAN 40 — IoT Policy

The `Home-IoT` SSID is mapped to VLAN 40.

### Allowed

- DHCP from OPNsense;
- DNS through approved resolvers;
- NTP;
- required internet access for devices that depend on cloud services;
- Home Assistant through explicitly required ports and protocols;
- MQTT to the `MQTT_BROKER` alias.

### Blocked

- Management VLAN;
- Client VLAN;
- Camera VLAN;
- unrestricted access to the Servers VLAN;
- access to arbitrary internal destinations.

### Communication with Home Assistant

```text
IoT device — VLAN 40
        │
        ▼
OPNsense firewall
        │
        ▼
Home Assistant — VLAN 20
```

The firewall should permit only the traffic required by each integration.

Because OPNsense is stateful, replies to connections initiated by Home Assistant are automatically allowed.

---

## VLAN 50 — Cameras Policy

### Allowed

- DHCP;
- DNS and NTP when required;
- communication with the future `NVR_HOST`;
- communication with Home Assistant when required by an integration.

### Blocked

- internet access by default;
- Management VLAN;
- Client VLAN;
- IoT VLAN;
- unrelated servers.

Internet access should be added only for devices that cannot operate without it.

---

## VLAN 60 — Guest Policy

### Allowed

- DHCP;
- DNS;
- internet access.

### Blocked

- all internal VLANs;
- management interfaces;
- local services;
- communication with other guest clients when client isolation is available.

The `Home-Guest` SSID is mapped to VLAN 60.

---

## Wireless VLAN Mapping

The TP-Link Omada EAP650 uses a PoE+ 802.1Q trunk connection to the managed switch.

| Access point function | VLAN |
| --- | ---: |
| Access point management | 10 |
| `Home` SSID | 30 |
| `Home-IoT` SSID | 40 |
| `Home-Guest` SSID | 60 |

The access point does not perform inter-VLAN routing. It maps wireless clients to VLANs and forwards tagged traffic to the managed switch.

---

## Core Service Policy

| Service | Source | Destination | Policy |
| --- | --- | --- | --- |
| DHCP | Each VLAN | OPNsense | Allow |
| DNS | Internal VLANs | Approved DNS service | Allow |
| NTP | Internal VLANs | Approved NTP service | Allow |
| MQTT | IoT | `MQTT_BROKER` | Allow required MQTT transport |
| Home Assistant | Clients and authorized IoT | `HOME_ASSISTANT` | Allow required integration traffic |
| Management | `ADMIN_CLIENTS` | Management devices | Allow |
| Internet | Guest | WAN | Allow |
| Internet | Cameras | WAN | Block by default |

---

## Discovery and Multicast

Some smart home integrations rely on multicast or broadcast discovery protocols that do not cross VLAN boundaries automatically.

Examples include:

- mDNS;
- SSDP;
- multicast discovery.

Recommended approach:

1. prefer direct IP-based integrations and DHCP reservations;
2. enable multicast reflection only when required;
3. limit reflection to VLAN 20 and VLAN 40;
4. avoid forwarding multicast to Guest or Camera networks unless strictly necessary.

Conceptual flow:

```text
VLAN 20 — Servers
        │
   controlled mDNS
        │
VLAN 40 — IoT
```

---

## Logging and Monitoring

OPNsense should log:

- blocked inter-VLAN traffic;
- denied access to Management;
- denied internet access from Cameras;
- unexpected IoT communication;
- firewall rule changes;
- VPN authentication events.

Logging should be reviewed before broadening any rule. A blocked connection should not automatically result in a permissive rule.

---

## Public Repository Safety

The following information is intentionally excluded:

- usernames and passwords;
- API tokens;
- MQTT credentials;
- VPN keys and peer addresses;
- public IP addresses;
- DNS provider credentials;
- certificates and private keys;
- exact host reservations;
- port forwarding and NAT rules;
- exported OPNsense, switch or access point configurations.

Only conceptual architecture and sanitized private addressing are documented publicly.

---

## Validation Checklist

- [ ] Each VLAN receives an address from the correct DHCP scope
- [ ] Each VLAN uses OPNsense as its gateway
- [ ] Guest devices cannot reach internal networks
- [ ] IoT devices cannot access Management
- [ ] IoT devices can reach only approved Home Assistant and MQTT services
- [ ] Cameras cannot access the internet by default
- [ ] Clients can access approved services in VLAN 20
- [ ] Only authorized administrator devices can access VLAN 10
- [ ] Multicast reflection is disabled unless required
- [ ] Firewall logs show expected blocks without disrupting required services
- [ ] No credentials or operational exports are committed to the public repository
