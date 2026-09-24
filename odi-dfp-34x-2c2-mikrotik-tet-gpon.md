# ODI DFP-34X-2C2 GPON SFP on MikroTik (Tet, Latvia)

Replace the ISP-supplied Nokia GPON ONT with an ODI DFP-34X-2C2 (RTL960x-based)
SFP "stick" plugged directly into a MikroTik router (tested on RB5009UG+S+IN).

Based on: <https://www.hitoha.moe/odi-dfp-34x-2c2-gpon-onu-sfp/> — reworked for
clarity, with the config contradictions fixed and security notes added.

> **Disclaimer**
>
> - This clones your ISP ONT's identity (serial number, hardware/software
>   versions) to authenticate on the ISP's GPON network. Doing so may violate
>   your ISP's terms of service. You are responsible for checking that yourself.
> - You can permanently brick the SFP module with wrong `flash` commands.
> - Everything below is provided as-is, without warranty.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Collect values from the Nokia ONT](#2-collect-values-from-the-nokia-ont)
3. [Initial RouterOS setup](#3-initial-routeros-setup)
4. [Configure the SFP module (telnet)](#4-configure-the-sfp-module-telnet)
5. [Fix the link speed](#5-fix-the-link-speed)
6. [Verify GPON authentication](#6-verify-gpon-authentication)
7. [Final RouterOS configuration](#7-final-routeros-configuration)
8. [Troubleshooting](#8-troubleshooting)
9. [Security notes](#9-security-notes)
10. [References](#10-references)

---

## 1. Prerequisites

- MikroTik router with an SFP+ cage that supports **2.5G-baseX**
  (RB5009 works). If your device only supports 1G, see the note in
  [step 5](#5-fix-the-link-speed).
- ODI DFP-34X-2C2 SFP module.
- Access to the original Nokia ONT to read provisioning values from it.
- Winbox/WebFig or CLI access to RouterOS.
- Basic familiarity with telnet and RouterOS CLI.

## 2. Collect values from the Nokia ONT

Before unplugging anything, log into the Nokia ONT and write down:

| Value | SFP flash parameter | Example format |
|---|---|---|
| Software version (active) | `OMCI_SW_VER1` | e.g. `3FE49398AAAA` |
| Software version ( standby) | `OMCI_SW_VER2` | e.g. `3FE49398BBBB` |
| Hardware version | `HW_HWVER` | e.g. `3FE49397AACA` |
| GPON serial number | `GPON_SN` | e.g. `ALCLxxxxxxxx` |

These are needed so the OLT accepts the SFP stick as if it were the Nokia ONT.

## 3. Initial RouterOS setup

Goal: give the SFP module a management IP so you can telnet into it **before**
the fiber side is authenticated.

### 3.1 Take the SFP port out of the bridge

The SFP port must NOT be a bridge member — it is a WAN-facing interface.

Winbox: **Bridge → Ports** → select the `sfp-sfpplus1` entry → change the
interface to `ether1` (or just remove the entry).

```routeros
/interface bridge port
remove [find interface=sfp-sfpplus1]
```

### 3.2 Assign a management IP

The stick lives at `192.168.1.1` on its host interface by default.

> **Note:** `192.168.1.0/24` is a very common subnet. Make sure it does NOT
> overlap with your LAN or anything on the ISP side. If it does, you will have
> routing conflicts — change your LAN subnet instead (the stick's IP is harder
> to change).

Winbox: **IP → Addresses → +**

```routeros
/ip address
add address=192.168.1.100/24 interface=sfp-sfpplus1
```

(`network` is auto-computed by RouterOS — no need to fill it in.)

At this point you should be able to `ping 192.168.1.1` from the router.

## 4. Configure the SFP module (telnet)

From the RouterOS terminal:

```routeros
/system telnet 192.168.1.1
```

Then set the OMCI provisioning values collected in step 2:

```sh
# OLT emulation mode.
# Mode 3 requires the modded firmware (M110_sfp_ODI_220923FS.tar):
# https://github.com/rajkosto/RTL960x/tree/main/Firmware/DFP-34X-2C2
flash set OMCI_OLT_MODE 3

# --- OR, on stock firmware only: -------------------------------------------
# flash set OMCI_OLT_MODE 21
# Mode 21 is a hack: /bin/checkomci crashes with a segmentation fault.
# Prefer mode 3 with the modded firmware whenever possible.
# -----------------------------------------------------------------------------

# Safety net for OMCI state machine:
# https://hack-gpon.org/ont-odi-realtek-dfp-34x-2c2/#gettingsetting-omci-olt-mode-and-fake-omci
flash set OMCI_FAKE_OK 1

# Values from step 2:
flash set OMCI_SW_VER1 <software_version_1>
flash set OMCI_SW_VER2 <software_version_2>
flash set HW_HWVER     <hardware_version>
flash set GPON_SN      <gpon_serial_number>
```

## 5. Fix the link speed

The stick does **not** auto-negotiate the host-side Ethernet speed with the
RB5009 — after reboot it may never come up, or take a very long time. Force the
speed on both sides instead.

On the SFP module (still in telnet):

```sh
# Force 2.5G on the host side (RB5009 supports it):
flash set LAN_SDS_MODE 6

# If your router/media converter does NOT support 2.5G, use 1GbaseX instead:
# flash set LAN_SDS_MODE 1

reboot
```

> Reference: <https://hack-gpon.org/ont-odi-realtek-dfp-34x-2c2/#gettingsetting-speed-lan-mode>

On RouterOS, match the speed and disable auto-negotiation:

```routeros
# For LAN_SDS_MODE 6:
/interface ethernet
set [find default-name=sfp-sfpplus1] auto-negotiation=no speed=2.5G-baseX

# For LAN_SDS_MODE 1:
# /interface ethernet set [find default-name=sfp-sfpplus1] auto-negotiation=no speed=1G-baseX
```

> This only controls the router↔stick **link** speed. It does NOT make your
> internet plan faster.

The stick takes ~1 minute to boot. If it doesn't come up as `running`, see
[Troubleshooting](#8-troubleshooting).

## 6. Verify GPON authentication

Telnet back into the stick (`/system telnet 192.168.1.1`) and check:

```sh
diag gpon get onu-state
```

- **O5** = operational — you are authenticated.
- **O1–O4** = still progressing, wait a bit.
- **O0 / LOID status "WRONG"** = authentication failure → see
  [Troubleshooting](#8-troubleshooting).

Once authenticated, you can read the VLANs provisioned by the OLT:

```sh
omcicli mib get 84
```

Note the internet VLAN ID (in this setup: **3912**).

## 7. Final RouterOS configuration

Terminate the ISP VLAN on the SFP interface and run DHCP on it.

```routeros
# VLAN interface for Tet internet (3912 = example, use your VLAN from step 6)
/interface vlan
add interface=sfp-sfpplus1 name=vlan-tet vlan-id=3912

# WAN interface list: only the VLAN belongs here.
# sfp-sfpplus1 itself must NOT be a bridge member and does not need to be
# in any interface list.
/interface list member
add interface=vlan-tet list=WAN

# DHCP client on the VLAN
/ip dhcp-client
add comment=tet-net interface=vlan-tet
```

Make sure the usual defconf pieces are in place (they exist on a default
config — just verify):

```routeros
# NAT for LAN clients:
/ip firewall nat
print where chain=srcnat and out-interface-list=WAN
# If missing:
# /ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade comment="defconf: masquerade"
```

**Done.** You should receive a public IP on `vlan-tet` and have internet
access:

```routeros
/ip dhcp-client print detail
/ping 1.1.1.1
```

## 8. Troubleshooting

### 8.1 Stick never shows `running` after reboot

The stick cycles through host-side link modes when no fiber is plugged in —
you can use this to catch the right one:

1. Unplug the **fiber** (leave the stick in the cage), then unplug and re-seat
   the **stick**.
2. Wait ~1 minute for it to boot.
3. The stick tries the next link mode roughly every **25 seconds**,
   cycling through: `1` (1GbaseX), `3` (SGMII 1G MAC), `4` (HiSGMII 2.5G PHY),
   `7` (SGMII force 1GbaseT).
4. Plug the fiber in at a later/earlier interval and check whether the
   interface becomes `running` in ROS.
5. Double-check ROS has **auto-negotiation off** and speed set to
   `1G-baseX` or `2.5G-baseX` matching `LAN_SDS_MODE`.

If this still fails, try stick + fiber in a cheap **media converter** first to
isolate whether the problem is the RB5009 cage. There are reports that
negotiation behaves better behind a media converter.

### 8.2 ONU state stuck at O0 / "WRONG"

Likely a bad or missing `MAC_KEY` ( wiped config, module from another ISP, ...).

1. **Back up the current flash config first:**

   ```sh
   flash all
   ```

   Copy the output somewhere safe.

2. **Wipe the config partition and re-provision from scratch:**

   ```sh
   flash_eraseall /dev/mtd3
   reboot
   ```

   > ⚠️ This erases *everything* — all `flash set` values including
   > `LAN_SDS_MODE` and `MAC_KEY`. You must redo the entire step 4–5
   > afterwards.

3. Re-run steps 4–5, then check `diag gpon get onu-state` again.
   **O1→O7 progressing is a good sign. O0/WRONG is not.**

4. If still O0/WRONG, regenerate the `MAC_KEY`. The key is
   `MD5("hsgq1.9a" + MAC_uppercase_hex_only)` — you can use the MAC of the
   stick itself (or even the Nokia ONT's MAC). Ready-made CyberChef recipe:

   <https://gchq.github.io/CyberChef/#recipe=To_Upper_case('All')Find_/_Replace(%7B'option':'Regex','string':'%5B%5E0-9A-F%5D*'%7D,'',true,false,true,false)Find_/_Replace(%7B'option':'Regex','string':'%5E'%7D,'hsgq1.9a',true,false,true,false)MD5()&input=MDA6MDA6MDA6MDA6MDA6MDA>

   Paste a MAC address as input, then set the result:

   ```sh
   flash set MAC_KEY <generated_md5>
   reboot
   ```

### 8.3 Dead stick / no telnet — UART recovery

As a last resort, solder wires to the UART **RX/TX** pads on the module's PCB
(pad locations: see hack-gpon.org link below).

One convenient setup that avoids powering hassles:

1. Solder RX/TX to a **CH341A** USB-UART adapter (the model with a voltage
   select switch — set it to 3.3V).
2. Plug the CH341A into the **USB port of the RB5009**.
3. Plug the stick into the SFP cage (gives it power and common ground).
4. From ROS, connect to the serial console:

   ```routeros
   /port set [find name=usb1] baud-rate=115200
   /system serial-terminal usb1
   ```

Untested idea: a media converter might give access to a half-broken stick
without soldering — see
<https://github.com/xvzf/zyxel-gpon-sfp/issues/8#issuecomment-1472746286>.

## 9. Security notes

- **The stick's admin interfaces (telnet, web UI) have well-known default
  credentials and no real access control.** Anything that can route to
  `192.168.1.1` owns the stick. At minimum:

  ```routeros
  /ip firewall filter
  add chain=input dst-address=192.168.1.1 in-interface-list=WAN action=drop \
      comment="block SFP stick management from WAN"
  add chain=forward dst-address=192.168.1.1 in-interface-list=WAN action=drop \
      comment="block SFP stick management from WAN (forward)"
  ```

  (Defconf already drops unsolicited WAN input — this is defense in depth.)

- **Telnet is plaintext.** Only acceptable because management traffic stays on
  the router↔stick link. Never expose `192.168.1.1` beyond the router.

- **Don't publish real provisioning values.** Scrub `GPON_SN`, MAC addresses,
  software/hardware versions and any OLT output from screenshots and logs
  before posting them anywhere.

## 10. References

- Original guide: <https://www.hitoha.moe/odi-dfp-34x-2c2-gpon-onu-sfp/>
- Stick documentation (OMCI modes, LAN_SDS_MODE, UART pads):
  <https://hack-gpon.org/ont-odi-realtek-dfp-34x-2c2/>
- Firmware, tools and community knowledge base:
  <https://github.com/Anime4000/RTL960x>
- Modded firmware used for `OMCI_OLT_MODE 3`:
  <https://github.com/rajkosto/RTL960x/tree/main/Firmware/DFP-34X-2C2>
