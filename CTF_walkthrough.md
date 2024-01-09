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
Except this short note, it seems there's nothing else interesting in the FTP server. But the note states that the PosgreSQL password should be changed, so it's probably worth a try to try default and simple credentials.
