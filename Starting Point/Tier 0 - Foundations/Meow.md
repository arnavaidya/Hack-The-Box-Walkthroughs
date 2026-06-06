# Meow

First machine on HTB. Very easy.

## Test Connection to target IP with an ICMP echo request

Command: `ping <target_ip>`

## Find open ports and services running on the target IP

Command: `nmap -Pn -sV <target_ip>`

## Brute forcing common usernames on a service

Command: `hydra -L <wordlist_path> -p "" <service_name>://<target_ip>`