# Building a Virtual Firewall with pfSense

*A hands-on lab documenting the build of a pfSense firewall VM — including a real installation failure and the network fix that resolved it — inside VMware Workstation.*

![Platform](https://img.shields.io/badge/platform-VMware%20Workstation-607078?style=flat-square)
![Firewall](https://img.shields.io/badge/firewall-pfSense%20CE-212121?style=flat-square)
![Status](https://img.shields.io/badge/status-complete-2E8B57?style=flat-square)

---

## Table of contents

- [Why I built this](#why-i-built-this)
- [Environment](#environment)
- [The build, step by step](#the-build-step-by-step)
- [Troubleshooting: a real installation failure](#troubleshooting-a-real-installation-failure)
- [Creating firewall rules](#creating-firewall-rules)
- [Routing host traffic through the firewall](#routing-host-traffic-through-the-firewall)
- [Validation](#validation)
- [Lessons learned](#lessons-learned)
- [Repo structure](#repo-structure)

---

## Why I built this

Firewalls are the first line of defense in almost every network, and pfSense is one of the most widely deployed open-source firewall platforms in both home labs and small enterprise environments. Rather than just reading about firewall rule logic (allow/block/reject, rule ordering, source/destination matching), I wanted to actually build one, break my own internet connection with it, and fix it — which is a much better way to internalize how firewalls actually behave in practice.

This project (based on Security Blue Team's BTL1 optional coursework) walks through standing up a pfSense virtual firewall, writing rules to block a specific domain, and routing my host machine's traffic through it so the rule actually takes effect.

## Environment

| Component | Spec |
|---|---|
| Hypervisor | VMware Workstation |
| Firewall VM | pfSense Community Edition, 1 vCPU, 1024MB RAM, ~8GB disk |
| Networking | Bridged adapter (firewall gets its own IP on the physical LAN) |
| Firewall LAN IP | *(fill in — e.g. 192.168.1.x)* |
| Host default gateway (before) | *(your router's IP)* |
| Host default gateway (after) | pfSense's IP — traffic now flows Host → pfSense → Router → Internet |

## The build, step by step

### 1. Getting the pfSense ISO

Downloaded the pfSense Community Edition ISO (AMD64, CD Image installer) from the official Netgate download portal.

![pfSense download selection](screenshots/00-download-selection.png)

### 2. Creating the VM in VMware

Created a new VM in VMware Workstation, pointed at the pfSense ISO. Since VMware doesn't always auto-detect a FreeBSD-based ISO correctly, the guest OS type was set manually. Allocated 1024MB RAM (pfSense's recommended minimum for comfortable use) and a modest disk size, since pfSense itself is lightweight.

Critically, the network adapter needed to be set to **Bridged** rather than the default NAT — this gives the firewall its own IP address on the physical network, which is required for it to act as a real gateway device rather than just another NATed client.

### 3. Installing pfSense

Booted into the installer and worked through the guided install (auto partitioning, base system extraction).

![WAN interface assignment](screenshots/01-wan-interface-assignment.png)
*(Interface assignment screen — WAN mapped to the VM's single network adapter)*

![Interface assignment confirmation](screenshots/02-interface-confirmation.png)
*(Confirming WAN-only assignment before continuing installation — no LAN interface, matching the single-NIC bridged setup)*

## Troubleshooting: a real installation failure

Partway through the install, the process stalled while fetching the `pfSense-base` package — it needs to reach `pkg.pfsense.org` mid-install to pull down the core system. After a period with no progress, the installer eventually surfaced a hard failure:

![Installation failure](screenshots/03-install-failure.png)
*(`pkg-static: Failed to fetch` error, followed by "The installation has failed!")*

**Root cause**: with the network adapter set to Bridged, the VM wasn't reliably reaching the internet during install — likely due to VMware's bridge not being mapped correctly to the active physical adapter at that stage.

**Fix**: temporarily switched the network adapter to **NAT** (which routes through the host's already-working connection unconditionally), re-ran the installer, and the package fetch completed without issue. Once the install succeeded, I switched the adapter back to **Bridged** before continuing — Bridged mode is required for the rest of the lab, since the firewall needs its own real network identity to route traffic for other devices.

> [!TIP]
> If a pfSense (or any FreeBSD-based) installer needs to fetch packages over the network mid-install and networking seems flaky under Bridged mode, temporarily switching to NAT for the install — then reverting to Bridged afterward — is a fast, reliable workaround.

### 4. Console network configuration

After a successful install and reboot, configured the WAN/LAN interfaces at the console: no VLANs, WAN mapped to the single available NIC, LAN left unassigned.

![Console showing assigned WAN IP](screenshots/04-console-wan-ip.png)

### 5. Logging into the web console

Navigated to the firewall's DHCP-assigned IP in a browser and logged in with the default credentials (`admin` / `pfsense`).

![pfSense login page](screenshots/05-login-page.png)

### 6. Running the setup wizard

Worked through pfSense's initial setup wizard: hostname/domain, DNS servers, timezone, and — critically — switched the WAN interface from DHCP to a **static IP** matching the address it was already using, so the firewall's address wouldn't shift on reboot.

![WAN static IP configuration](screenshots/06-wan-static-config.png)

Set a proper admin password to replace the insecure default, then reached the pfSense dashboard.

![pfSense dashboard](screenshots/07-dashboard.png)

## Creating firewall rules

By default, pfSense blocks all incoming connections on an interface with no rules defined — a safe default. To demonstrate rule logic, I built a rule set to block a specific domain (`redhunt.net`) while allowing all other traffic.

### Finding the target IP

```
ping redhunt.net
```

![Ping resolving target domain](screenshots/08-ping-target.png)

### Creating an alias

Rather than hardcoding the IP directly into a rule, created a reusable **Alias** — if the site's IP ever changes, only the alias needs updating, not every rule referencing it.

![Firewall alias for target domain](screenshots/09-alias-creation.png)

### Writing the block rule

| Setting | Value |
|---|---|
| Action | Block |
| Protocol | Any |
| Source | Network — `192.168.x.0/24` (my LAN) |
| Destination | Single host or alias — the alias created above |

![Block rule configuration](screenshots/10-block-rule.png)

### Writing an allow-all rule

Since pfSense blocks everything by default with no rules present, an explicit **allow-all** rule was required underneath the block rule — otherwise, redirecting my host's traffic through the firewall would have cut off all internet access, not just the targeted domain.

![Allow-all rule configuration](screenshots/11-allow-all-rule.png)

Firewall rules are evaluated top-down, so **rule order matters**: the specific block rule for the target domain sits above the general allow-all rule, ensuring the more specific rule is checked first.

![Final rule order](screenshots/12-rule-order.png)

## Routing host traffic through the firewall

Rules alone don't do anything until traffic actually flows through the firewall. This required reconfiguring my host machine's networking.

### Checking current network config

```
ipconfig /all
```

![Host network config before changes](screenshots/13-host-ipconfig-before.png)

### Planning the new topology

**Before**: Host → Router → Internet (pfSense uninvolved)

![Network diagram before](screenshots/14-topology-before.png)

**After**: Host → pfSense → Router → Internet

![Network diagram after](screenshots/15-topology-after.png)

### Updating pfSense's own gateway

Set pfSense's upstream gateway to the router's IP, giving the firewall itself internet access.

![pfSense gateway configuration](screenshots/16-pfsense-gateway.png)

### Updating the host's network settings

Changed the host's network adapter from automatic (DHCP) configuration to static: IP unchanged, but **default gateway repointed to pfSense's IP**, and DNS pointed at pfSense as well.

![Host network settings before](screenshots/17-host-network-before.png)
![Host network settings after — gateway set to pfSense](screenshots/18-host-network-after.png)

## Validation

With traffic now flowing through pfSense, tested both outcomes:

- **General browsing**: worked normally (thanks to the allow-all rule)
- **The blocked domain**: attempting to reach `redhunt.net` failed to resolve/connect, confirming the block rule was being enforced

![Blocked domain confirmation](screenshots/19-blocked-domain.png)

## Lessons learned

> [!IMPORTANT]
> **A firewall with no rules blocks everything by default.** This is a safe default, but it means that redirecting traffic through a freshly configured firewall — even temporarily — will cut off connectivity entirely unless an allow-all (or sufficiently broad) rule exists first.

> [!NOTE]
> **Rule order determines behavior, not just rule content.** Firewalls evaluate rules top-down and stop at the first match. A correctly written block rule placed *below* a broad allow rule will never fire, since the allow rule matches first.

> [!TIP]
> **Aliases decouple rules from specific values.** Referencing an alias instead of a hardcoded IP means a single update point if the underlying value (like a domain's IP) ever changes — a small habit that scales well as a rule set grows.

> [!TIP]
> **Installer network dependencies aren't always obvious.** A modern installer silently reaching out to the internet mid-install (to fetch packages) can fail in non-obvious ways under certain virtual networking modes. When an install stalls or fails during a "fetching" step, checking basic network reachability first — before assuming a corrupted ISO or bad config — saves time.

## Repo structure

```
.
├── README.md
└── screenshots/
    ├── 00-download-selection.png
    ├── 01-wan-interface-assignment.png
    ├── 02-interface-confirmation.png
    ├── 03-install-failure.png
    ├── 04-console-wan-ip.png
    ├── 05-login-page.png
    ├── 06-wan-static-config.png
    ├── 07-dashboard.png
    ├── 08-ping-target.png
    ├── 09-alias-creation.png
    ├── 10-block-rule.png
    ├── 11-allow-all-rule.png
    ├── 12-rule-order.png
    ├── 13-host-ipconfig-before.png
    ├── 14-topology-before.png
    ├── 15-topology-after.png
    ├── 16-pfsense-gateway.png
    ├── 17-host-network-before.png
    ├── 18-host-network-after.png
    └── 19-blocked-domain.png
```

---

*Built as part of Security Blue Team's BTL1 (Blue Team Level 1) optional coursework.*
