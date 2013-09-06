How to write to a local socket?
===============================
Friday, August 27th, 2010
One member of the rsyslog comunity wrote:

I’d like to forward via a local UNIX domain socket, instead. I think  I understand how to configure the ‘imuxsock’ module so my unprivileged instance reads from a non-standard socket location. But I can’t figure out how to tell my root instance to forward via a local domain socket.

I didn’t figure out a completely RSyslog-native method, but another poster’s message pointed me toward ‘socat’ and ‘omprog’, which I have working, now. (It would be really nice if RSyslog could support this natively, though.)

In case anyone else wants to set this up, maybe this will save you some effort. I’m also interested in any comments/criticisms about this method, I’d love to hear suggestions for better ways to make this work.

Also, I rolled it all up into a Fedora/EL RPM spec, and I’ll send it on to anyone who’s interested–just ask.

Setup steps:

Install the ‘socat’ utility.
Build RSyslog with the `–enable-omprog` ./configure flag.
Create two separate RSyslog config files, one for the ‘root’ instance (writes to the socket) and a 
second for the ‘unprivileged’ instance (reads from the socket).
Rewrite your RSyslog init script to start two separate daemon instances, one using each config file
(and separate PID files, too).
Create the user ‘rsyslogd’ and the group ‘rsyslogd’.
Set permissions/ownerships as needed to allow the user ‘rsyslogd’ to write to the file ‘/var/log/rsyslog.log’
Create an executable script called '/usr/libexec/rsyslogd/omprog_socat' that contains the lines:

    #!/bin/bash
    /usr/bin/socat -t0 -T0 -lydaemon -d - UNIX-SENDTO:/dev/log
    
The ‘root’ instance config file should contain (modifying the output actions to taste):

    $ModLoad imklog
    $ModLoad omprog
    $Template FwdViaUNIXSocket,"<%pri%>%syslogtag%%msg%"
    $ActionOMProgBinary /usr/libexec/rsyslogd/omprog_socat
    *.* :omprog:;FwdViaUNIXSocket
    
The ‘unprivileged’ instance config file should contain (modifying the output actions to taste):

    $ModLoad imuxsock
    $PrivDropToUser rsyslogd
    $PrivDropToGroup rsyslogd
    *.* /var/log/rsyslog.log

The ‘root’ daemon can only accept input from the kernel message buffer, and nothing else 
(especially not the syslog socket (/dev/log) or any network sockets). The unprivileged user
will handle all of local and network log messages. To merge the kernel logs into the same 
data channel as everything else, here’s what happens:

[During the RSyslog daemons' startup]

A) At startup, the ‘root’ daemon’s ‘imklog’ module starts listening for kernel messages 
(via ‘/prog/kmsg’), and its ‘omprog’ module starts an instance of ‘socat’ (called via the 
‘omprog_socat’ wrapper), establishing a persistent one-way IO connection where ‘omprog’ 
pipes its output to the STDIN of ‘socat’.

(Note that this same ‘socat’ instance remains running throughout the life of the RSyslog daemon, 
handling everything ‘omprog’ outputs. Contrast this, efficiency-wise, against the built-in ‘subshell’ 
module [the '^/path/to/program' action], which runs a separate instance instance of the child program 
for each message.)

B) At startup, the ‘unprivileged’ daemon’s ‘imuxsock’ module opens the system logging socket 
(‘/dev/log’) and starts listening for incoming log messages from other programs.

[During normal operation]1) The kernel buffer produces a message string on ‘/proc/kmsg’.2) 
The ‘root’ RSyslog daemon reads the message from ‘/proc/kmsg’, assigning it the priority number 
of ‘kern.info’ and the string tag ‘kernel’.3) The ‘root’ daemon prepends the priority number and 
tag as a header to the message string, and then passes it to the ‘omprog’ module for output 
(via persistent pipe) to the running ‘socat’ instance.4) The ‘socat’ instance receives the 
header-framed message and sends it to the system logging socket (‘/dev/log’).

5) The ‘unprivileged’ RSyslog daemon reads the message from ‘/dev/log’, assigning it the priority 
and tag given in the message header, plus all of the other properties (timestamp, hostname, etc.) 
a message object should have.

6) The ‘unprivileged’ daemon formats the message and writes it to the output file.

The only real difference I can see in the forwarded messages is that the ‘source’ property is set 
to ‘imuxsock’ instead of ‘imklog’. I don’t think that’s a real problem, though, since the priority 
and tag are still distinct.

