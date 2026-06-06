# Dancing

## Test Connection to target IP with an ICMP echo request

Command: `ping <target_ip>`

## Find open ports and services running on the target IP

Command: `nmap -Pn -sV <target_ip>`

## List SMB shares

Command: `smclient -L <target_ip>`

## List directories in SMB

Command: `ls`

## Change directory within SMB

Command: `cd <dir_name>`

## Download file from SMB

Command: `get <file_name>`