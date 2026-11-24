# Blocking SSH Attackers on AWS EC2 with CrowdSec

## Problem
Bots try to guess SSH passwords on public servers all day. The server needs to detect these attacks and block the attacker automatically.

## Solution
Install **CrowdSec** (a security engine that reads logs and detects attacks) together with a **firewall bouncer** (the part that blocks the attacker's IP in the firewall).

- **CrowdSec** = the detective. It detects the attack.
- **Bouncer** = the security guard. It blocks the attacker.
- **Decision** = a ban on one IP address for a period of time.

## Architecture

<img width="1657" height="896" alt="crowdsec-architecture" src="https://github.com/user-attachments/assets/6b2bcd43-63ce-4648-9495-665d91fed74d" />


| Server | Role |
|---|---|
| `victim-server` | Ubuntu EC2 with CrowdSec and the firewall bouncer installed |
| `attacker-server` | A second EC2 used only to send fake SSH logins |

## Steps
1. Launched two Ubuntu EC2 instances in the default VPC: `victim-server` and `attacker-server`
2. Installed CrowdSec on the victim server and checked that the service was running
3. Installed the firewall bouncer (`crowdsec-firewall-bouncer-iptables`) so bans are enforced in the firewall
4. Verified the bouncer and the detection rules (collections), including `crowdsecurity/sshd`
5. From the attacker server, sent repeated failed SSH logins with a fake username
6. Checked on the victim server what CrowdSec detected and which IPs it banned
7. Removed the ban and terminated both servers

## Commands used

```bash
sudo apt install crowdsec -y                         # install the detection engine
sudo apt install crowdsec-firewall-bouncer-iptables -y   # install the blocker
sudo systemctl status crowdsec                       # check CrowdSec is running
sudo cscli bouncers list                             # show connected bouncers
sudo cscli collections list                          # show detection rules (collections)
sudo cscli alerts list                               # attacks CrowdSec detected
sudo cscli decisions list                            # active bans
sudo cscli decisions delete --all                    # remove all bans
```

From the attacker server:
```bash
ssh -o StrictHostKeyChecking=no fakeuser@<victim-public-ip>   # failed login attempts
```

## Screenshots

CrowdSec bouncer and detection collections (sshd) working:

<img width="1365" height="675" alt="Screenshot 2026-11-24 073640" src="https://github.com/user-attachments/assets/c348bc5f-3842-470b-bc47-07743814cde5" />


Active ban list after the fake logins:

  <img width="1365" height="767" alt="Screenshot 2026-11-24 074816" src="https://github.com/user-attachments/assets/319307b5-a386-4b2e-a692-3356a4e22fa5" />
<img width="1365" height="278" alt="Screenshot 2026-11-24 074801" src="https://github.com/user-attachments/assets/8d2068b1-38ac-481d-b3bf-cad340660107" />


## Problems I faced and how I fixed them

| Problem | Cause | Fix |
|---|---|---|
| `command not found` when testing | I typed `fakeuser@ip` without the `ssh` command | Started the command with `ssh` |
| Syntax error near unexpected token | I typed the `< >` brackets, which are only placeholders | Typed the real IP address without brackets |
| No attack was detected | I ran the fake logins on the victim server instead of the attacker server | Ran them from the attacker server |
| Invalid security group rule | A rule was created as Custom TCP with port 0 | Deleted it, because SSH port 22 was already open |

## Key concepts
- CrowdSec reads logs, detects attacks and creates decisions. It does not block by itself.
- A bouncer enforces the decisions (here: the firewall bouncer using iptables).
- CrowdSec ignores private IP addresses by default, so the test uses a public IP.
- Never test from your own laptop, because you could ban your own IP.

## What I learned
- The difference between detection (CrowdSec) and remediation (bouncer)
- How to read alerts and decisions with `cscli`
- How SSH brute-force attacks show up in logs
- How to remove a ban

## Clean up
Removed the bans with `cscli decisions delete --all` and terminated both EC2 instances to avoid charges. In a real setup, SSH should be allowed only from trusted IP addresses.

## Tools
AWS (EC2, Security Groups), CrowdSec, Ubuntu Linux
