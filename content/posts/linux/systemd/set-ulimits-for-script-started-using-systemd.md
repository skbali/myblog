+++
title = "Set ulimits for script started using systemd"
summary = "How to set ulimits for a script started using systemd"
date = 2019-09-24T15:50:00Z   
tags = ["linux", "script", "syslog", "flock", "queue", "shell"]
canonicalURL = "https://blog.skbali.com/2019/09/set-ulimits-for-a-script-started-using-systemd/"
ShowReadingTime = true
draft = false
+++

In my previous [post](/posts/linux/systemd/start-script-on-boot-using-systemd/) I demonstrated how easy it was to start a script on boot using `systemd` in Linux. 

In this post I am going to show how you can set `ulimits` for the same script at startup. I will use the same script I used earlier even though it does not need any higher limits.

I have a script `monitor.sh` that simply runs a curl command in an endless while loop. This particular example does not do anything other than run the curl, but you could easily add a check and send alert if the curl statement fails

Let's write our monitor.sh script first which runs the curl in an endless loop. Using an editor you are comfortable with, save the following code in a file `/home/ubuntu/monitor.sh`

```bash
#!/bin/bash

while true;
do
    echo "Running monitor..."
    curl -o /dev/null -k https://example.com &> /dev/null
    sleep 600
done
```

Make sure you have execute permission set.
```bash
chmod u+x /home/ubuntu/monitor.sh
```

Next, we will set up the systemd configuration file. Create a new file sudo vi `/etc/systemd/system/mymonitor.service`. 

```bash
[Unit]
Description=My Monitor Service
After=network.target

[Service]
User=ubuntu
Group=ubuntu
ExecStart=sh -c '/home/ubuntu/monitor.sh'
Restart=always
RestartSec=5
LimitNOFILE=1024:4096

[Install]
WantedBy=multi-user.target
```
Let's examine the new lines I added to the  configuration.

In the `[Service]` section I added LimitNOFILE to set soft limit of 1024 and hard limit of 4096. This sets the soft and hard limit for number of open files by this process.

If you just want to set a higher hard limit you could change it to `LimitNOFILE=10240`

Let's go ahead and reload the systemd daemon to ensure it picks up the new configuration file. Stop and start the service.

```bash
sudo systemctl daemon-reload
sudo systemctl stop mymonitor.service
sudo systemctl enable mymonitor.service
sudo systemctl start mymonitor.service
```

Now Let's check the ps and limits again.
```bash
ps -ef | grep monitor

Use the pid from above command to check the limits

root@ip-172-30-1-14:/home/ubuntu# cat /proc/379/limits
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             31353                31353                processes
Max open files            1024                 4096                 files
Max locked memory         8388608              8388608              bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       31353                31353                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```
You can see that the soft and hard limit has changed for this process.

We can also set other limits if we wanted to change them. [See this page for other properties](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)

## Conclusion
Changing `ulimits` for a process that starts up on boot can be easily done in your `systemd` unit file.

**Note**: 
- If you make any changes to your script monitor.sh, make sure to stop and start your service so that a new process is launched.
- Disable the service if not needed.

## Further Reading
- [Wiki Systemd](https://wiki.archlinux.org/title/Systemd)
- [Systemd service](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)


