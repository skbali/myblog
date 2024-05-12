+++
title = "Queue up multiple instances of a shell script"
summary = "How to queue up multiple instances of the same shell script on linux server"
date = 2019-03-09T17:54:00Z   
tags = ["linux", "script", "syslog", "flock", "queue", "shell"]
ShowReadingTime = true
draft = false
+++

In this post, I am going to show you how to queue up multiple instances of a shell script so that only one instance is active at a time. 
The other instances will queue up and run as soon as an active script finishes. 
I will also show you how to make sure that the scripts do not lock up for a very long time.

## Background
Recently I came across a requirement in a project where a shell script had to run many times on a server. This script needed to use some local resource that could only be accessed by a single job at a time. I had to make sure that the multiple copies of the shell script were not active at the same time.

My traditional locking method for a bash script was to check for existence of the same script and exit. This approach was not going to work here as I wanted to queue up the jobs and not terminate.

## Procedure
Instead of a check for process and exit approach, I used `flock` to help queue up my jobs. This approach allowed multiple copies of the same script to wait for up to 5 minutes to acquire a lock.

My sample shell script is called myQueue.sh and the code is listed below.

### Script
Save the following code in a file `myQueue.sh`

```bash
#!/bin/bash

mySelf=$(basename $0)
lock="/var/lock/${mySelf}"

exec {fd}>$lock
flock --timeout 300 "$fd" || exit 1

echo "$$ ... Starting script..." | logger -t $mySelf
echo "$$ ... ... running" | logger -t $mySelf
sleep 10
echo "$$ ... ... sleeping" | logger -t $mySelf
echo "$$ ... Ending script..." | logger -t $mySelf

exit 0
```
Add permission to execute the script
```bash
chmod u+x myQueue.sh
```

Let's take a closer look at the code and how it works.
```bash
mySelf=$(basename $0)
lock="/var/lock/${mySelf}"
```
I am defining a variable to get the current script name and I plan to create a lock file under `/var/lock`. This lock file will be named after the script name.
```bash
exec {fd}>$lock
flock --timeout 300 "$fd" || exit 1
```
I use exec to get a file handle on the lock file I defined earlier. The `{fd}` variable gets an unused file handle for my file. 
The flock statement gets an exclusive lock on the file, and I am willing to wait up to 300 seconds to acquire the lock. If this fails the script will exit.

Without the `--timeout` option it is possible that you may end up a infinite wait if you do not specify any other options.
```bash
echo "$$ ... Starting script..." | logger -t $mySelf
echo "$$ ... ... running" | logger -t $mySelf
sleep 10
echo "$$ ... ... sleeping" | logger -t $mySelf
echo "$$ ... Ending script..." | logger -t $mySelf

exit 0
```

The rest of the code up to `exit 0`, is where your statements should be which need exclusive access to a resource. At the end of the script the previously acquired lock is released.

**Note:** If the script dies unexpectedly or if the server reboot, the lock is released.

In my sample code I added a sleep, some echo statements and I am using logger to output the echo to syslog.

Now I am going to submit a few copies of this job in the background by running `./myQueue.sh &`

```bash
jobs
[1]   Running                 ./myQueue.sh &
[2]   Running                 ./myQueue.sh &
[3]   Running                 ./myQueue.sh &
[4]   Running                 ./myQueue.sh &
[5]-  Running                 ./myQueue.sh &
[6]+  Running                 ./myQueue.sh &
```
In the output below you can clearly see that only one instance is active after the flock and subsequent jobs start up only after an active job finishes

```bash
Apr 07 15:11:27 ip-172-30-1-14 myQueue.sh[5088]: 5080 ... Starting script...
Apr 07 15:11:27 ip-172-30-1-14 myQueue.sh[5091]: 5080 ... ... running
Apr 07 15:11:37 ip-172-30-1-14 myQueue.sh[5557]: 5080 ... ... sleeping
Apr 07 15:11:37 ip-172-30-1-14 myQueue.sh[5559]: 5080 ... Ending script...
Apr 07 15:11:37 ip-172-30-1-14 myQueue.sh[5561]: 5148 ... Starting script...
Apr 07 15:11:37 ip-172-30-1-14 myQueue.sh[5563]: 5148 ... ... running
Apr 07 15:11:47 ip-172-30-1-14 myQueue.sh[5566]: 5148 ... ... sleeping
Apr 07 15:11:47 ip-172-30-1-14 myQueue.sh[5568]: 5148 ... Ending script...
Apr 07 15:11:47 ip-172-30-1-14 myQueue.sh[5570]: 5210 ... Starting script...
Apr 07 15:11:47 ip-172-30-1-14 myQueue.sh[5572]: 5210 ... ... running
Apr 07 15:11:57 ip-172-30-1-14 myQueue.sh[5575]: 5210 ... ... sleeping
Apr 07 15:11:57 ip-172-30-1-14 myQueue.sh[5577]: 5210 ... Ending script...
Apr 07 15:11:57 ip-172-30-1-14 myQueue.sh[5579]: 5273 ... Starting script...
Apr 07 15:11:57 ip-172-30-1-14 myQueue.sh[5581]: 5273 ... ... running
Apr 07 15:12:07 ip-172-30-1-14 myQueue.sh[5588]: 5273 ... ... sleeping
Apr 07 15:12:07 ip-172-30-1-14 myQueue.sh[5590]: 5273 ... Ending script...
Apr 07 15:12:07 ip-172-30-1-14 myQueue.sh[5592]: 5383 ... Starting script...
Apr 07 15:12:07 ip-172-30-1-14 myQueue.sh[5594]: 5383 ... ... running
Apr 07 15:12:17 ip-172-30-1-14 myQueue.sh[5616]: 5383 ... ... sleeping
Apr 07 15:12:17 ip-172-30-1-14 myQueue.sh[5618]: 5383 ... Ending script...
Apr 07 15:12:17 ip-172-30-1-14 myQueue.sh[5620]: 5446 ... Starting script...
Apr 07 15:12:17 ip-172-30-1-14 myQueue.sh[5622]: 5446 ... ... running
Apr 07 15:12:27 ip-172-30-1-14 myQueue.sh[5625]: 5446 ... ... sleeping
Apr 07 15:12:27 ip-172-30-1-14 myQueue.sh[5627]: 5446 ... Ending script...
```
## Other uses
Say you wanted to modify this to do a traditional lock, where if you cannot get the lock, your script exits right away. Modify the flock as shown below to achieve this.

```bash
flock -n "$fd" || exit 1
```

The `-n` option makes it fail rather than wait if the lock cannot be immediately acquired.

Another possibility is that you only want certain areas of your script to have exclusive access. You can modify your code as shown below

```bash
exec {fd}>$lock
flock --timeout 300 "$fd" || exit 1
# some task here that needs exclusive lock
flock -u "$fd"
# No more lock here
```

The `-u` option releases the previously acquired lock. 

## Conclusion
Using `flock`, I can queue up multiple instances of a shell script or terminate secondary copies. I am now actively using flock to control certain parts of my shell scripts. 

## References
- [Stack Overflow](https://stackoverflow.com/questions/8297415/in-bash-how-to-find-the-lowest-numbered-unused-file-descriptor)
- [flock](https://www.man7.org/linux/man-pages/man2/flock.2.html)