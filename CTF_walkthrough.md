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
The nmap scan showed us that port 5432 was open, we should find a PostgreSQL server here. We now connect to this port and try to authentify ourselves with some commonly used combinations of login and password.

-> postgres/postgres fonctionne

After successfully logging in, we can find the postgres databases : template0 and template1. These two are here by default and mandatory to postgre's functions, so not very interisting, what is interesting though, is a database titled "users", bingo !
When looking inside, we can find the user table composed of two rows, user and password we can see that the passwords row is hashed.

By copying the desired hash and putting in into some website like "hashanalyzer", we can learn that the hash used is md5 one solution to retrieve the password would be to bruteforce it, a short password hashed with md5 can be broken in just a few minutes after all...

...

It worked ! We now know that the password of our user is simply "passw0rd1!", only 10 characters long, not very effective with a poor hashing system...

(utiliser psql --host=<VM_address>)
(\l pour lister les db)
(\c pour se connecter à une db)
(\d pour lister les tables)

## Webserver

We now have access to the webserver, but where do we go from here ? We can search for some clues in the robots.txt, for this we simply add /robots.txt to the end of the URL, these pages are used by search engines to indicate what URL of a website they are allowed to access when responding to queries. Most often they are used to disallow access to certain pages like if you had a page that you didn't want to be public.

```
content of robots.txt
```

Interesting, /cockpit_ui is disallowed maybe we should try to explore that lead.

## Cockpit

We're now on the /cockpit_ui page, we can now login using the username and password we previously found (passw0rd1!)

...

Login is successful ! And now we have a access to terminal, there are many things that can be done from here but first let's check what privileges we have.

```
privileges command + results
```

Ok we are not root but we can see that we have access to docker which is very powerful, we are very close to finding that flag.

## Privilege escalation with Docker

Due to us being in the docker group we can run containers, a container is an isolated space where we can run applications in a controlled environment, similar to a virtual machine in some ways but more lightweight.

By creating a docker container we automatically become the root user of said container we will be using this to our advantage with the simple command
```
docker run -it -v /:/data debian
```
To explain this command, we are building a docker container called debian which will be build with the image of the data folder. 
The option -it gives us an interactive command prompt to execute commands inside of the container.

We gained root privileges we now have access to everything, we could search for the flag manually or use some command such as :
```
find /data -iname "*flag*"
```

The flag is located at 
```
/data/root/.flag
```

Congratulations !

