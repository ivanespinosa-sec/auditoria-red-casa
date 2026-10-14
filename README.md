# Home Network Security Audit

In this project I analysed my home network: IPs, MAC addresses, ports, security and speed. It's split into three phases: **Phase 1, Inventory**; **Phase 2, Router review**; and **Phase 3, Fixes**. At the end there's a re-scan to compare before and after, a summary of findings and what I learned. Let's get into it.

The screenshots follow the real order in which I did each step. Below each scan there's a table with what the screenshot shows, device by device. I've covered sensitive data (MAC addresses, passwords, QR codes and part of the Wi-Fi network names).

## Scope and tools

| Item | Details |
|---|---|
| Network analysed | My home network, with my family's knowledge. Range `192.168.1.0/24` |
| Machine | MacBook (macOS 15.6.1), using the Terminal app |
| Tools | Nmap 7.991 and the router's admin panel (Vodafone F@st 5670, Sagemcom) |
| Dates | Phase 1: 8 October 2026 · Phase 2: 10 October · Phase 3 and re-scan: night of 10–11 October |

## Contents
1. [Basic concepts](#basic-concepts)
2. [Phase 1 – Inventory](#phase-1--inventory)
3. [Phase 2 – Router review](#phase-2--router-review)
4. [Phase 3 – Fixes](#phase-3--fixes)
5. [Re-scan](#re-scan-after-the-fixes)
6. [Summary of findings](#summary-of-findings-and-fixes)
7. [Scope](#scope)
8. [What I learned](#what-i-learned)

---

# Basic concepts

| Concept | What it means |
|---|---|
| `192.168.1.0/24` | My local network: the 256 addresses from `192.168.1.0` to `192.168.1.255`. In the text I shorten each IP to its last number (.1, .128…) |
| IP | The address of each device inside the network. The router hands it out (DHCP) and it can change |
| MAC | The network card's identifier. Its first characters show the manufacturer, which is what nmap shows in brackets |
| Random MAC | Modern phones make up a MAC address so they can't be tracked; that's why nmap doesn't recognise the manufacturer and shows `Unknown` |
| Port | A "door" through which a device offers a service (web, SSH, DNS…) |
| `open` / `closed` / `filtered` | Open: a service is listening. Closed: the device replies that nothing is there. Filtered: no reply, usually because of a firewall |
| Latency | How long the device takes to reply. It shows the device is switched on |
| `sudo` | Runs the command as administrator. The Mac asks for the user's password (`Password:`), which isn't shown while typing |

---

# Phase 1 – Inventory
*8 October 2026*

## 1. Discovering the devices (20:26)
```bash
sudo nmap -sn 192.168.1.0/24 -oN ~/Desktop/escaneo-dispositivos.txt
```

| Part of the command | What it does |
|---|---|
| `sudo` | Runs it as administrator. Needed so nmap can see each device's MAC address |
| `nmap` | The scanning tool |
| `-sn` | Only checks which devices are on, without looking at ports |
| `192.168.1.0/24` | Scans my whole local network |
| `-oN ~/Desktop/escaneo-dispositivos.txt` | Saves the result to a text file on the Desktop |

![Device scan, 20:26](docs/capturas/01_escaneo_dispositivos.png)

**What the screenshot shows:**

| IP | MAC | Manufacturer (from the MAC) | Latency | Device |
|---|---|---|---|---|
| .1 | covered | Sagemcom Broadband SAS | 0.0039 s | Router |
| .128 | covered | Apple | 0.027 s | Apple TV |
| .130 | covered | Unknown | 0.097 s | TP-Link range extender (identified in step 4) |
| .131 | covered | Qingdao Intelligent&Precise Electronics | 0.032 s | TV (identified in Phase 2) |
| .145 | covered | Amazon Technologies | 0.13 s | Alexa |
| .147 | not shown | – | – | My MacBook |

**Result:** 6 devices up in 3.59 seconds. The .147 has no MAC line because it's the MacBook I'm scanning from: nmap doesn't show the MAC of the machine running the scan.

## 2. Scanning the ports (20:41)
```bash
nmap -T4 --top-ports 100 192.168.1.0/24 -oN ~/Desktop/escaneo-puertos.txt
```

| Part of the command | What it does |
|---|---|
| `-T4` | "Aggressive" timing: makes the scan faster |
| `--top-ports 100` | Only checks the 100 most common ports on each device |
| `192.168.1.0/24` | My whole local network |
| `-oN ~/Desktop/escaneo-puertos.txt` | Saves the result on the Desktop |

![Port scan, 20:41](docs/capturas/02_escaneo_puertos.png)

**What the screenshot shows:**

| IP | Device | Open ports | Rest of the 100 ports |
|---|---|---|---|
| .1 | Router | 53 (`domain`), 80 (`http`), 443 (`https`), 49152 (`unknown`) | 54 closed and 42 filtered |
| .128 | Apple TV | 5000 (`upnp`), 49152 (`unknown`), 49153 (`unknown`) | 97 closed |
| .130 | Range extender | 22 (`ssh`), 80 (`http`) | 98 filtered |
| .147 | My MacBook | 5000 (`upnp`) | 99 closed |
| .131 and .145 | TV and Alexa | Not shown | – |

**Result:** 4 devices responded, in 30.14 seconds.

**What each open port is:**

| Port | Service | Explanation |
|---|---|---|
| 53 | DNS | The router translates website names (like google.com) into IP addresses |
| 80 | HTTP | Unencrypted web page. On the router it's the admin panel |
| 443 | HTTPS | Encrypted web page. On the router it's the same panel, accessed securely |
| 22 | SSH | Remote terminal access to manage a device |
| 5000 | AirPlay | Nmap labels it `upnp`, but on macOS it's the AirPlay receiver. The .128 is also an Apple device, so it's probably the same thing |
| 49152 and 49153 | Unknown | High ports that nmap doesn't identify. The router's 49152 disappeared after I turned off UPnP (see re-scan) |

**The .130 case.** At first I assumed the .130 was my iPhone, but this result didn't fit: a normal iPhone doesn't have SSH or a web server open. There was something on my network with remote admin services and I didn't know what it was. It could be something that goes unnoticed, like a smart device, but I had to identify it. I also checked my iPhone's real MAC address (on the iPhone: Settings → General → About → Wi-Fi Address) and it didn't match the .130.

**Why don't .131 and .145 show up?** They appeared in the device scan, but not here. I ran this command without `sudo`, and without admin rights nmap checks whether a device is on by trying to connect to a few ports. If the device doesn't answer those attempts, nmap treats it as down and doesn't scan its ports. That's why I later tested the .131 on its own (step 5).

## 3. Repeating the discovery (20:48)
```bash
sudo nmap -sn 192.168.1.0/24
```
The same command as step 1, but without `-oN`, so this time I didn't save it to a file.

![Second device scan, 20:48](docs/capturas/03_escaneo_dispositivos_2.png)

**What the screenshot shows:**

| IP | MAC | Manufacturer (from the MAC) | Latency | Device |
|---|---|---|---|---|
| .1 | covered | Sagemcom Broadband SAS | 0.0042 s | Router |
| .128 | covered | Apple | 0.0073 s | Apple TV |
| .130 | covered | Unknown | 0.20 s | Range extender |
| .131 | covered | Qingdao Intelligent&Precise Electronics | 0.073 s | TV |
| .136 | covered | Unknown | 0.20 s | **My iPhone (new)** |
| .145 | covered | Amazon Technologies | 0.16 s | Alexa |
| .147 | not shown | – | – | My MacBook |

**Result:** 7 devices in 4.17 seconds. The .136 appears, my iPhone, which hadn't answered in the first scan. It shows as `Unknown` because it uses a random MAC. Phones on standby don't always respond, which is why it's worth repeating the scan.

## 4. Identifying the .130 (20:53)
```bash
nmap -sV -p 22,80 192.168.1.130
```

| Part of the command | What it does |
|---|---|
| `-sV` | Asks which program and version are answering on each port |
| `-p 22,80` | Limits the test to ports 22 and 80, the ones that were open |
| `192.168.1.130` | Only this device |

![Services on the .130](docs/capturas/04_servicios_130.png)

**What the screenshot shows:**

| Port | State | Service | Version |
|---|---|---|---|
| 22 | open | ssh | `Cisco/3com IPSSHd 6.6.0 (protocol 2.0)` |
| 80 | open | http | `TP-LINK router http config` |

**Result:** port 80 is the web panel of a TP-Link device. The version on port 22 is nmap's estimate; I haven't verified it.

**Conclusion:** the .130 is my **TP-Link TL-WA850RE range extender**. I confirmed it later in the router panel (Phase 2), where it appears with that name and that IP.

## 5. Checking the .131 (20:56)
```bash
nmap -sV -T4 --top-ports 100 192.168.1.131
```
It combines the previous options: the 100 most common ports, fast, and detecting versions, only on the .131.

![Scan of the .131](docs/capturas/05_tv_131.png)

**What the screenshot shows:**

| IP | Result | Time |
|---|---|---|
| .131 | `Note: Host seems down` · 0 hosts up | 2.47 s |

**Result:** the device had appeared in the device scan, but didn't respond to this one. Nmap says it may be blocking ping and suggests repeating the test with `-Pn`, which skips that check. I didn't get round to repeating it with `-Pn`.

From its MAC manufacturer (Qingdao Intelligent&Precise Electronics) I suspected it was the TV, and I confirmed it later with the router log (Phase 2).

## Phase 1 result

| IP | Manufacturer (from the MAC) | First guess | Final identification |
|---|---|---|---|
| .1 | Sagemcom | Router | Router |
| .128 | Apple | Apple TV | Apple TV (wired; on the router it's called "Sala-de-estar") |
| .130 | Unknown | My iPhone ❌ | TP-Link TL-WA850RE range extender (step 4 and router panel) |
| .131 | Qingdao Intelligent&Precise | TV | TV (its MAC matches the `WL_VIDEO_5G` device in the router log, Phase 2) |
| .136 | Unknown (random MAC) | – | My iPhone |
| .145 | Amazon Technologies | Alexa | Alexa |
| .147 | – (machine I scan from) | My MacBook | My MacBook (on the router it's called "MacBookPro") |

---

# Phase 2 – Router review
*10 October 2026*

## How I got into the panel
1. I looked up the router's address on the Mac: **System Settings → Wi-Fi → Details (of my network) → Router**. It's `192.168.1.1`.
2. I typed `192.168.1.1` into the address bar. It didn't load in my usual browser; it did in Safari.
3. I logged in with the username and password on the router's label.

Router: **Vodafone F@st 5670** (Sagemcom). The home page has cards (F@st 5670, Access Control, Wi-Fi 2.4GHz, Wi-Fi 5GHz…) with a gear icon that opens the settings for each section.

## Home screen (screenshot not published)
I'm not publishing these screenshots because they show the network and device names. What I saw:
- Three Wi-Fi networks: 2.4 GHz, 5 GHz and a guest network.
- **All devices were connected to the 2.4 GHz network.** The 5 GHz and guest networks had no devices.
- On cable (LAN card) there was only the Apple TV ("Sala-de-estar", .128).

| Device | IP | Signal | Link speed |
|---|---|---|---|
| My iPhone | .136 | −47 dBm | 216.66 Mbps |
| My MacBook | .147 | −34 dBm | 229.41 Mbps |
| My mum's iPhone | .150 | −45 dBm | 229.41 Mbps |
| Alexa (no name) | .145 | −41 dBm | 65 Mbps |
| My brother's Galaxy | .144 | −63 dBm | 72.222 Mbps |
| My mum's iPad | .146 | −39 dBm | 130 Mbps |

Signal is measured in dBm: the closer to 0, the better (−34 is very good, −63 is acceptable).

## Wi-Fi encryption
**Path:** Wi-Fi 2.4GHz card (gear icon) → **Basic** tab.

**WPA2 Personal** on 2.4 and 5 GHz. It's a sound encryption method. The passwords are the factory ones and different for each band.

![2.4 and 5 GHz Wi-Fi with WPA2 Personal](docs/capturas/08_wifi_wpa2.png)

## WPS
**Path:** same page → **WPS** tab → **2.4 GHz** button and then **5 GHz**.

**Enabled on both bands.** WPS PIN pairing has known weaknesses, so it's a finding.

![WPS enabled on 2.4 GHz](docs/capturas/09_wps_24_on.png)
![WPS enabled on 5 GHz](docs/capturas/10_wps_5_on.png)

## Access Control
**Path:** Access Control card (gear icon). It has six tabs: Parental Control, Port Forwarding, Firewall IPv4, Firewall IPv6, DMZ and User.

**Parental Control:** off (switch set to OFF). In the device list there's one called `ESP_089145`; ESP is a common chip in smart-home devices, but I haven't been able to identify which one it is.

![Parental control off](docs/capturas/11_control_parental.png)

**Port Forwarding → UPnP IGD: enabled.** The router itself warns that it can be a risk. The port forwarding rules table was empty.

![UPnP enabled](docs/capturas/12_upnp_on.png)

**Firewall IPv4: Low level**, with LAN→WAN and WAN→LAN set to "Allow all" (everything allowed both ways) and "Respond To Ping" enabled. LAN is my home network and WAN is the internet. The IPv6 firewall was the same.

![Firewall at Low level](docs/capturas/13_firewall_low.png)

**DMZ: off.** The DMZ exposes a device directly to the internet; having it off is correct.

![DMZ off](docs/capturas/14_dmz_off.png)

**User:** the panel username is `1234` and it can't be changed (the field is greyed out); only the password can be changed. The password was also the factory one, `1234`. That password no longer works, because I changed it in Phase 3; the username is still the same.

![Panel user](docs/capturas/15_usuario_panel.png)

## Device information (screenshot not published)
**Path:** F@st 5670 card → **Device Info** tab → **General**.

I'm not publishing this screenshot because it shows the serial number, MAC addresses and public IP.

| Item | Value |
|---|---|
| Hardware | 2.00 |
| Software (firmware) | `SGDV20000151`. Updates are managed by the ISP |
| Interface | 5.122.1 |
| Uptime | 3 days and 18 hours without a restart |
| Memory in use | 76 % |
| Local network | `192.168.1.0/24`, router at `192.168.1.1` |

## DHCP leases (screenshot not published)
**Path:** Device Info → **DHCP Leases**. It's the list of devices the router has given an IP to.

| IP | Name on the router | Device |
|---|---|---|
| .128 | Sala-de-estar | Apple TV |
| .130 | TL-WA850RE | TP-Link range extender |
| .136 | iPhone | My iPhone |
| .140 and .144 | Galaxy-M12 | My brother's phone (appears twice; possibly a different random MAC per network, not verified) |
| .145 | (no name) | Alexa |
| .146 | iPad | My mum's iPad |
| .147 | MacBookPro | My MacBook |
| .150 | iPhone | My mum's iPhone |

## Router logs
**Path:** F@st 5670 card → **Maintenance** tab → **Logs**. I reviewed page 1 of 15 (from 02:50 to 11:35).
- **No logins to the panel by anyone else** in those entries.
- **Remote management by the ISP:** every 2 hours the router connects to `acs.masmovil.com` via TR-069, the system the ISP uses to manage the router remotely. Looking it up, I found it's common on ISP routers. It can't be turned off from the panel.
- **Internal network names:** the router uses names with the pattern `WL_…`. My iPhone connected at 10:00 to `WL_PRIV`, my private network. `WL_VIDEO_5G` also appears, from which a device disconnected at 10:55. Comparing its MAC with the first scan, **it matches the .131 exactly**, so it's the TV.
- **Repeated DNS errors:** queries from my devices (Bonjour, calendar) that the router can't resolve. Not important.

## Guest network
Off. No finding.

## Phase 2 findings
- Factory password on the admin panel.
- WPS enabled on both bands.
- UPnP enabled.
- Firewall at the most permissive level (Low) on IPv4 and IPv6.

---

# Phase 3 – Fixes
*Night of 10–11 October 2026*

For each fix I explain why I did it, the goal, how I did it, before, after and its limits. Every change in the panel needs **Apply** to be saved.

## Fix 1: Turning off WPS (2.4 and 5 GHz)
- **Why:** WPS PIN has known weaknesses. The 8-digit PIN can be guessed and gives access to the Wi-Fi without knowing the password.
- **Goal:** close that way in.
- **How:** Wi-Fi 2.4GHz & 5GHz → **WPS** tab → **2.4 GHz** → **Enable WPS** switch to OFF → **Apply**. Same on **5 GHz**.
- **Before:** enabled on both bands.
- **After:** disabled on both, checked in the panel. As the router itself warns, the physical WPS button is also disabled.
- **Limit:** it only affects someone within range of the signal. To connect a new device you have to type the password.

![WPS disabled on 2.4 GHz](docs/capturas/18_wps_24_off.png)
![WPS disabled on 5 GHz](docs/capturas/18_wps_5_off.png)

## Fix 2: Router admin panel password
- **Why:** the panel used the factory password. Any device connected to my Wi-Fi could log in and change the settings.
- **Goal:** only I can manage the router.
- **How:** Access Control → **User** tab → typed the old password in **Old Password** and the new one in **New Password** and **Confirm New Password** → **Apply**.
- **Before:** factory username and password.
- **After:** a new, long and unique password, stored in a password manager. The router confirmed it with the message "Data successfully saved".
- **Limit:** the ISP still manages the router remotely (TR-069); that's out of my control.

![Panel password change](docs/capturas/19_contrasena_panel.png)

## Fix 3: Firewall from Low to Medium (IPv4 and IPv6)
- **Why:** it was at the most permissive level on both versions.
- **Goal:** add an extra layer of defence.
- **How:** Access Control → **Firewall IPv4** tab → **Medium** button → **Apply**. Same on **Firewall IPv6**.
- **Before:** Low, with everything allowed both ways.
- **After:** according to the panel, incoming NetBIOS traffic from the internet is now blocked (WAN→LAN: "Block all below: NETBIOS"). Outgoing traffic (LAN→WAN) is still allowed.
- **Limit:** the router already does NAT, which by itself blocks unsolicited incoming connections, so the real improvement is probably small. I haven't noticed any connection problems since the change.

![Firewall IPv4 at Medium](docs/capturas/20_firewall_medium_ipv4.png)
![Firewall IPv6 at Medium](docs/capturas/20_firewall_medium_ipv6.png)

## Fix 4: Turning off UPnP
- **Why:** UPnP lets any device on the network open ports to the internet without telling me. There were no open rules, so this is prevention.
- **Goal:** ports only get opened if I decide so.
- **How:** Access Control → **Port Forwarding** tab → **Enable UPnP IGd** switch to OFF → **Apply**.
- **Before:** enabled, with an empty rules table.
- **After:** disabled, checked in the panel.
- **Limit:** it can affect consoles, online games or some video calls. I haven't noticed any problems since the change.

![UPnP disabled](docs/capturas/21_upnp_off.png)

## Fix 5: Checking the TP-Link range extender (TL-WA850RE)
- **Why:** it has its own admin panel inside my network, with SSH (22) and web (80) open according to the scan.
- **How:** I typed `192.168.1.130` into the browser, which opens the extender's login page, and tried the factory credentials.
- **Result:** they don't work; my brother changed the password when he set it up.
- **Change:** none.
- **Limit:** I couldn't check how strong that password is or the firmware version.

## Fix 6: Moving the MacBook to the 5 GHz band
This fix isn't about security, it's about speed.

**Starting point**
- The router has two networks: 2.4 GHz and 5 GHz. All my devices were on 2.4 GHz, because that's the one saved the first time they connected.
- At first I thought the 5 GHz network was one ending in `_EXT`. It wasn't: that's probably the extender's network (not verified). The 5 GHz network had a completely different name, which I found in the router panel (Wi-Fi 2.4GHz & 5GHz → Basic). Its password, which I checked on that same page, was also different from the 2.4 GHz one.

**Why 5 GHz**
- 5 GHz offers more speed and less interference than 2.4 GHz, in exchange for less range and worse wall penetration.
- I use the MacBook close to the router (−34 dBm signal), so the loss of range doesn't affect me.
- I didn't move the other devices: they're further away, some smart-home devices only work on 2.4 GHz, or they depend on the extender.

**How:** on the MacBook, Wi-Fi icon in the menu bar → chose the 5 GHz network → typed its password.

**How I measured it:** I ran speed tests on testvelocidad.es (**INICIO** button) to check the difference between the two bands, in the same spot, at roughly the same time and always with the MacBook.

| Band | Download | Upload | Ping |
|---|---|---|---|
| 2.4 GHz | 97.17 Mbit/s | 84.75 Mbit/s | 11 ms |
| 5 GHz | 744.9 Mbit/s | 618.6 Mbit/s | 11 ms |

- **Download:** speed when receiving data (watching videos, downloading).
- **Upload:** speed when sending data (uploading files, video calls).
- **Ping:** how long a packet takes to go and come back; the lower, the better.

![Speed test on 2.4 GHz](docs/capturas/22_velocidad_24.png)
![Speed test on 5 GHz](docs/capturas/22_velocidad_5.png)

**Result:** download went from 97 to 745 Mbit/s, **about 7.7 times more**, and upload from 85 to 619 Mbit/s. The improvement is only for this machine.

**Why the difference is so big (hypothesis, I didn't measure each factor separately):**
- **Competition:** on 5 GHz the MacBook was the only device, so it didn't have to wait its turn to talk to the router. On 2.4 GHz it shared the band with everything else.
- **Signal:** the more distance and walls, the weaker the signal and the lower the speed, even if the network is empty. 5 GHz weakens sooner than 2.4 GHz.

**Things to keep in mind:**
- Maximum speed also depends on the contracted line, which I don't know.
- Wi-Fi speed varies with the time of day and what's being used at home, so the values are approximate.

**Renaming the 5 GHz network.** To tell it apart easily, in Wi-Fi 2.4GHz & 5GHz → **Basic** → 5 GHz **SSID** field, I gave it the same name as the 2.4 GHz network ending in `_5G` → **Apply**.

![5 GHz network renamed](docs/capturas/23_nombre_5g.png)

---

# Re-scan (after the fixes)
*11 October 2026, 01:25*

I ran the same commands as in Phase 1, saving them under a different name so I wouldn't overwrite the "before" files:
```bash
sudo nmap -sn 192.168.1.0/24 -oN ~/Desktop/escaneo-dispositivos-despues.txt
nmap -T4 --top-ports 100 192.168.1.0/24 -oN ~/Desktop/escaneo-puertos-despues.txt
```

**Devices found (`-sn`): 8**

| IP | Device | Seen before? |
|---|---|---|
| .1 | Router | Yes, in the Phase 1 scans |
| .128 | Apple TV | Yes, in the Phase 1 scans |
| .130 | Range extender | Yes, in the Phase 1 scans |
| .134 | Desktop PC (through the extender) | No, new in this scan |
| .136 | My iPhone | Yes, in the repeat scan in step 3 |
| .145 | Alexa | Yes, in the Phase 1 scans |
| .147 | My MacBook | Yes, in the Phase 1 scans |
| .150 | My mum's iPhone | New in a scan, but I had already seen it in the router panel (Phase 2) |

The TV (.131) didn't show up because it was switched off.

**Open ports, before and after:**

| IP | Device | Before (8 Oct) | After (11 Oct) |
|---|---|---|---|
| .1 | Router | 53, 80, 443, 49152 | 53, 80, 443 |
| .128 | Apple TV | 5000, 49152, 49153 | 5000, 49152, 49153 (no change) |
| .130 | Range extender | 22, 80 | 22, 80 (no change) |
| .144 | My brother's Galaxy | – (not in the scan; it was in the router panel) | None |
| .147 | My MacBook | 5000 | 5000 (no change) |
| .150 | My mum's iPhone | – (not in the scan; it was in the router panel) | 49152 |

- **Router:** port 49152 no longer shows up on the router. It matches turning off UPnP, although I changed several settings at once and can't put it down to just one. The 49152 on my mum's iPhone belongs to that device, not the router: it's a port many devices use for their own services, so it's unrelated to the router's UPnP.
- **Number of devices:** 6 showed up before (7 in the repeat) and 8 after. The difference is because different people were at home and phones on standby sometimes don't answer.

**New findings**
- **.134:** a new IP that shares its MAC with the extender. It's the desktop PC, connected through it: the extender presents its own MAC on behalf of the devices behind it.
- **The router panel accepts HTTP (port 80):** if I log in that way, the password travels unencrypted over the local network. The right way is to always use HTTPS (`https://192.168.1.1`).
- **SSH open on the range extender:** it's its admin access. I left it as it was, since its password isn't the factory one.
- **Port 5000 on my MacBook:** it's the AirPlay receiver. It can be turned off in System Settings → General → AirDrop & Handoff → AirPlay Receiver. I haven't turned it off.

---

# Summary of findings and fixes

| Finding | Severity | Fix | Status |
|---|---|---|---|
| Factory password on the admin panel | High | New password | ✅ Fixed |
| WPS enabled | Medium | Disabled | ✅ Fixed |
| UPnP enabled | Low-medium | Disabled | ✅ Fixed |
| Firewall at Low (IPv4 and IPv6) | Low | Medium | ✅ Fixed |
| MacBook on 2.4 GHz (speed) | Performance | Moved to 5 GHz | ✅ Improved (that machine only) |
| Main Wi-Fi with factory password | Low-medium | – | ⏸️ Not changed (optional) |
| Panel reachable over HTTP | Low | Always log in over HTTPS | ⏸️ Not changed |
| Port 5000 (AirPlay) on the MacBook | Low | – | ⏸️ Not changed (optional) |

---

# Scope
The whole analysis was done from inside my network, which was the goal of the project: see what devices there are, which ports they have open and how the router is configured.

---

# What I learned
- **I learned to use nmap.** I'd never used it before, and now I know how to discover the devices on a network, see their ports and find out what's behind each one with options like `-sn`, `--top-ports`, `-sV` or `-Pn`.
- **There are lots of ports and connections I didn't know about.** DNS, SSH, AirPlay, high ports like 49152… Every device at home has its own "doors" open, and you don't always know why.
- **A router has much more inside than it seems.** At first sight it's a box with lights, but inside there's a firewall, DMZ, port forwarding, logs, remote management by the ISP…
- **I discovered features I didn't know, like WPS or UPnP.** They come enabled by default to make everything easier, but that doesn't mean they're the most secure option.
- **Putting my own ideas to the test.** I was convinced the .130 was my iPhone, and it was the range extender. Looking at the ports, the MAC and the name on the router cleared it up.
- **A scan doesn't see everything.** Phones on standby sometimes don't respond and use random MACs, so it's worth repeating the scan and comparing it with the router's list.
- **The IP isn't a reliable way to identify a device**, because it can change. The MAC and the name shown on the router are more reliable.
- **Measuring speed isn't that simple.** The connection fluctuates with the time of day and what's being used at home, so it's hard to get an exact measurement.
- **The band matters.** Moving the MacBook from 2.4 to 5 GHz multiplied its speed, but for devices far from the router it's better to stay on 2.4 GHz.
