# Home Network Security Audit

Security and performance assessment of a residential two-story network:
dual-band ISP router/modem, single-band WiFi repeater, and IoT devices
(IP cameras, smart lamp).

## Scope

- Target: personal home network (own equipment, educational purpose)
- 2 subnets (router network + repeater network), 10+ discovered devices
- Assessment of: network exposure, firmware security, DNS hygiene, WiFi performance

## Executive Summary

| Severity | Count | Key finding |
|---|---|---|
| Critical | 1 | Repeater firmware has no real authentication (client-side only, credentials in plaintext) |
| High | 1 | ISP-activated public hotspot running on customer modem |
| Medium | 2 | UPnP exposed, IP cameras on default credentials |
| Low | 1 | DNS resolved through untrusted intermediate device |

**Result:** mapped the full attack surface, applied DNS hardening, and
identified the physical performance ceiling of the wireless infrastructure.

## Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Host discovery, port/service scanning (`-sn`, `-F -Pn`) |
| `ping` | Latency and link-stability measurement |
| `speedtest-cli` | Internet throughput measurement |
| `dig` / `resolvectl` | DNS resolution diagnostics and timing |
| `curl` | Web-interface inspection (login forms, config pages) |
| `nmcli` | Connection and DNS configuration (Linux) |
| `ip` | Interface and routing inspection |
| WiFiMan (Ubiquiti) | Phone-based signal scanning and network discovery |

## Network Topology

```
        FRONT / entrance
           |
        MODEM/ROUTER (1st floor, front wall)     dual-band 2.4GHz + 5GHz
           |
        REPEATER (ground floor, entrance)        single-radio 2.4GHz
           |
        LAPTOP (ground floor, middle room)
           |
        IP cameras (2) + smart lamp + phones
```

## Methodology

1. **Reconnaissance** - enumerate hosts and services on both subnets with `nmap`.
2. **Firmware analysis** - inspect web panels of the repeater and router via
   `curl` (login forms, embedded config, exposed APIs).
3. **Attack-surface assessment** - open ports, default credentials, UPnP, public hotspot.
4. **Performance diagnostics** - repeated `ping` and `speedtest-cli` runs
   across repeater positions.
5. **Hardening** - applied DNS change and documented mitigations.

## Findings

### CRITICAL - Repeater firmware has no real authentication

- **Device:** WiFi repeater (goahead-based "WiFi-Repeater Web Server" firmware)
- **Exposure:** ports `80/tcp` (HTTP) and `23/tcp` (telnet)
- **Detail:** admin credentials are embedded in the login page source; the
  "validation" is client-side JavaScript only. Any device on the network can
  open the config pages without a password.

```html
var musername = "admin";
var userpass = "admin";
```

- **Impact:** full device takeover, network configuration manipulation, potential pivot point.
- **Remediation:** no vendor fix (proprietary firmware). Mitigation: strong
  WPA2 password, isolate the network via VLAN/AP isolation at the router.

### HIGH - ISP public hotspot active on customer modem

- **Finding:** the ISP auto-enables a public "WiFi Zone" SSID on the modem,
  giving other ISP clients internet access through the subscriber's connection.
- **Impact:** consumes subscriber bandwidth; expands the attack surface if
  client isolation is not enforced.
- **Remediation:** request the ISP to disable the public hotspot.

### MEDIUM - UPnP exposed on router

- **Exposure:** `1900/udp` (UPnP/SSDP)
- **Impact:** internal services can be exposed without consent; common abuse vector.
- **Remediation:** disable UPnP in the router admin panel; change the default admin password.

### MEDIUM - IP cameras on default credentials

- **Exposure:** ports `80/tcp` (HTTP) and `554/tcp` (RTSP)
- **Impact:** unauthorized video access.
- **Remediation:** change credentials in the vendor app; keep cameras on a
  dedicated segment if possible.

### LOW - DNS resolved through untrusted intermediate device

- **Finding:** DNS requests were forwarded by the repeater (via the ISP) with
  ~224 ms resolution latency.
- **Action taken:** switched laptop to Cloudflare DNS `1.1.1.1` / `1.0.0.1`
  (`nmcli`, `ipv4.ignore-auto-dns yes`).
- **Result:** resolution dropped to ~37 ms (cold) and ~0-1 ms (cached);
  removed the intermediate device from the DNS trust chain.

## Wireless Performance Notes

- The 5 GHz band of the router cannot reach the ground-floor room through the
  concrete floor slab (5 GHz attenuates more than 2.4 GHz per meter).
- The repeater is a single-radio 2.4 GHz device: half-duplex repeating caps
  real throughput at ~6-11 Mbps regardless of position. This is a hardware
  ceiling, not a configuration issue.
- Repositioning tests identified the optimal repeater placement (LAN latency
  1.3 ms, stable).

## Commands Used

```bash
# Host discovery
nmap -sn 192.168.x.0/24

# Port scan (fast, no ping, with host timeout)
nmap -F -Pn --host-timeout 90s 192.168.x.x

# Latency checks
ping -c 4 192.168.x.1
ping -c 4 8.8.8.8

# DNS diagnostics
dig @1.1.1.1 google.com
resolvectl status wlp3s0

# Web-interface inspection
curl -s http://192.168.x.1/ | grep -i "musername\|userpass"

# DNS hardening (Linux/NetworkManager)
nmcli con mod <conn> ipv4.dns "1.1.1.1 1.0.0.1" ipv4.ignore-auto-dns yes
nmcli device reapply <iface>
```

## Remediations Applied

- [x] DNS switched to Cloudflare `1.1.1.1` / `1.0.0.1` (laptop)
- [x] Network topology fully mapped
- [x] Optimal repeater placement identified
- [x] Disable UPnP + change router admin password
- [x] Change IP camera credentials
- [ ] Pending: request ISP to disable the public hotspot

> Note: UPnP, router admin password, and camera credentials remediation
> applied on the same day as this audit.

## Notes

- Analysis performed on the author's own network for educational purposes.
- All IP addresses are shown anonymized as `192.168.x.x`.
