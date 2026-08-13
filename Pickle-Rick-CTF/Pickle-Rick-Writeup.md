Written by Sibongamandla Mnyandu

# Pickle Rick Writeup

## Introduction

Pickle Rick is a beginner-friendly TryHackMe room built around a gag from the show: Rick has turned himself into a pickle and lost the recipe to turn back. My job is to help Morty. Scattered across a web server are the three ingredients Rick needs, and prying them loose means chaining a few small web weaknesses into full command execution on the box. What starts as a cartoon joke ends with a root shell.

This writeup walks through how I rooted the machine on TryHackMe (https://tryhackme.com/room/picklerick) and recovered all three ingredients.

## CTF Background

- **Target IP:** `10.81.173.133`
- **Attacker box:** Kali Linux
- **Tooling:** Firefox, Nmap, Gobuster, browser Inspector
- **Objective:** Recover the three secret ingredients for Rick's potion.

The landing page sets the scene. Rick begs for help finding the "last three secret ingredients," and he can't remember his own password. That's a hint that credentials are lying around somewhere in plain sight.

![The Help Morty landing page](Screenshots/Screenshot%202026-08-13%20at%2020.06.19.png)

---

## Enumeration

Two habits pay off early on almost every web box: read the HTML the server hands you, and scan the ports underneath it.

Popping open the page source in the Inspector, Rick left himself a sticky note in an HTML comment. It hands over a username outright:

```
Note to self, remember username! Username: R1ckRul3s
```

![Username hidden in an HTML comment](Screenshots/Screenshot%202026-08-13%20at%2020.08.45.png)

While the browser did its thing, I pointed Nmap at the host. Only two doors are open, SSH on 22 and Apache on 80, and the web title "Rick is sup4r cool" confirms I'm in the right place.

```
nmap -sV 10.81.173.133
```

![Nmap results showing SSH and Apache](Screenshots/Screenshot%202026-08-13%20at%2020.24.17.png)

`robots.txt` is the next obvious stop, and it doesn't disappoint. Instead of crawler rules, it holds a single word:

```
Wubbalubbadubdub
```

That's not disallowed paths. That's a password, sitting in the open.

![robots.txt reveals a password](Screenshots/Screenshot%202026-08-13%20at%2020.28.29.png)

I still had no login page to use it on, so I turned Gobuster loose to map hidden directories against the `common.txt` wordlist.

```
gobuster dir -u http://10.81.173.133 -w /usr/share/wordlists/dirb/common.txt
```

![Gobuster running the common wordlist](Screenshots/Screenshot%202026-08-13%20at%2020.34.39.png)

The first pass surfaces `assets`, `index.html`, and a `server-status`. Useful context, but no login yet.

![Gobuster finishes the first pass](Screenshots/Screenshot%202026-08-13%20at%2020.38.27.png)

So I ran it again, this time asking for `.php` and `.txt` files specifically. That's what shakes the login page loose.

```
gobuster dir -u http://10.81.173.133 -w /usr/share/wordlists/dirb/common.txt -x php,txt
```

![Gobuster with php and txt extensions](Screenshots/Screenshot%202026-08-13%20at%2020.40.00.png)

Now the good stuff appears. `login.php` returns a 200, and a `denied.php` lurks behind a redirect.

![login.php and denied.php discovered](Screenshots/Screenshot%202026-08-13%20at%2020.43.46.png)

---

## Foothold

I now had every piece of the puzzle: a username from the page comment, a password from `robots.txt`, and a login page from Gobuster. Feeding `R1ckRul3s` / `Wubbalubbadubdub` into the portal drops me straight in.

![Logging into the portal](Screenshots/Screenshot%202026-08-13%20at%2020.45.18.png)

Past the door sits a "Command Panel," a text box that runs commands on the server and prints the output back to the page. This is the whole ballgame. The web app is handing me a shell in a browser field.

![The Command Panel after login](Screenshots/Screenshot%202026-08-13%20at%2020.45.33.png)

A quick detour explains the `denied.php` from earlier: it's a dead end that taunts anyone who isn't the "REAL rick," pickle artwork and all.

![The denied.php troll page](Screenshots/Screenshot%202026-08-13%20at%2020.45.46.png)

The panel also filters a few commands, which I confirmed by peeking at the portal's source before diving in.

![Viewing the portal source](Screenshots/Screenshot%202026-08-13%20at%2020.51.34.png)

---

## Ingredient Hunt

### First ingredient

Listing the working directory is always my opening move. `ls -la` exposes a file named to look important:

```
ls -la
```

![ls -la reveals the secret ingredients file](Screenshots/Screenshot%202026-08-13%20at%2020.57.20.png)

Reading `Sup3rS3cretPickl3Ingred.txt` gives up the first ingredient:

```
mr. meeseek hair
```

![First ingredient revealed](Screenshots/Screenshot%202026-08-13%20at%2020.58.39.png)

A neighbouring `clue.txt` nudges me onward with "Look around the file system for the other ingredient." Rick, ever helpful.

![clue.txt hints at exploring the filesystem](Screenshots/Screenshot%202026-08-13%20at%2021.02.02.png)

### Second ingredient

Following the clue, I walked up into `/home`. Two accounts live there, and `rick` is the one that matters.

```
ls -la /home
ls -la /home/rick
```

![Listing /home and rick's directory](Screenshots/Screenshot%202026-08-13%20at%2021.05.48.png)

Inside sits a file cheekily called `second ingredients`, spaces and all, which means it needs quoting.

![The second ingredients file located](Screenshots/Screenshot%202026-08-13%20at%2021.17.52.png)

My first instinct, `head`, gets slapped down. The panel disables it "to make it hard for future PICKLEEEE RICCCKKKK." A small speed bump, nothing more.

```
head "/home/rick/second ingredients"
```

![head is disabled by the panel](Screenshots/Screenshot%202026-08-13%20at%2021.29.27.png)

So I swapped tools. Where `head` is blocked, `less` sails right through and prints the file:

```
less "/home/rick/second ingredients"
```

Second ingredient, done:

```
1 jerry tear
```

![Second ingredient revealed with less](Screenshots/Screenshot%202026-08-13%20at%2022.00.29.png)

### Third ingredient

The last ingredient wasn't going to sit somewhere a normal user could reach. It'd be under `/root`, so I checked what privileges the web user actually holds:

```
sudo -l
```

The answer is the punchline of the whole box. The `www-data` account may run any command as root, no password required, spelled out plainly as `(ALL) NOPASSWD: ALL`.

![sudo -l shows full NOPASSWD root access](Screenshots/Screenshot%202026-08-13%20at%2022.01.56.png)

That misconfiguration hands me root. I listed `/root` with `sudo` and found the final file waiting:

```
sudo ls -la /root
```

![Listing /root reveals 3rd.txt](Screenshots/Screenshot%202026-08-13%20at%2022.02.45.png)

Reading it closes out the recipe:

```
sudo less /root/3rd.txt
```

```
3rd ingredients: fleeb juice
```

![Third ingredient revealed](Screenshots/Screenshot%202026-08-13%20at%2022.04.35.png)

All three ingredients are in hand. First **mr. meeseek hair**, then **1 jerry tear**, and finally **fleeb juice** from inside root's home. Rick can be Rick again.

---

## Takeaways

Every step here interlinks with the one before it, and none of them relied on a clever exploit. The whole compromise ran on carelessness.

Rick's real undoing was leaving a username in an HTML comment and a password in `robots.txt`. Anyone who bothered to read the page already held half the keys. That's a reminder worth repeating: comments and crawler files are readable by the whole world, so nothing sensitive belongs in either.

The Command Panel then took a bad situation and made it worse. Passing user input to the shell without sanitising it is textbook command injection, and it turned a login page into a remote shell. Blocklisting a command like `head` does nothing when `less`, `cat`, `tail`, or a dozen others still work, driving home that denylists are a weak substitute for real input handling.

The killing blow was that `sudo` rule. Giving `www-data` passwordless root means a foothold and a full takeover become the same event. Least privilege exists precisely so one careless web bug doesn't cost you the entire machine.

## References

- [TryHackMe: Pickle Rick](https://tryhackme.com/room/picklerick)
- [OWASP Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [GTFOBins (bypassing restricted commands)](https://gtfobins.github.io/)
