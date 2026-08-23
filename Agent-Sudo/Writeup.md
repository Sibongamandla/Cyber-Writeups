Written by Sibongamandla Mnyandu

# Agent Sudo Writeup

## Introduction

Agent Sudo is a TryHackMe room themed around a secret underground server operated by rogue agents. What starts as a cryptic message about "codenames" on a web page turns into a methodical chain of User-Agent header manipulation to discover an agent's identity, FTP credential brute-forcing, multi-layered steganography to extract hidden messages and passwords, and finally exploiting a sudo security bypass (CVE-2019-14287) to escalate from a standard user to root.

This writeup walks through how I rooted the machine on TryHackMe (https://tryhackme.com/room/agentsudo) and recovered both flags.

## CTF Background

- **Target IP:** `10.80.143.38`
- **Attacker box:** Kali Linux
- **Tooling:** Nmap, Gobuster, Burp Suite, Hydra, Binwalk, zip2john, John the Ripper, 7z, CyberChef, Steghide, Google Reverse Image Search
- **Objective:** Enumerate the machine, crack hidden credentials through steganography, gain an SSH foothold, and escalate privileges to root.

---

## Reconnaissance & Enumeration

The first step is always mapping out what the target exposes. I ran an aggressive Nmap scan to identify open ports and service versions:

```bash
nmap -sS -sV -A -T4 -vv 10.80.143.38
```

![Running the Nmap scan](Screenshots/Screenshot%202026-08-23%20at%2018.11.23.png)

The scan revealed three open ports:

![Nmap discovers three open ports](Screenshots/Screenshot%202026-08-23%20at%2018.13.13.png)

- **Port 21/tcp (FTP):** vsftpd 3.0.3
- **Port 22/tcp (SSH):** OpenSSH 7.6p1 (Ubuntu)
- **Port 80/tcp (HTTP):** Apache httpd 2.4.29 (Ubuntu)

The detailed service enumeration confirmed the exact versions and revealed the HTTP page title as "Annoucement":

![Detailed Nmap service enumeration](Screenshots/Screenshot%202026-08-23%20at%2018.13.31.png)

---

## Web Enumeration: Finding Agent C

Navigating to `http://10.80.143.38` in the browser presented a message from "Agent R" instructing agents to use their own **codename** as the User-Agent to access the site:

![The web page from Agent R](Screenshots/Screenshot%202026-08-23%20at%2018.23.17.png)

The page explicitly told us the access mechanism: the `User-Agent` HTTP header. Before diving into header manipulation, I ran a quick Gobuster directory scan to check for hidden paths:

```bash
gobuster dir -u http://10.80.143.38/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt -t 15 -q
```

![Gobuster directory enumeration results](Screenshots/Screenshot%202026-08-23%20at%2018.40.05.png)

Gobuster turned up nothing interesting beyond `index.php`, confirming that the User-Agent header is the intended attack vector.

### Brute-Forcing the User-Agent with Burp Suite Intruder

The message was signed by "Agent R", so the codename format seemed to be a single letter. My first instinct was to try `User-Agent: R` since that is the agent who wrote the message. I intercepted the request in Burp Suite and swapped the header:

![Intercepting the request in Burp Suite with User-Agent set to R](Screenshots/Screenshot%202026-08-23%20at%2018.35.21.png)

This returned a different response: "What are you doing! Are you one of the 25 employees?" This confirmed that the server is actively checking the User-Agent value, but at first the "25 employees" part did not click for me:

![Response when using User-Agent R, mentioning 25 employees](Screenshots/Screenshot%202026-08-23%20at%2018.35.45.png)

I decided to try another letter and set the User-Agent to `A`. That request came back with a `200 OK` but no useful content. At that point it clicked: if Agent R's own codename is the letter R, then every agent probably maps to a letter of the alphabet. And "25 employees" makes perfect sense because there are 26 letters in total and Agent R (the sender) is already accounted for, leaving 25 others.

This meant I needed to test every letter from A to Z to find which agent has something interesting behind their page. Doing that manually, one letter at a time, would be tedious. The much faster approach was to automate it using Burp Suite's **Intruder**. I set it to Sniper mode, marked the `User-Agent` value as the payload position, and loaded A through Z as the payload list:

![Burp Intruder setup with A-Z payload list](Screenshots/Screenshot%202026-08-23%20at%2018.46.39.png)

The Intruder results made the answer obvious: every letter returned a `200 OK` except for the letter **C**, which returned a `302` redirect with a different response length:

![Intruder results showing C returns a 302 redirect](Screenshots/Screenshot%202026-08-23%20at%2018.47.59.png)

I followed the redirect by setting the `User-Agent` to `C` in Burp's Repeater:

![Burp Repeater with User-Agent set to C](Screenshots/Screenshot%202026-08-23%20at%2018.49.06.png)

The server redirected to `/agent_C_attention.php`, which exposed a message addressed to **chris** from Agent R. The message mentioned "agent J", a deal, and told chris to change his weak password:

![Agent C's attention page revealing the name chris and weak password hint](Screenshots/Screenshot%202026-08-23%20at%2018.49.42.png)

This gave me a confirmed username (`chris`) and a strong signal that his credentials could be brute-forced.

---

## Hash Cracking & Brute-Force

### FTP Password Brute-Force

With the username `chris` in hand and the hint about a weak password, I targeted the FTP service using Hydra with the `rockyou.txt` wordlist:

```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.80.143.38
```

Hydra cracked the password almost instantly: **crystal**

![Hydra brute-forcing FTP and finding chris:crystal](Screenshots/Screenshot%202026-08-23%20at%2018.58.47.png)

### Looting the FTP Server

I logged into the FTP server with the cracked credentials:

![Successful FTP login as chris](Screenshots/Screenshot%202026-08-23%20at%2019.02.35.png)

Listing the directory revealed three files:

![FTP directory listing showing To_agentJ.txt, cute-alien.jpg, and cutie.png](Screenshots/Screenshot%202026-08-23%20at%2019.02.47.png)

I downloaded all three files using `mget *`:

![Downloading all files from FTP](Screenshots/Screenshot%202026-08-23%20at%2019.04.32.png)

Reading `To_agentJ.txt` revealed a note from Agent C to Agent J, explaining that the alien photos are fake and that Agent J's login password is hidden inside one of the pictures:

![Contents of To_agentJ.txt, login password stored in a fake picture](Screenshots/Screenshot%202026-08-23%20at%2019.05.58.png)

The two images downloaded were `cute-alien.jpg` and `cutie.png`:

![cute-alien.jpg, a cartoon alien image](Screenshots/Screenshot%202026-08-23%20at%2019.07.31.png)

![cutie.png, a green alien sticker](Screenshots/Screenshot%202026-08-23%20at%2019.07.39.png)

### Extracting Hidden Data from cutie.png

I started by running `binwalk` on `cutie.png` to check for embedded files:

```bash
binwalk cutie.png
```

![Binwalk scan revealing an embedded zip archive inside cutie.png](Screenshots/Screenshot%202026-08-23%20at%2019.08.19.png)

Binwalk detected a **Zip archive** embedded at offset `0x8702`, containing a file named `To_agentR.txt`. I extracted it:

```bash
binwalk -e cutie.png
```

![Binwalk extraction output](Screenshots/Screenshot%202026-08-23%20at%2019.09.44.png)

The extraction produced a `_cutie.png.extracted/` directory containing the `8702.zip` file. However, the zip was password-protected:

![Extracted directory contents showing the zip file](Screenshots/Screenshot%202026-08-23%20at%2019.16.01.png)

### Cracking the Zip Password

I used `zip2john` to extract the hash from the zip file, then cracked it with John the Ripper:

```bash
zip2john 8702.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

John cracked the zip password: **alien**

![John the Ripper cracking the zip password as alien](Screenshots/Screenshot%202026-08-23%20at%2019.20.18.png)

I extracted the zip using `7z`:

```bash
7z x 8702.zip
```

![7z extraction with the cracked password](Screenshots/Screenshot%202026-08-23%20at%2019.22.53.png)

The extracted `To_agentR.txt` contained a message from Agent R to Agent C with an encoded string, `QXJlYTUx`:

![Listing the extracted files including To_agentR.txt](Screenshots/Screenshot%202026-08-23%20at%2019.26.35.png)

![Contents of To_agentR.txt with the encoded string QXJlYTUx](Screenshots/Screenshot%202026-08-23%20at%2019.27.03.png)

### Decoding the Steganography Password

The string `QXJlYTUx` looked like base64. I decoded it using CyberChef, which revealed: **Area51**

![CyberChef decoding QXJlYTUx from base64 to Area51](Screenshots/Screenshot%202026-08-23%20at%2019.28.59.png)

At this point I had a decoded string but was not sure what to do with it. The TryHackMe room's next question asked "What is the steg password?" which I did not understand at first. I searched what a "steg password" actually refers to and found that it is a steganography passphrase, a secret key used to lock or unlock data hidden inside an ordinary file like an image:

![Searching what a steg password is](Screenshots/Screenshot%202026-08-23%20at%2019.34.25.png)

![Google results explaining steganography passwords and tools like steghide](Screenshots/Screenshot%202026-08-23%20at%2019.35.16.png)

This told me exactly what I needed to do. The decoded string `Area51` must be the steganography passphrase, and I needed to use `steghide` to extract a hidden message from the `.jpg` image (steghide only works with JPEG and other specific formats, not PNG).

### Extracting the Hidden Message from cute-alien.jpg

I ran `steghide info` on `cute-alien.jpg` to confirm there was embedded data, entering `Area51` as the passphrase:

```bash
steghide info cute-alien.jpg
```

![steghide info revealing an embedded message.txt encrypted with rijndael-128](Screenshots/Screenshot%202026-08-23%20at%2019.35.46.png)

It confirmed a hidden file called `message.txt` was inside. I extracted it:

```bash
steghide extract -sf cute-alien.jpg
```

The extracted `message.txt` contained credentials for Agent J. His name is **james** and his SSH password is **hackerrules!**

![steghide extraction revealing james's SSH password: hackerrules!](Screenshots/Screenshot%202026-08-23%20at%2019.37.06.png)

---

## Capturing the User Flag

With the SSH credentials in hand, I logged into the target machine as james:

```bash
ssh james@10.80.143.38
```

![SSH login as james on the target machine](Screenshots/Screenshot%202026-08-23%20at%2019.41.10.png)

James's home directory contained two files: `user_flag.txt` and `Alien_autospy.jpg`:

```bash
cat user_flag.txt
```

```
b03d975e8c92a7c04146cfa7a5a313c7
```

![Reading the user flag](Screenshots/Screenshot%202026-08-23%20at%2019.42.31.png)

### Identifying the Photo Incident

To investigate `Alien_autospy.jpg`, I transferred it to my Kali box using `scp`:

```bash
scp james@10.80.143.38:Alien_autospy.jpg .
```

![SCP transfer of Alien_autospy.jpg and opening it with xdg-open](Screenshots/Screenshot%202026-08-23%20at%2019.58.52.png)

Opening the image revealed a still from the famous 1947 "alien autopsy" footage:

![The Alien_autospy.jpg image, a Roswell alien autopsy still](Screenshots/Screenshot%202026-08-23%20at%2019.59.04.png)

A reverse image search on Google confirmed this is from the **Roswell alien autopsy**, which is the answer to the room's incident question:

![Google reverse image search confirming Roswell alien autopsy](Screenshots/Screenshot%202026-08-23%20at%2019.59.57.png)

---

## Privilege Escalation

Back in the SSH session as james, I checked what sudo privileges were available:

```bash
whoami
id
sudo -l
```

![Checking sudo permissions, (ALL, !root) /bin/bash](Screenshots/Screenshot%202026-08-23%20at%2020.03.56.png)

The output was telling:

```
User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```

This sudoers rule means james can run `/bin/bash` as **any user except root**. At first glance this looks like it properly prevents root access, but I wanted to understand this pattern better and see if there was a way around it.

### Researching the (ALL, !root) Pattern

I searched for `(ALL, !root) /bin/bash` to learn more about what this configuration allows and whether it has any known weaknesses. The very first result on Exploit-DB pointed to **CVE-2019-14287**, a sudo security bypass:

![Google search results for the sudo bypass, CVE-2019-14287](Screenshots/Screenshot%202026-08-23%20at%2020.04.35.png)

The Exploit-DB page described how specifying user ID `-1` (or `#-1`) tricks sudo into running the command as root. This works because sudo converts `-1` to UID `0` (root) internally, but the exclusion check against `!root` happens *before* that conversion. So the `!root` rule never catches it.

Before attempting the exploit, I needed to confirm the target was actually running a vulnerable version. CVE-2019-14287 affects sudo versions prior to 1.8.28:

```bash
sudo --version
```

```
Sudo version 1.8.21p2
```

![Sudo version 1.8.21p2, confirmed vulnerable to CVE-2019-14287](Screenshots/Screenshot%202026-08-23%20at%2020.07.29.png)

Version 1.8.21p2 is well within the vulnerable range, so the exploit should work.

### Exploiting CVE-2019-14287

The exploit from the Exploit-DB page is a single command:

```bash
sudo -u#-1 /bin/bash
```

![Exploit-DB reference for the sudo -u#-1 bypass](Screenshots/Screenshot%202026-08-23%20at%2020.09.37.png)

Running it as james immediately dropped me into a root shell:

![Root shell obtained, uid=0(root)](Screenshots/Screenshot%202026-08-23%20at%2020.10.32.png)

### Root Flag

With a root shell, the standard next step is to check `/root/root.txt` for the final flag:

```bash
cat /root/root.txt
```

```
b53a02f55b57d4439e3341834d70c062
```

The message also revealed that **Agent R** is **DesKel**, the room's creator.

![Reading the root flag and discovering Agent R is DesKel](Screenshots/Screenshot%202026-08-23%20at%2020.12.39.png)

---

## Takeaways

This box chains together several distinct security weaknesses to demonstrate how seemingly minor issues compound into full system compromise:

1. **Information Disclosure via HTTP Headers:** The web application's reliance on User-Agent-based access control leaked agent identities. A simple Burp Suite Intruder sweep through the alphabet exposed the username `chris` and the structure of the agent naming scheme.
2. **Weak Credentials & Brute-Force Exposure:** Agent C's FTP password (`crystal`) fell to a basic dictionary attack in seconds. Publicly exposed services with weak passwords are low-hanging fruit for attackers.
3. **Steganography as Obfuscation, Not Security:** Hiding credentials inside images and behind base64 encoding only adds inconvenience, not protection. Once the images were accessible, standard forensic tools (binwalk, zip2john, steghide) extracted everything with minimal effort.
4. **Unpatched Software (CVE-2019-14287):** The most critical failure. The sudoers configuration `(ALL, !root) /bin/bash` was intended to prevent root access, but the server ran sudo 1.8.21p2 which is vulnerable to a trivial bypass. A single `sudo -u#-1 /bin/bash` command negated the entire privilege restriction. Patching sudo to version 1.8.28+ would have prevented this escalation entirely.

## References

- [TryHackMe: Agent Sudo](https://tryhackme.com/room/agentsudo)
- [CVE-2019-14287, sudo Security Bypass](https://nvd.nist.gov/vuln/detail/CVE-2019-14287)
- [Exploit-DB: sudo 1.8.27 Security Bypass](https://www.exploit-db.com/exploits/47502)
- [Steghide Documentation](https://steghide.sourceforge.net/)
- [Binwalk Firmware Analysis Tool](https://github.com/ReFirmLabs/binwalk)
