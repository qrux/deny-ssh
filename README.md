What is deny-ssh?
=================

It's a shell script (bash) that blacklists the baddies that try brute-force SSH attacks.  It has no complex software dependencies; it's entirely implemented in shell tools (sed, grep, ls, awk, cut).

Doesn't DenyHosts aleady exist?
-------------------------------

Yes--and this effort was inspired by it.  It's great, but it relies on Python.  It's just another dependency, and as disk space and memory are again important considerations (as more and more people migrate toward VPS and cloud-solutions) I thought a tool which exists that has similar features but doesn't require YAPL would be useful.  So, I thank the folk(s) at DenyHosts for implementing an awesome idea and pointing out how valuable this kind of service can be.

How does it work?
-----------------

It creates entries in /etc/hosts.deny like this:

	ALL: 114.207.112.95
	ALL: 218.30.22.142
	ALL: 61.253.249.157

Of course, this only working on systems like Linux that properly support hosts.deny; some BSDs don't, so you'll have to port this.

It also writes a log file (/var/log/deny-ssh.log) that has entries that look like this.

	2012-0224 19:19:32 deny-ssh: 114.207.112.95 > 3 revmap_failures (35 x)
	2012-0224 19:20:33 deny-ssh: 222.89.142.222 > 5 invalid_user_attempts (235 x)
	2012-0224 19:20:33 deny-ssh: 61.253.249.157 > 7 frequency_in_10m_window (39 x)

It contains a script that does the heavy-lifting, a wrapper script that "daemonizes" the heavy-lifter, and a sample sysv-init script.  The wrapper does nothing more than look for the heavy-lifter and then output the contents of the heavy-lifter to a log file.  Since this utility was designed specfically to work with X/LAPP (https://github.com/qrux/xlapp), it relies on the functions in /lib/lsb/init-functions.  Since you may not have the functions "start-daemon" and "killproc", you may have to adapt it to your system.  Should be trivial, and I may include comments that help in that regard.

Deny-ssh looks in /var/log/auth.log for entries like this:

	Feb 23 08:12:15 host sshd[18858]: Failed password for root from 61.253.249.157 port 35849 ssh2
	Feb 23 08:12:22 host sshd[18866]: Failed password for root from 61.253.249.157 port 39911 ssh2
	Feb 23 08:12:29 host sshd[18875]: Failed password for root from 61.253.249.157 port 43947 ssh2
	Feb 23 08:12:35 host sshd[18883]: Failed password for root from 61.253.249.157 port 46722 ssh2
	Feb 23 08:12:41 host sshd[18891]: Failed password for root from 61.253.249.157 port 49053 ssh2
	Feb 23 08:12:48 host sshd[18899]: Failed password for root from 61.253.249.157 port 51371 ssh2
	Feb 23 08:12:54 host sshd[18907]: Failed password for root from 61.253.249.157 port 52842 ssh2
	Feb 23 23:13:46 host sshd[24669]: Invalid user trinity from 222.89.142.222
	Feb 23 23:13:51 host sshd[24681]: Invalid user pix from 222.89.142.222
	Feb 23 23:13:59 host sshd[24701]: Invalid user pix from 222.89.142.222
	Feb 23 23:14:02 host sshd[24709]: Invalid user alaurenzana from 222.89.142.222
	Feb 23 23:14:09 host sshd[24725]: Invalid user bristow from 222.89.142.222
	Feb 23 23:14:18 host sshd[24741]: Invalid user oracle from 222.89.142.222
	Feb 24 13:30:46 host sshd[4759]: reverse mapping checking getaddrinfo for 114-207-112-95.tongkni.co.kr [114.207.112.95] failed - POSSIBLE BREAK-IN ATTEMPT!

There are 3 types of failed attempts that generate a blacklist entry:

* *revmap* - A reverse-mapping error.
* *invalid* - Too many attempts to access invalid users.
* *frequency* - Too many attempts at existing users.

In the first, we just scan the ssh log for reverse-mapping errors.  If this number exceeds the threshold, we blacklist the offending host.

In the second, we just look at how many invalid-user errors in the ssh log.  If this number exceeds the threshold, we blacklist.

In the third, we have to look at the ssh log file, and break it down by 10-minute windows.  If, in any window, the number of failed attempts exceeds the threshold, we blacklist.

That's it!


What if I'm scurred?
--------------------

Totally fine.  I would be a little paranoid, too.  So, I include a CLI switch '-n'.  This shows you what deny-ssh sees in your ssh log, and doesn't write any changes to system files.  This is just a nice script to scan your logs to see if this is even an issue.

When I run it on my machine, I see this:

	host [~/lfs] # deny-ssh -n
	Using invalid-user threshold of 5
	Using interval-freq threshold of 7
	Using reverse-map threshold of 3
	119.57.72.26 > 5 invalid_user_attempts (12 x)
	124.117.239.67 > 7 frequency_in_10m_window (406 x)
	176.53.62.181 > 7 frequency_in_10m_window (36 x)
	180.153.139.4 > 5 invalid_user_attempts (212 x)
	202.103.30.24 > 7 frequency_in_10m_window (54 x)
	203.252.154.194 > 7 frequency_in_10m_window (498 x)
	208.115.198.173 > 5 invalid_user_attempts (239 x)
	217.199.212.245 > 7 frequency_in_10m_window (54 x)
	218.108.230.103 > 7 frequency_in_10m_window (45 x)
	218.30.22.142 > 7 frequency_in_10m_window (37 x)
	219.146.225.147 > 7 frequency_in_10m_window (530 x)
	221.233.134.15 > 5 invalid_user_attempts (783 x)
	222.87.204.13 > 5 invalid_user_attempts (276 x)
	222.89.142.222 > 5 invalid_user_attempts (235 x)
	37.53.255.158 > 7 frequency_in_10m_window (13 x)
	37.55.27.117 > 7 frequency_in_10m_window (13 x)
	37.55.89.53 > 7 frequency_in_10m_window (13 x)
	60.13.176.3 > 7 frequency_in_10m_window (406 x)
	61.145.116.154 > 7 frequency_in_10m_window (118 x)
	61.253.249.157 > 7 frequency_in_10m_window (39 x)
	64.16.224.103 > 7 frequency_in_10m_window (38 x)
	80.45.169.178 > 5 invalid_user_attempts (197 x)
	95.132.182.20 > 7 frequency_in_10m_window (13 x)
	114.207.112.95 > 3 revmap_failures (35 x)
	217.199.212.245 > 3 revmap_failures (16 x)
	218.28.228.30 > 3 revmap_failures (12 x)

This "scan-mode" can be run at any time--since it only reads the file and reports its findings, it can even be run while the daemon is running!

The scanning ability is fun--and useful in its own right!


Okay--I'm convinced.  How do I use this thing?
----------------------------------------------
I install the sysv-init links so that deny-ssh starts *BEFORE* ssh.

Enjoy!
======

