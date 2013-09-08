## Sending messages with tags larger than 32 characters

Friday, October 21st, 2011    
The relevant syslog RFCs 3164 and 5424 limit the syslog tag to 32 characters max. Messages with larger tag length are malformed and may be discarded by receivers. Anyhow, some folks sometimes need to send tags longer than permitted.

To do so, a new template must be created and used when sending. The simplest way is to start with the standard forwarding template. The standard templates are hardcoded inside rsyslog. Thus they do not show up in your configuration file (but you can obtain them from the source, of course). In 5.8.6, the forwarding template is defined as follows:

template (name="ForwardFormat" type="string" string="<%PRI%>%TIMESTAMP:::date-rfc3339% %HOSTNAME%
%syslogtag:1:32%%msg:::sp-if-no-1st-sp%%msg%")
NOTE: all templates are on one line in rsyslog.conf. They are broken here for readability.

This template is RFC-compliant. Now look at the part in red. It specifies the tag. Note that, 
via the property replacer, it is restricted to 32 characters (from position 1 to position 32 inclusive).
This is what you need to change. To remove the limit … just remove it ;-) This leads to a template like this:

template (name="LongTagForwardFormat" type="string" string="<%PRI%>%TIMESTAMP:::date-rfc3339% %HOSTNAME%
%syslogtag%%msg:::sp-if-no-1st-sp%%msg%")
Note that I have renamed the template in order to avoid conflicts with build-in templates. As it is a custom template, it is not hardcoded, so you need to actually configure it in your rsyslog.conf. Then, you need to use that template if you want to send messages to a remote host. This can be done via the usual way. Let’s assume you use legacy plain TCP syslog. Then the line looks as follows:

action(type="omfwd" 
Target="server.example.net"
Port="10514"
Protocol="tcp"
Template="LongTagForwardFormat"
)
This will bind the forwarding action to the newly defined template. Now tags of any size will be forwarded. Please keep in mind that receivers may have problems with large tags and may truncate them or drop the whole message. So check twice that the receiver handles long tags well.

Rsyslog supports tags to a build-defined maximum. The current (5.8.6) default is 511 characters, but this may be different if you install from a package, use a newer version of rsyslog or use sources obtained from someone else. So double-check.

Tags: config snippet, interoperability, rsyslog, syslog, tag
Posted in Basic Configuration, Guides for rsyslog, The recipies | No Comments »

Using the syslog receiver module
Tuesday, March 29th, 2011
We want to use rsyslog in its general purpose. We want to receive syslog. In rsyslog, we have two possibilities to achieve that.

Things to think about
First of all, we will determine, which way of syslog reception we want to use. We can receive syslog via UDP or TCP. The config statements are each a bit different in both cases.

In most cases, UDP syslog should be fully sufficient and performing well. But, you should be aware, that on large message bursts messages can be dropped. That is not the case with TCP syslog, since the sender and receiver communicate about the arrival of network packets. That makes TCP syslog more suitable for environments where log messages may not be lost or that must ensure PCI compliance (like banks).

Config Statements

    module(load="imudp") # needs to be done just once
    input(type="imudp" port="514")
    
and

module(load="imtcp") # needs to be done just once
input(type="imtcp" port="514")
How it works
The configuration part for the syslog server is pretty simple basically. Though, there are some parameters that can be set for both modules. But this is not necessary in most cases.

In general, first the module needs to be loaded. This is done via the directive module().

module(load="imudp / imtcp")
The module must be loaded, because the directives and functions rely on it. Just by using the right commands, rsyslog will not know where to get the functionality code from.

And next is the command that runs the syslog server itself is called input().

input(type="imudp" port="514")
input(type="imtcp" port="514")
Basically, the command says to run a syslog server on a specific port. Depending on the command, you can easily determine the UDP and the TCP server.

You can of course use both types of syslog server at the same time, too. You just need to load both modules for that and configure the server command to listen to specific ports. Then you can receive both UDP and TCP syslog.

Important

In general, we suggest to use TCP syslog. It is way more reliable than UDP syslog and still pretty fast. The main reason is, that UDP might suffer of message loss. This happens when the syslog server must receive large bursts of messages. If the system buffer for UDP is full, all other messages will be dropped. With TCP, this will not happen. But sometimes it might be good to have a UDP server configured as well. That is, because some devices (like routers) are not able to send TCP syslog by design. In that case, you would need both syslog server types to have everything covered. If you need both syslog server types configured, please make sure they run on proper ports. By default UDP syslog is received on port 514. TCP syslog needs a different port because often the RPC service is using this port as well.

Tags: config snippet, rsyslog, syslog, TCP, UDP
Posted in Basic Configuration | 1 Comment »
