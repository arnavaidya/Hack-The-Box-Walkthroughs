# Redeemer

## Test Connection to target IP with an ICMP echo request

Command: `ping <target_ip>`

## Find all open ports beyond the first 1000 on the target IP

Command: `nmap -sS -p 1-10000 -sV -v <target_ip>`

## Specify Redis hostname

Command: `redis-cli -h <target_ip>`

## Obtain info and stats once connected to Redis server

Command: `info`

## Select a DB from Redis server

Command: `select`

## Obtain all keys in the DB

Command: `keys *`

## Check out a key

Command: `get <key_name>`