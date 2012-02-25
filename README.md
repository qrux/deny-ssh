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

	Feb 23 08:12:15 ntp1 sshd[18858]: Failed password for root from 61.253.249.157 port 35849 ssh2
	Feb 23 08:12:22 ntp1 sshd[18866]: Failed password for root from 61.253.249.157 port 39911 ssh2
	Feb 23 08:12:29 ntp1 sshd[18875]: Failed password for root from 61.253.249.157 port 43947 ssh2
	Feb 23 08:12:35 ntp1 sshd[18883]: Failed password for root from 61.253.249.157 port 46722 ssh2
	Feb 23 08:12:41 ntp1 sshd[18891]: Failed password for root from 61.253.249.157 port 49053 ssh2
	Feb 23 08:12:48 ntp1 sshd[18899]: Failed password for root from 61.253.249.157 port 51371 ssh2
	Feb 23 08:12:54 ntp1 sshd[18907]: Failed password for root from 61.253.249.157 port 52842 ssh2
	Feb 23 23:13:46 ntp1 sshd[24669]: Invalid user trinity from 222.89.142.222
	Feb 23 23:13:51 ntp1 sshd[24681]: Invalid user pix from 222.89.142.222
	Feb 23 23:13:59 ntp1 sshd[24701]: Invalid user pix from 222.89.142.222
	Feb 23 23:14:02 ntp1 sshd[24709]: Invalid user alaurenzana from 222.89.142.222
	Feb 23 23:14:09 ntp1 sshd[24725]: Invalid user bristow from 222.89.142.222
	Feb 23 23:14:18 ntp1 sshd[24741]: Invalid user oracle from 222.89.142.222
	Feb 24 13:30:46 ntp1 sshd[4759]: reverse mapping checking getaddrinfo for 114-207-112-95.tongkni.co.kr [114.207.112.95] failed - POSSIBLE BREAK-IN ATTEMPT!

There are 3 types of failed attempts that generate a blacklist entry:

* *revmap* - A reverse-mapping error.
* *invalid* - Too many attempts to access invalid users.
* *frequency* - Too many attempts at existing users.

In the first, we just scan the ssh log for reverse-mapping errors.  If this number exceeds the threshold, we blacklist the offending host.

In the second, we just look at how many invalid-user errors in the ssh log.  If this number exceeds the threshold, we blacklist.

In the third, we have to look at the ssh log file, and break it down by 10-minute windows.  If, in any window, the number of failed attempts exceeds the threshold, we blacklist.

That's it!

I install the sysv-init links so that deny-ssh starts *BEFORE* ssh.  Enjoy!


FAQ
---

(Coming soon!)

