# Simple CTF — TryHackMe Write-up

## Introduction

Hi everyone!
This is my first CTF write-up, documenting my journey through the **Simple CTF** room on TryHackMe.
I'm still learning, and I thought it would be a great idea to start documenting my practical experience, the challenges I encounter, and the concepts I discover along the way.
Let's learn together and document our progress! ❤️

**Room:** [Simple CTF — TryHackMe](https://tryhackme.com/room/easyctf)

---

## 1. Reconnaissance
<img width="1645" height="123" alt="image" src="https://github.com/user-attachments/assets/ad77ebc2-d454-4cbc-8819-5470ecfaf097" />

The first step is reconnaissance: gathering information about our target and identifying the services running on it.
We start by using **Nmap**, a network scanning tool, to discover open ports and identify the services and their versions.

```bash
nmap -sC -sV -p- <target_ip>
```

- `-sC`: Runs default NSE scripts.
- `-sV`: Attempts to identify service versions.
- `-p-`: Scans all TCP ports.

The scan reveals three open ports:
<img width="1165" height="361" alt="image" src="https://github.com/user-attachments/assets/a28f894b-7e30-4abf-9f41-ead5ecd3fa22" />

| Port | Service |
|---|---|
| 21 | FTP |
| 80 | HTTP |
| 2222 | SSH |

Two ports are below port 1000, while one is above it.

**Question: How many ports are open below port 1000?**

Answer: `2`

We also discover that SSH is running on the non-standard port 2222.

## 2. Enumeration

Now that we know which services are running, we move to enumeration.

First, let's investigate the HTTP service by visiting the target's website.

To discover hidden directories and files, we use **Gobuster**.

```bash
gobuster dir -u <target_URL> -w <wordlist>
```
<img width="1114" height="477" alt="image" src="https://github.com/user-attachments/assets/5744a055-e3c2-4eba-82fc-2b6f3b6563cd" />

Gobuster performs directory enumeration by testing potential paths against the web server.

The scan reveals an interesting directory:

`http://<target_ip>/simple/`

The server responds with HTTP status code `200`, meaning the requested resource was successfully accessed.

Let's investigate it!
<img width="1507" height="804" alt="image" src="https://github.com/user-attachments/assets/bc38a504-7124-47d4-b69f-072261333d1b" />

While exploring the website, we discover that it is running:

**CMS Made Simple version 2.2.8**

This is an important discovery because older versions of CMS Made Simple have documented security vulnerabilities.

## 3. Vulnerability Analysis

After identifying the CMS and its version, we search for known vulnerabilities affecting it.

One interesting vulnerability is:

**CVE-2019-9053 — SQL Injection**
<img width="1618" height="900" alt="image" src="https://github.com/user-attachments/assets/675a1842-014d-44ec-af2a-debd5abd70fb" />

This vulnerability affects CMS Made Simple versions up to 2.2.9.

SQL Injection (SQLi) occurs when an application handles user input unsafely, potentially allowing an attacker to manipulate database queries.
<img width="1839" height="226" alt="image" src="https://github.com/user-attachments/assets/78ea97dc-130a-4c05-9691-df843e37765c" />

Let's answer the room's questions:

**What's the CVE you're using against the application?**

Answer: `CVE-2019-9053`

**To what kind of vulnerability is the application vulnerable?**

Answer: `SQL Injection (SQLi)`

## 4. Exploitation

After identifying the vulnerability, we move to exploitation.
Our objective is to investigate whether we can exploit the SQL injection to retrieve sensitive information from the application.
<img width="1821" height="607" alt="image" src="https://github.com/user-attachments/assets/bd0b82af-17cb-42b3-b305-b5dfbadee62a" />

### Finding the exploit

We use SearchSploit to search for publicly available exploits associated with the CVE.

```bash
searchsploit --cve 2019-9053
```
<img width="1162" height="514" alt="image" src="https://github.com/user-attachments/assets/cf94362e-66f1-432e-a59b-973fb86c6fa5" />

We find an exploit and copy it into our current working directory:

```bash
searchsploit -m 46635.py
```

Then, we inspect the downloaded file:

```bash
ls -l 46635.py
```

Before executing an exploit, it's important to understand what it does and how to use it.

We try to display its help menu:

```bash
python 46635.py -h
```

However, we encounter a `SyntaxError`.

After investigating, we notice that the script uses Python 2 syntax, particularly in its `print` statements, while we're trying to execute it using Python 3.
We start adapting the syntax, but keep in mind that additional Python 2 compatibility issues may also need to be addressed.
After inspecting the script, we identify its usage syntax:
<img width="1254" height="94" alt="image" src="https://github.com/user-attachments/assets/21709865-d8a5-4aba-a2b5-5b0c333384eb" />

```bash
python 46635.py -u <target_URL> --crack -w <wordlist_path>
```

Let's try it against our target!

### Extracting credentials

The exploit successfully retrieves several pieces of information, including:

- A username
- An email address
- A salt
- A password hash

However, the password cracking process does not complete successfully because of an error in the exploit.

At this point, we have two possible approaches:

**Method 1 — Fix the exploit**

We can investigate the remaining errors, correct the script, and attempt to crack the extracted password hash.
<img width="808" height="196" alt="image" src="https://github.com/user-attachments/assets/f2fb0685-7431-4a44-b833-c4e42c25c08e" />

**Method 2 — Try SSH password brute-forcing**

Since we have discovered an SSH service on port 2222 and obtained a username, we can investigate whether a password from a wordlist allows authentication.
For this, we use Hydra:

```bash
hydra -l <username> -P /usr/share/wordlists/rockyou.txt ssh://<target_ip>:2222
```

Hydra attempts authentication using the supplied username and passwords from the wordlist.

And we get a match!

**The password is:** `secret`

<img width="1884" height="307" alt="image" src="https://github.com/user-attachments/assets/db1da9df-240f-44e7-97f9-931be112f591" />

### Answering the room's question

**Where can you log in with the details obtained?**

Answer: `SSH`

## 5. Initial Access

Now that we have the credentials, we can connect to the target through SSH.

```bash
ssh mitch@<target_ip> -p 2222
```
<img width="578" height="297" alt="Capture d&#39;écran 2026-11-01 130332" src="https://github.com/user-attachments/assets/e0fcc3ee-ddce-4904-9750-9532c6429f98" />

We enter the password we discovered.

And... tada! We're now logged in as `mitch`!

Our initial access is successful.

### Exploring the machine

Let's start by checking our current directory and exploring the filesystem.
<img width="850" height="286" alt="image" src="https://github.com/user-attachments/assets/3d368d81-7650-4a2a-ad30-ff4b77909776" />

We can find the user flag using:

```bash
cat user.txt
```

We can also explore the home directory to identify other users:

```bash
cd /home
ls
```

We discover another user named `sunbath`.

**What's the user flag?**

Answer: `G00d j0b, keep up!`

**Is there any other user in the home directory? What's its name?**

Answer: `sunbath`

## 6. Privilege Escalation

We now have access to the machine as a standard user, but our objective isn't complete.
We need to investigate whether we can escalate our privileges and obtain root access.

First, let's check which commands our current user is allowed to execute with sudo:

```bash
sudo -l
```

This command lists the sudo privileges available to the current user.

And here's where things get interesting!
<img width="723" height="88" alt="image" src="https://github.com/user-attachments/assets/edda17c2-af9e-40d9-ba35-5ff9658c5239" />

We discover that `mitch` can execute **Vim as root without providing a password**.

Why is this a security issue?

Vim is a text editor, but it also supports executing shell commands. When Vim is launched with root privileges, those commands can inherit those privileges.
This means that unrestricted sudo access to Vim can be abused to obtain a root shell.

### Using GTFOBins

To understand how to exploit this configuration, we consult [GTFOBins](https://gtfobins.org/), a reference containing techniques for abusing Unix binaries when they have specific privileges or configurations.
<img width="1477" height="781" alt="image" src="https://github.com/user-attachments/assets/55658a58-7875-41a6-947a-d8d19e45e975" />

Since we can execute Vim through sudo, we look for its sudo exploitation technique.
We can use:

```bash
sudo vim -c ':!/bin/sh'
```

This launches Vim as root and executes a shell command, allowing us to obtain a privileged shell.
We can verify our privileges using:

```bash
whoami
```

If everything worked, the output should be:

```text
root
```
<img width="585" height="132" alt="image" src="https://github.com/user-attachments/assets/475af1c2-5ce4-42f7-aba2-1c81063d3f7f" />
We've successfully escalated our privileges!

### Finding the root flag

Now we need to locate the root flag.

We can search for it using:

```bash
find / -name "root.txt" 2>/dev/null
```

- `find /`: Searches from the root of the filesystem.
- `-name "root.txt"`: Looks for files with that name.
- `2>/dev/null`: Redirects error messages to avoid displaying permission-related errors.

Once we find the file, we read its contents:

```bash
cat <root_flag_path>
```
<img width="633" height="325" alt="image" src="https://github.com/user-attachments/assets/91234175-6aae-4297-bc79-da4a1f265924" />
And we've reached our final objective!

**What's the root flag?**

Answer: `W3ll d0n3. You made it!`

---

## 7. Conclusion & Lessons Learned

And that's it! We've successfully completed the Simple CTF room.

This was a great first practical exercise because it introduced me to several important stages of a penetration-testing workflow.

Some of the key things I learned:

- **Reconnaissance:** Identifying open ports and running services using Nmap.
- **Enumeration:** Discovering hidden directories and identifying the technologies used by a web application.
- **Vulnerability Analysis:** Recognizing outdated software and researching its known vulnerabilities.
- **Exploitation:** Investigating a SQL injection vulnerability and understanding how credentials can be exposed.
- **Initial Access:** Using discovered credentials to access a remote machine through SSH.
- **Privilege Escalation:** Identifying dangerous sudo permissions and abusing Vim to obtain root access.

One important takeaway is that a seemingly harmless permission can become a serious security risk when configured incorrectly.

This is also a good reminder that understanding *why* a command works is just as important as knowing how to execute it.

Thanks for following along! More wr
