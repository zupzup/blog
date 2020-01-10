I recently moved this blog from AWS (S3 / Cloudfront) to a minimal Ubuntu/Nginx setup. The lock-in was real, so I had to migrate not only my static website (rather simple) and the TLS certificate, but also the domain.

But after a couple of hours it was done and everything was running the same way as before with a Page Speed score of 100.

However, one problem still remained. As I mentioned in a [previous](https://zupzup.org/no-ads-no-tracking-one-roundtrip/) post, I don't run any trackers or ads on this site and don't ever plan to. The only stats I'm interested in regarding traffic are unique visitors to get a feeling of what people find useful/enjoyable to read.

On AWS I simply used the generated `Cloudfront` statistics for this, but now I had to come up with a new, preferably minimal, solution for this problem.

I quickly found [GoAccess](https://goaccess.io/), a powerful command line tool for access log analysis, which looked promising - I tried it on one of my Nginx access logs and was happy with the output. Setup and configuration for my needs was covered by following the installation instructions.

The only issue left was how to get my access logs from the server to this tool. Fortunately, my use-case is rather simple - I just want all of my existing access logs and look at the statistics every couple of weeks.

In order to be able to do that, I wrote the following simple script:

## Nginx Access Log Script

Prerequisites for this script are that you have a `sudo` user on the system where your logs are located and `ssh` access.

```bash
#!/bin/bash
HOST=some_ip_or_host
USER=someuser

echo "Deleting old tmp logs..."
ssh $USER@$HOST 'sudo rm -rf /tmp/nginx_logs/'

echo "Copying access logs to tmp..."
ssh $USER@$HOST 'sudo mkdir -p /tmp/nginx_logs/'
ssh $USER@$HOST 'sudo cp /var/log/nginx/access.* /tmp/nginx_logs/'

echo "Chowning logs..."
ssh $USER@$HOST 'sudo chown -R ${USER} /tmp/nginx_logs'

echo "Extracting logs..."
ssh $USER@$HOST 'for file in /tmp/nginx_logs/*.gz; do gunzip -c "${file}" > /tmp/nginx_logs/$(basename "${file}" .gz); done'

echo "Concatenating logs..."
ssh $USER@$HOST 'for file in /tmp/nginx_logs/*; do cat "${file}" >> /tmp/nginx_logs/access.concat.log; done'

echo "Downloading concatenated access log..."
scp $USER@$HOST:/tmp/nginx_logs/access.concat.log /tmp/bloglog.log

echo "Starting GoAccess..."
goaccess /tmp/bloglog.log
```

First, we configure the host and user, so we don't have to type them out every time. Then, we simply delete the previously copied logs. Of course this could be trivially optimized by only copying over new files, but for my current data volumes this is entirely unnecessary.

Then, we copy all access logs to a tmp folder, make ourselves the owner, so we can do some operations and then download it with scp.

Following that, we `gunzip` the zipped logs and concatenate them using `cat` into the `access.concat.log` file.

Once the file is written, we download it with `scp` and run `goaccess` with the target file.

That's it. Running this script will open `GoAccess` with your concatenated access logs. [Here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-goaccess-web-log-analyzer-with-apache-on-debian-7) is a nice tutorial on how to configure and use GoAccess.

## Conclusion

This is a very minimal solution for analyzing low-volume Nginx access logs. GoAccess has proven to be a great choice so far in terms of minimalism, speed and reliability (for my small use-case at least).

I am a big fan of minimal toolchains, especially for my own personal projects and while the solution outlined above could be optimized in many ways, it does exactly what I need in a reliable way - good enough for now and if more use-cases pop up, we'll see how well it can be adapted to those. ;)

#### Resources

* [GoAccess](https://goaccess.io/)
* [DigitalOcean Tutorial on GoAccess](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-goaccess-web-log-analyzer-with-apache-on-debian-7)

