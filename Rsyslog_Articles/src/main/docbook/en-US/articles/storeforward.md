Storing and forwarding remote messages
--------------------------------------
Friday, April 1st, 2011

In this scenario, we want to store remote sent messages into a specific local file and 
forward the received messages to another syslog server. Local messages should still be 
locally stored.

### Things to think about ###

How should this work out? Basically, we need a syslog listener for TCP and one for UDP, the local logging service and two rulesets, one for the local logging and one for the remote logging.

TCP recpetion is not a build-in capability. You need to load the imtcp plugin in order to enable it. This needs to be done only once in rsyslog.conf. Do it right at the top.

Note that the server port address specified in $InputTCPServerRun must match the port address that the clients send messages to.

### Config Statements ###

~~~
# Modules
$ModLoad imtcp
$ModLoad imudp
$ModLoad imuxsock
$ModLoad imklog

# Templates
# log every host in its own directory
$template RemoteHost,"/var/syslog/hosts/%HOSTNAME%/%$YEAR%/%$MONTH%/%$DAY%/syslog.log"

### Rulesets
# Local Logging
$RuleSet local
kern.*                                                 /var/log/messages
*.info;mail.none;authpriv.none;cron.none                /var/log/messages
authpriv.*                                              /var/log/secure
mail.*                                                  -/var/log/maillog
cron.*                                                  /var/log/cron
*.emerg                                                 *
uucp,news.crit                                          /var/log/spooler
local7.*                                                /var/log/boot.log
# use the local RuleSet as default if not specified otherwise
$DefaultRuleset local

# Remote Logging
$RuleSet remote
*.* ?RemoteHost
# Send messages we receive to Gremlin
*.* @@W.X.Y.Z:514

### Listeners
# bind ruleset to tcp listener
$InputTCPServerBindRuleset remote
# and activate it:
$InputTCPServerRun 10514

$InputUDPServerBindRuleset remote
$UDPServerRun 514
~~~

### How it works ###

The configuration basically works in 4 parts. First, we load all the modules (imtcp, imudp, imuxsock, imklog). Then we specify the templates for creating files. The we create the rulesets which we can use for the different receivers. And last we set the listeners.

The rulesets are somewhat interesting to look at. The ruleset “local” will be set as the default ruleset. That means, that it will be used by any listener if it is not specified otherwise. Further, this ruleset uses the default log paths vor various facilities and severities.

The ruleset “remote” on the other hand takes care of the local logging and forwarding of all log messages that are received either via UDP or TCP. First, all the messages will be stored in a local file. The filename will be generated with the help of the template at the beginning of our configuration (in our example a rather complex folder structure will be used). After logging into the file, all the messages will be forwarded to another syslog server via TCP.

In the last part of the configuration we set the syslog listeners. We first bind the listener to the ruleset “remote”, then we give it the directive to run the listener with the port to use. In our case we use 10514 for TCP and 514 for UDP.

### Important ###
There are some tricks in this configuration. Since we are actively using the rulesets, we must specify those rulesets before being able to bind them to a listener. That means, the order in the configuration is somewhat different than usual. Usually we would put the listener commands on top of the configuration right after the modules. Now we need to specify the rulesets first, then set the listeners (including the bind command). This is due to the current configuration design of rsyslog. To bind a listener to a ruleset, the ruleset object must at least be present before the listener is created. And that is why we need this kind of order for our configuration.
