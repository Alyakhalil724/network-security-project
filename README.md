# Open-Source Firewall Project: nftables vs IPFire

A hands-on network security project in which I built, configured and attacked two open-source firewalls in an isolated virtual lab, then compared how they performed.

- **nftables**: a Linux kernel packet-filtering framework, configured by hand from the command line
- **IPFire**: a complete firewall distribution with a web interface and zone-based network design

Each firewall sat between a Kali Linux "attacker" and an internal Ubuntu web server. I tested them with real offensive tools (Nmap, dirb, hping3, curl, telnet) to see what an outside attacker could find or reach.

> Built for the Network Security module (CA1) in my third year of Digital Forensics & Cybersecurity at TU Dublin.

---

## Lab architecture

All machines ran in **Oracle VirtualBox** on isolated host-only and internal networks. Nothing was exposed to a real network.

### Setup 1: nftables

```
┌──────────────────┐        ┌─────────────────────────┐        ┌──────────────────────┐
│  Kali Linux      │        │  nftables firewall      │        │  Internal server     │
│  (attacker)      │──────▶ │  enp0s3: external       │──────▶ │  Ubuntu + web server │
│  host-only net   │        │  enp0s8: internal       │        │  10.0.0.10           │
│  192.168.56.0/24 │        │  NAT masquerade out     │        │                      │
└──────────────────┘        └─────────────────────────┘        └──────────────────────┘
```

### Setup 2: IPFire

```
┌──────────────────┐        ┌─────────────────────────┐        ┌──────────────────────┐
│  Kali Linux      │        │  IPFire                 │        │  Internal server     │
│  (attacker)      │──────▶ │  RED   192.168.3.1      │──────▶ │  Ubuntu + web server │
│  192.168.3.10    │        │  GREEN 192.168.1.1      │        │  192.168.1.100       │
└──────────────────┘        └─────────────────────────┘        └──────────────────────┘
```

IPFire's colour-coded zones split the network by trust level: **RED** is the untrusted external side and **GREEN** is the trusted internal LAN.

---

## nftables configuration

The ruleset uses a **default-deny forward chain**. Only established or related traffic and explicitly allowed flows pass between the networks.

```nft
table inet firewall {
    chain input {
        type filter hook input priority filter; policy accept;
        ct state { established, related } accept
        iifname "lo" accept
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state invalid drop
        ct state { established, related } accept
        iifname "enp0s8" oifname "enp0s3" accept                          # internal -> external
        iifname "enp0s3" oifname "enp0s8" ip daddr 10.0.0.10 tcp dport 80 accept   # web only
    }

    chain output {
        type filter hook output priority filter; policy accept;
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "enp0s3" masquerade
    }
}
```

| Chain | Purpose |
|---|---|
| `input` | Traffic aimed at the firewall itself; allows loopback and existing connections |
| `forward` | The key chain. Controls everything crossing between networks; **drop by default** |
| `output` | Traffic the firewall generates itself |
| `postrouting` | NAT masquerading so the internal server can reach out through the firewall |

## IPFire configuration

IPFire was configured through its web UI with a default-deny stance. Verified from the shell with `iptables -L -n -v`:

- **INPUT**: policy DROP, only related/established and loopback traffic accepted
- **FORWARD**: policy DROP, so nothing crosses zones unless explicitly allowed
- **OUTPUT**: policy ACCEPT for traffic the firewall generates

---

## Testing and results

### nftables

| Test | Tool | What I did | Result |
|---|---|---|---|
| External scanning | `nmap -sT` / `nmap -sS` | Scanned the firewall, then the internal server through it | Firewall showed only SSH (22) open. All 1000 ports on the internal server came back **filtered**, so its services were hidden |
| Application-layer attack | `dirb`, curl loop | Directory brute-force and 20 rapid HTTP requests at the web server | Every request **failed**; dirb couldn't connect at all |
| Denial of service | `hping3 -S --flood` | SYN flood of about 5.9 million packets at the firewall's external interface | **0 replies, 100% dropped**; the internal web service stayed healthy throughout |

### IPFire

| Test | Tool | What I did | Result |
|---|---|---|---|
| Stealth mode | `nmap -sU` | Scanned the internal server from Kali | **0 hosts up**, so the attacker couldn't even tell the server existed |
| Internal connectivity | `ping`, `curl` | Checked the server could reach the GREEN gateway and its own web service | Both worked; the internal network ran normally |
| External access | `curl`, `telnet` to port 80 | Tried to reach the web server from Kali | Connections **refused or silently dropped**, giving no information back to the attacker |

---

## nftables vs IPFire

| | nftables | IPFire |
|---|---|---|
| Type | Kernel packet-filtering framework | Complete firewall operating system |
| Management | Command line, hand-written rules | Web-based GUI |
| Strength | Fine-grained control, lightweight, fast | Easy to deploy, zone model, built-in IDS/IPS |
| Best for | Experienced admins, custom Linux setups | Small and medium networks, home labs |
| Learning curve | Steeper | Gentler |

---

## Problems I hit (and fixed)

- **Wrong subnet mask:** the internal server was isolated because it got a `/32` instead of a `/24`. Fixed by setting the mask explicitly.
- **IP conflict:** Kali and the firewall shared an adapter and ended up with the same IP. Fixed by moving Kali onto its own host-only network.
- **Firewall that blocked nothing:** my first nftables forward chain used `policy accept`. Switched it to `policy drop` with specific allow rules.
- **Bypassed firewall:** in the IPFire setup, all adapters were initially on the same internal network, so Kali could reach the server directly and skip the firewall. Fixed by separating the networks.
- **Rules lost on reboot:** IPFire rules had to be reapplied after restarts, a reminder of why persistent configuration matters in production.
- **Apache wouldn't install:** I used Python's built-in web server instead for testing.

## What I learned

- A default-deny policy with explicit allow rules is far safer than trying to block known-bad traffic.
- Segmentation only works if the virtual network design is right. One misplaced adapter let the attacker bypass the firewall entirely.
- Network-layer firewalls are excellent at hiding hosts and absorbing floods, but they can't tell good HTTP from bad. Application-layer threats need deeper inspection, such as an IDS/IPS or a WAF.

---

## Tools

`nftables` · `IPFire` · `Oracle VirtualBox` · `Kali Linux` · `Ubuntu Server` · `Nmap` · `dirb` · `hping3` · `curl` · `telnet`

## Repository contents

```
├── README.md
└── Network_Security_CA1.docx   # Full report: configuration, screenshots and test evidence
```

## Authors

- **Alya Khalil**: [LinkedIn](https://www.linkedin.com/in/alya-khalil-10b755337/)
- **Benjamin Oludipe**

---

**Disclaimer:** All testing was done in an isolated virtual lab on machines I own and configured. 
