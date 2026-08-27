<div align="center">

# 🔥 Building a Virtual Firewall with pfSense

*A hands-on lab documenting the build and configuration of a pfSense firewall VM in VMware Workstation — from installation through custom firewall rules and full traffic routing.*

![Platform](https://img.shields.io/badge/platform-VMware%20Workstation-607078?style=for-the-badge&logo=vmware&logoColor=white)
![Firewall](https://img.shields.io/badge/firewall-pfSense%20CE-212121?style=for-the-badge)
![Status](https://img.shields.io/badge/status-complete-2E8B57?style=for-the-badge)

</div>

---

### 📋 At a glance

| | |
|---|---|
| **Goal** | Stand up a pfSense firewall, write real rules, and route live traffic through it |
| **Hypervisor** | VMware Workstation |
| **Firewall** | pfSense Community Edition |
| **Key skill demonstrated** | Firewall rule logic (block / pass / rule ordering), network routing, virtualization |
| **Outcome** | Successfully blocked a specific domain for the entire LAN while preserving normal internet access |

---

## Table of contents

- [Why I built this](#why-i-built-this)
- [Environment](#environment)
- [The build, step by step](#the-build-step-by-step)
- [Creating firewall rules](#creating-firewall-rules)
- [Routing host traffic through the firewall](#routing-host-traffic-through-the-firewall)
- [Validation](#validation)
- [Skills demonstrated](#-skills-demonstrated)
- [Lessons learned](#lessons-learned)
- [Repo structure](#-repo-structure)

---

## Why I built this

Firewalls are the first line of defense in almost every network, and pfSense is one of the most widely deployed open-source firewall platforms in both home labs and small enterprise environments. Rather than just reading about firewall rule logic — allow/block/reject, rule ordering, source/destination matching — I wanted to actually build one, put my own traffic through it, and watch the rules take effect in real time.

This project (based on Security Blue Team's BTL1 optional coursework) walks through standing up a pfSense virtual firewall, writing rules to block a specific domain, and routing my host machine's traffic through it so the rule actually takes effect.

## Environment

| Component | Spec |
|---|---|
| Hypervisor | VMware Workstation |
| Firewall VM | pfSense Community Edition, 1 vCPU, 1024MB RAM, ~8GB disk |
| Networking | Bridged adapter (firewall gets its own IP on the physical LAN) |
| Firewall LAN IP | *(e.g. 192.168.1.x)* |
| Router / upstream gateway | *(your router's IP)* |

## The build, step by step

### 1. Getting the pfSense ISO

Downloaded the pfSense Community Edition ISO (AMD64, CD Image installer) from the official Netgate download portal.

![pfSense download selection](screenshots/01-download-selection.png)

### 2. Creating the VM in VMware

Created a new VM, pointed at the pfSense ISO. Since VMware doesn't always auto-detect a FreeBSD-based ISO correctly, the guest OS type was set manually.

![VM name and OS type](screenshots/02-vm-name-os.png)

Allocated 1024MB RAM (pfSense's recommended minimum for comfortable use) and a modest disk size, since pfSense itself is lightweight.

Critically, the network adapter needed to be set to **Bridged** rather than the default NAT — this gives the firewall its own IP address on the physical network, which is required for it to act as a real gateway device rather than just another NATed client.

![Network adapter set to Bridged](screenshots/03-bridged-adapter.png)

### 3. Installing pfSense

Booted into the installer and worked through the guided install.

![pfSense installer welcome screen](screenshots/04-installer-welcome.png)

### 4. Console network configuration

At first boot, configured the WAN/LAN interfaces at the console: skipped VLAN setup, and prepared to assign interfaces.

![Interface assignment menu](screenshots/05-interface-assignment-menu.png)

Mapped WAN to the VM's single available network interface, and left LAN unassigned — a deliberate single-interface setup, since this firewall only needs to inspect traffic leaving toward the internet for this lab.

![WAN interface assignment](screenshots/06-wan-assignment.png)

Once configured, the console displayed the WAN interface's DHCP-assigned IP address — the address needed to reach the web-based management console.

![Console showing assigned WAN IP](screenshots/07-console-wan-ip.png)

### 5. Logging into the web console

Navigated to the firewall's assigned IP over HTTPS in a browser (accepting the self-signed certificate warning) and logged in with the default credentials.

![pfSense login page](screenshots/08-login-page.png)

### 6. Running the setup wizard

Worked through pfSense's initial setup wizard.

![Setup wizard welcome](screenshots/09-wizard-welcome.png)

Configured hostname, domain, and DNS servers.

![General information — hostname and DNS](screenshots/10-general-info.png)

Set the time server and timezone.

![Time server configuration](screenshots/11-time-server.png)

Critically, switched the WAN interface from DHCP to a **static IP** matching the address it was already using, so the firewall's address wouldn't shift on reboot.

![WAN static IP configuration](screenshots/12-wan-static-config.png)

Set a proper admin password to replace the insecure default, then reached the pfSense dashboard, which surfaces system info, interface status, and resource usage at a glance.

![pfSense dashboard](screenshots/13-dashboard.png)

## Creating firewall rules

By default, pfSense blocks all incoming connections on an interface with no rules defined — a safe default, visible on the empty WAN rules page.

![Empty firewall rules page](screenshots/14-firewall-rules-empty.png)

To demonstrate rule logic, I built a rule set to block a specific domain (`redhunt.net`) while allowing all other traffic.

### Finding the target IP

```
ping redhunt.net
```

This resolves the domain to an IP address without actually needing a response — a quick `Ctrl+C` after the first line is enough.

### Creating an alias

Rather than hardcoding the IP directly into a rule, created a reusable **Alias** — if the site's IP ever changes, only the alias needs updating, not every rule referencing it.

![Firewall alias for target domain](screenshots/15-alias-creation.png)

### Writing the block rule

| Setting | Value |
|---|---|
| Action | Block |
| Protocol | Any |
| Source | Network — my LAN subnet |
| Destination | Single host or alias — the alias created above |

![Block rule configuration](screenshots/16-block-rule.png)

### Writing an allow-all rule

Since pfSense blocks everything by default with no rules present, an explicit **allow-all** rule was required underneath the block rule — otherwise, redirecting my host's traffic through the firewall would have cut off all internet access, not just the targeted domain.

![Allow-all rule configuration](screenshots/17-allow-all-rule.png)

Firewall rules are evaluated top-down, so **rule order matters**: the specific block rule for the target domain sits above the general allow-all rule, ensuring the more specific rule is checked first.

![Final rule order](screenshots/18-rule-order.png)

## Routing host traffic through the firewall

Rules alone don't do anything until traffic actually flows through the firewall. This required reconfiguring my host machine's networking.

### Checking current network config

```
ipconfig /all
```

![Host network config before changes](screenshots/19-host-ipconfig-before.png)

### Planning the new topology

**Before**: Host → Router → Internet directly (pfSense uninvolved in the path)

![Network diagram before](screenshots/20-topology-before.png)

**After**: Host → pfSense → Router → Internet

![Network diagram after](screenshots/21-topology-after.png)

### Updating pfSense's own gateway

Set pfSense's upstream gateway to the router's IP, giving the firewall itself internet access — without this, pfSense would have nowhere to forward traffic once it started receiving it from the host.

![pfSense gateway configuration](screenshots/22-pfsense-gateway.png)

### Updating the host's network settings

Located the correct network adapter to modify.

![Host network connections list](screenshots/23-network-connections.png)

Changed the host's network adapter from automatic (DHCP) configuration to static: same IP address, but **default gateway repointed to pfSense's IP**, and DNS pointed at pfSense as well.

![Host network settings — gateway set to pfSense](screenshots/24-host-network-static.png)

## Validation

With traffic now flowing through pfSense, tested both outcomes:

- **General browsing**: worked normally, thanks to the allow-all rule
- **The blocked domain**: attempting to reach `redhunt.net` failed to resolve/connect, confirming the block rule was being enforced correctly

![Blocked domain confirmation](screenshots/25-blocked-domain.png)

## 🧠 Skills demonstrated

<table>
<tr>
<td width="33%" valign="top">

**Firewall administration**
- Rule creation (block / pass / reject)
- Rule ordering and evaluation logic
- Aliases for maintainable rule sets

</td>
<td width="33%" valign="top">

**Networking**
- Static IP configuration
- Default gateway routing
- DNS server assignment

</td>
<td width="33%" valign="top">

**Virtualization**
- VM provisioning in VMware
- Bridged networking configuration
- Guest OS installation (FreeBSD-based)

</td>
</tr>
</table>

## Lessons learned

> [!IMPORTANT]
> **A firewall with no rules blocks everything by default.** This is a safe default, but it means that redirecting traffic through a freshly configured firewall — even temporarily — will cut off connectivity entirely unless an allow-all (or sufficiently broad) rule exists first.

> [!NOTE]
> **Rule order determines behavior, not just rule content.** Firewalls evaluate rules top-down and stop at the first match. A correctly written block rule placed *below* a broad allow rule will never fire, since the allow rule matches first.

> [!TIP]
> **Aliases decouple rules from specific values.** Referencing an alias instead of a hardcoded IP means a single update point if the underlying value (like a domain's IP) ever changes — a small habit that scales well as a rule set grows.

> [!TIP]
> **A firewall only affects traffic that actually passes through it.** Writing correct rules isn't enough — the network's routing (default gateways, both on the firewall and on client machines) has to actually direct traffic through the firewall for those rules to take effect.

## 📁 Repo structure

<details>
<summary>Click to expand</summary>

```
.
├── README.md
└── screenshots/
    ├── 01-download-selection.png
    ├── 02-vm-name-os.png
    ├── 03-bridged-adapter.png
    ├── 04-installer-welcome.png
    ├── 05-interface-assignment-menu.png
    ├── 06-wan-assignment.png
    ├── 07-console-wan-ip.png
    ├── 08-login-page.png
    ├── 09-wizard-welcome.png
    ├── 10-general-info.png
    ├── 11-time-server.png
    ├── 12-wan-static-config.png
    ├── 13-dashboard.png
    ├── 14-firewall-rules-empty.png
    ├── 15-alias-creation.png
    ├── 16-block-rule.png
    ├── 17-allow-all-rule.png
    ├── 18-rule-order.png
    ├── 19-host-ipconfig-before.png
    ├── 20-topology-before.png
    ├── 21-topology-after.png
    ├── 22-pfsense-gateway.png
    ├── 23-network-connections.png
    ├── 24-host-network-static.png
    └── 25-blocked-domain.png
```

</details>

---

<div align="center">

*Built as part of Security Blue Team's BTL1 (Blue Team Level 1) optional coursework.*

</div>
