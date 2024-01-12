# Solution

This guide will walk you through the steps to complete the CTF.  
If you want to do it by yourself, then don't read it! It's cheating!


## Services discovery
The first thing to do when trying to break in a system is to analyse open ports and running services.
We can do this with `nmap`:

```shell
nmap -sS <VM_address>
```

We see a few open ports:
```
PORT     STATE    SERVICE
21/tcp   open     ftp
80/tcp   open     http
443/tcp  open     https
5432/tcp open     postgresql
9090/tcp filtered zeus-admin
```

We see that there's probably an FTP server running (port 21), as well as a web server (ports 80 and 443) and a PostgreSQL server (port 5432).

We'll try to connect to each of them one by one to try to gather some information about the machine, and, maybe, find vulnerabilities or misconfigurations / weak configurations.


## FTP server
Let's start with port 21, which is probably an FTP server. We'll try to connect to it using anonymous authentication (with FileZilla for example, or using the `ftp` command):

```shell
ftp -a <VM_address>
```
The `-a` flag forces anonymous authentication.

```
Connected to <VM_address>.
220 (vsFTPd 3.0.3)
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

Nice! It worked! Let's see what we have in here.

```shell
ftp> ls
229 Entering Extended Passive Mode (|||17856|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              50 Jan 08 21:29 notes.txt
226 Directory send OK.
ftp> 
```

There's a file named `notes.txt` in the FTP root. Let's view its content.

```shell
ftp> get notes.txt -
remote: notes.txt
229 Entering Extended Passive Mode (|||9033|)
150 Opening BINARY mode data connection for notes.txt (50 bytes).
TODO:
- update apache config
- rotate pg password
226 Transfer complete.
50 bytes received in 00:00 (138.32 KiB/s)
ftp>
```

Ok, so it looks like a note for the sysadmins.
Except this short note, it seems there's nothing else interesting in the FTP server. But the note states that the PostgreSQL password should be changed, so it's probably worth a try to try default and simple credentials.


## PostgreSQL
The nmap scan showed us that port 5432 was open, so we should find a PostgreSQL server here if default ports are used. We now connect to this port using `psql` and try to authenticate with some commonly used combinations of login and password (the default credentials for PostgreSQL used to be postgres/postgres, so we try that first):

```
psql -h <VM_address> -U postgres
```

And... it turns out it worked!

After successfully logging in, we can find the postgres databases 'postgres', 'template0' and 'template1'. These are here by default and mandatory to postgre's functions, so not very interesting, what is interesting though, is a database titled "users", bingo!
When looking inside, we can find an "users" table composed of two columns: `username` and `passhash`. We can see that the passwords are hashed. Hashes are 32-character hexadecimal strings, so it's probably MD5.

By copying the desired hash and putting in into some website like "hashanalyzer", we can learn that the hash used is actually MD5. One solution to retrieve the password would be to bruteforce it (using hashcat for example), a short password hashed with MD5 can be broken in just a few minutes after all...

...

It worked! We now know that the password of our user is simply "passw0rd1!", only 10 characters long, not very effective with a poor hashing system...


## Webserver

Let's get back to the webserver, we haven't explored down that road yet.
Ok, where do we start? At first sight, there's only a default apache page. We can search for some clues in the robots.txt file; for this we simply add `/robots.txt` to the end of the URL. This file is used to indicate to search engines which URLs of a website they are allowed (or disallowed) to access when indexing websites. Most often they are used to disallow access to certain pages, if you have a page that you don't want to be public for example.

Here, the robots.txt file contains:
```
User-agent: *
Disallow: /{{ cockpit_subdir }}
```

Interesting, `/cockpit_ui` is disallowed, so maybe we should explore that lead. Cockpit is a remote administration tool that provides a web interface with useful tools to manage a server (networking, storage, logs...)


## Cockpit

When accessing `/cockpit_ui`, we get a 404 error. Hum. Maybe we were put on a wrong track on purpose by the robots.txt file.
However, when we try with an added leading slash (`/cockpit_ui/`), we land on a Cockpit authentication page! Cockpit is protected by user/password authentication, and by default it uses UNIX logins.
We can try to login using the username and password we previously found ("user"/"passw0rd1!").

...

Login is successful! And now thanks to Cockpit we have access to a terminal via the web ui! There are many things that can be done from here but first let's check what privileges we have.

```
user@bookworm:~$ sudo -l
[sudo] password for user: 
Sorry, user user may not run sudo on bookworm.
user@bookworm:~$ groups
user docker
```

Ok, we neither are root nor in the sudo group, but we can see that we are in the docker group, which means we have full access to Docker, which is very powerful. We are very close to finding that flag (or at least having root access), since being in the docker group will allow us to elevate our privileges.


## Privilege escalation with Docker

Due to us being in the docker group we can run containers, a container is an isolated space where we can run applications in a controlled environment, similar to a virtual machine in some ways but more lightweight.

By creating a docker container we by default become the root user inside said container; and we will be using this to our advantage. We can create a docker container, and map the root filesystem (/) of the host to the /data directory inside the container:
```
docker run -it -v /:/data debian
```
To explain this command, we are starting a docker container from the debian image, and using the `-v` (`--volume`) option to mount / from the host to /data in the container.
The -it options give us an interactive command prompt to execute commands inside of the container.

We are root inside the container, and since we have the host root on /data, we now have access to everything. We can search for the flag manually or use some command such as find:
```
find /data -iname "*flag*"
```

The flag is located at 
```
/data/root/.flag
```

Congratulations!
