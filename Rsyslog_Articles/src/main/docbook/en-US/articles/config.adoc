### Configure rsyslog ###

We configure rsyslog 
* to recive UDP messages, 
* to filter them depending on the IP of the host, and
* to store them in a file.

### How to configure the module ###

The module has to be configured first. The general line for this configuration is: 

    module (load=”im<type of protocol>”)

So in our example, where we want UDP, it will look like this:

    module (load=”imudp”)

### How to configure the input for rsyslog ###

For the input, you have to give two different information to rsyslog. 

The first information needed is the protocol type of the input; in our example again `UDP`. 
Like in the first line there is an `im` in front of the protocol-type.

The other information is to configure a port for rsyslog, in our example 514. These two 
information items are together in only one line. The line is:

    input (type=”<protocol of input>“ port=”<number of port>“)

This means for the example, the line has to be

    input (type=”imudp” port=”514”)

### How to configure a filter for fromhost-IPs and store them in a file ###

A filter always has, like a normal conditional sentence, an “if…then” part. If you want to configure 
it to do something with all notes from a specific IP, between “if” and “then” will be the property 
“$fromhost-ip ==”-IP, you want to filter-”. After this stays a “then” and after the “then” follows 
an action in brackets, which I will explain later. 

In my example I want only the notes from the host with the IP 172.19.1.135. So the line will be

    If $fromhost-ip == “172.19.1.135” then [

After this we have to tell the computer, what to do if that case is given. In this example we want it
to store these messages in the file “/var/log/network1.log”. This is an action with the type “omfile”. 

To configure the file where to store the messages, the action is “action (type=”omfile” File=”-filename-“). So in this example, it will look like this:

    Action (type=”omfile” file=”/var/log/network1.log”)
    ]
 

All the lines together now are

    Module (load=“imupd“)

    Input (type=”imudp” port=”514”)
    If $fromhost-ip == “172.19.1.135“ then [
        Action (type=”omfile” File=”/var/log/network1.log”)
    ]

All in all it means: The input for rsyslog will listen to syslog via UDP on port 514. If the IP from the Computer, which sends the messages, is 172.19.1.135, then the action in the brackets will get activated for these. In the action the messages will be stored in the file /var/log/network1.log.

 

Rsyslog and rulesets
Rulesets are a bit more complicated. A ruleset is a set of rules, as the name implies. These are bound to an input. This works by adding an option to the input, namely “ruleset=”-rulesetname-“”. For example, if I want to bind a ruleset “rs1” to a input the line will look like this:

Input (type=”imudp” port=”514” ruleset=”rs1”)
But you still have to define, what the ruleset should do. In this guide I will limit myself to explain, how to create a ruleset, which has one action: to store all the messages in a file. In my example I want to store the messages in the file /var/log/network1.log”.

You define a ruleset like the normal configuration. To define it, you first name it with ruleset (name=”-rulesetname-“). After this you write what it does, in my example the action action (type=”omfile” file=”/var/log/network1.log”). This action you write in these curly brackets: {}.

So my full example looks like this

    Module (load=”imudp”)

    Input (type=”imudp” port=”514” ruleset=”rs1”)

    Ruleset (name=”rs1”) {
        Action (type=”omfile” file=”/var/log/network1.log”)
    }

In that second example for configurations you can see, how to store all messages from the input into a file by using a ruleset. A rulesset can consist of multiple rules, but without binding it to the input it is useless. It can be bound to an input multiple times or even other rulesets can be called.