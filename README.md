# GPON SFP on MikroTik — Guide

Replace an ISP-issued Nokia GPON ONT with an **ODI DFP-34X-2C2** (RTL960x)
SFP module plugged straight into a MikroTik router, using the ISP's fiber
directly — no ONT box in between.

Tested with:

- **Router:** MikroTik RB5009UG+S+IN (any SFP+ device with 2.5G-baseX support should work; 1G fallback included)
- **SFP module:** ODI DFP-34X-2C2
- **ISP:** Tet (Latvia), internet on VLAN 3912 — adapt the VLAN ID to your provider

## 📖 Full guide

👉 **[odi-dfp-34x-2c2-mikrotik-tet-gpon.md](odi-dfp-34x-2c2-mikrotik-tet-gpon.md)**

## What's inside

| Step | Topic |
|---|---|
| 1 | Prerequisites (hardware, firmware, skills) |
| 2 | Reading provisioning values off the original Nokia ONT |
| 3 | Initial RouterOS setup & stick management IP |
| 4 | OMCI provisioning via telnet (`flash set` …) |
| 5 | Forcing the host link speed (`LAN_SDS_MODE`, no auto-neg) |
| 6 | Verifying GPON auth (`diag gpon get onu-state` → O5) |
| 7 | Final config: VLAN + DHCP client + NAT |
| 8 | Troubleshooting: link never comes up, O0/WRONG auth, MAC_KEY recovery, UART unbrick |
| 9 | Hardening: firewalling the stick's telnet/web admin |
| 10 | References (hack-gpon.org, RTL960x, modded firmware) |

## TL;DR

1. Copy `GPON_SN`, `HW_HWVER`, `OMCI_SW_VER1/2` from your ISP ONT.
2. Give the stick a management IP on the SFP interface, telnet to `192.168.1.1`.
3. `flash set` the cloned values + `OMCI_FAKE_OK 1` + `LAN_SDS_MODE 6` (2.5G) or `1` (1G), then reboot.
4. In ROS: disable auto-negotiation, set `2.5G-baseX` (or `1G-baseX`) on the SFP port.
5. Create a VLAN interface for the ISP's internet VLAN, run a DHCP client on it, done.

## ⚠️ Disclaimer

- Cloning your ONT's identity may violate your ISP's terms of service — **your responsibility to check**.
- `flash` commands can brick the module. Follow the guide in order and back up with `flash all` before wiping anything.
- No support contract: if it breaks, you get to keep both pieces (see the troubleshooting/UART sections).

## Credits & sources

- Original write-up: <https://www.hitoha.moe/odi-dfp-34x-2c2-gpon-onu-sfp/>
- Module documentation: <https://hack-gpon.org/ont-odi-realtek-dfp-34x-2c2/>
- Firmware & tools: <https://github.com/Anime4000/RTL960x> and <https://github.com/rajkosto/RTL960x>
