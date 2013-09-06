### Data Flow ###

<img src="http://www.rsyslog.com/doc/dataflow.png" width="680" height="305" alt="Drawing"/>

### Bird's Eye View of Rsyslog Configuration Elements ###

In a rsyslog cnfiguration file, **rulesets** are not the only elements that must
be defined at the top level.  Inputs, templates, modules, and a few directives must 
also be defined at the top level alongside the templates.

There are a few **directives** that need to be defined outside of any statements,
i.e. at the top level.  Examples of such directives are `xxx` 
and 'yyy'.  A directive always starts with a $-sign.

Among the top-level defineable elements, **primary-rulesets** are conceptually
at a higher level than other elements, including **subordinate-rulesets**.  rulesets 
can be defined hierarchically, i.e. one ruleset can call another ruleset
(called subordinate-ruleset as opposed to primary-ruleset)  A primary-ruleset is
a ruleset not reachable via any other ruleset.

Even though an **input** element must also be defined at the top level, i.e. the same
level as rulesets, it is conceptually contained in and belongs to one and only one
particular ruleset.  A ruleset contains, and is pointed to by, one or more inputs using 
the input statement's `ruleset=<ruleset-name>` option.  In other words, there is a 
one-to-many relationship between rulesets and inputs.

There is also a one-to-many relationship between **input-modules** and inputs.  Each 
input must be linked to one and only one input-module using its `type=<input-module-name>`.

Also, there is an implied many-to-many relationship between 
rulesets and **output-modules** via nested action statements.  Each nested action statement of 
a ruleset, via its `type=<output-module-name>`, must specify one and only one output-module
to be utilized for sinking the qualified messages.  On the other hand, the same output-module
can be referred to by more than one ruleset.

Similar to output-modules, **Templates** can also be syntactically bound to the action part 
of the containing rules of a ruleset, and therefore reachable via rulesets only.

It must be noted here that there is also a top-level **main-queue** configuration 
element that explicitly defines a main-queue, unfortunately, only for so-called default 
ruleset.  It is  unfortunate because trying to configure the default ruleset leads to 
an unstructured configuration file with the default ruleset's configuration items splayed 
all over the file.  The default ruleset is the legacy way to assign a ruleset to any 
input-module with a missing `ruleset=<ruleset-name>` option.  The options and rules of 
the default ruleset must be astray outside of a ruleset element and sensitive to the 
order of definitions; and therefore highly error-prone.

In a sense, primary-rulesets are the top of the food-chain reaching all the other elements,
except a few directives that don't have an structured equivalent.


### Ruleset Elements ###

A ruleset is a construct defined with the following syntax:  

    ruleset (<option> ...) { <if or action or stop statement> ... }

An example of a rullset is:

    ruleset (name=”rs1”) {
        if $fromhost-ip == '192.168.152.137' then {
            action(
                type="omfile"
                file="/var/log/remotefile02"
            )
            stop
        }
    }


Using the `ruleset=<rulesetname>` option of the input statement, a rulesets can be 
bound to an input.  For example, to bind a ruleset “rs1” to an input:

    input (
        type=”imudp” 
        port=”514”
        ruleset=”rs1”
    )

So, an a fully defined configuration may looks like:

    module (load=”imtcp”)
    module (load="omfile")

    input (type=”imtcp” port=”514” ruleset=”rs1”)

    ruleset (name=”rs1”) {
        if $fromhost-ip == '192.168.152.137' then {
            action(type="omfile" file="/var/log/remotefile02")
        }
    }

As a result of the above configuration, the rsyslog will listen to syslog via TCP on port 514.
If a received message is sent by a computr with the IP 172.19.1.135, then the messages will be 
stored in the file /var/log/network1.log.

Interestingly enough, the most important component of an explicitly defined ruleset 
(arguably even the most important component of rsyslog), i.e. its main-queue, is not an 
independently defineable element.  Each ruleset has one and only one associated main-queue.
Each main-queue is served by a pool of dedicated worker-threads.  The worker-threads are in charge
of enqueing incomming messages captured by input elements and dequeing and pushing messages
to the filter engines and parsers and then placing them in zero or more action-queues.

The only other kind of worker-threads created by rsyslog is the threads serving the 
action-queues.  In this case, there is no pool per say.  Each action-queue can only
be served by a single worker-thread dedicated to the loaded module associated to the
action element involved. 


## Input Elements ##

Input Elements are the second most important configuration elements of
rsyslog, after rulesets.   





### Multiple Rulesets ###

Starting with version 4.5.0 and 5.1.1, rsyslog supports multiple rulesets within a single configuration.
This is especially useful for routing the reception of remote messages to a set of specific rules.
 
Note that the **input module** must support binding to non-standard rulesets, so the functionality may 
not be available with all inputs.  In this document, I am using imtcp, an input module that supports 
binding to non-standard rulesets since rsyslog started to support them.

### What is a Ruleset? ###

If you have worked with (r)syslog.conf, you know that it is made up of what I call **rules** (others tend 
to call them selectors, a sysklogd term).  Each rule consist of a **filter** and one or more **actions**
to be carried out when the filter evaluates to true.  A filter may be as simple as a traditional syslog 
priority based filter (like "*.*" or "mail.info" or as complex as a script-like expression.
Details on that are covered in the config file documentation. After the filter come action specifiers,
and an action is something that does something to a message, e.g. write it to a file or forward it to
a remote logging server.

A traditional configuration file is made up of one or more of these rules.  When a new message arrives,
its processing starts with the first rule (in order of appearance in rsyslog.conf) and continues for
each rule until either all rules have been processed or a so-called `discard` action happens, in which
case processing stops and the message is thrown away (what also happens after the last rule has been 
processed).

The **multi-ruleset** support now permits to specify more than one such **rule sequence**. You can think
of a traditional config file just as a single default rule set, which is automatically bound to each 
of the inputs.  This is even what actually happens.  When rsyslog.conf is processed, the config file 
parser looks for the directive

    ruleset(name="rulesetname");

Where name is any name the user likes (but must not start with "RSYSLOG_", which is the name space 
reserved for rsyslog use).  If it finds this directive, it begins a new rule set (if the name was not 
yet known) or switches to an already-existing one (if the name was known).  All rules defined between
this `$RuleSet` directive and the next one are appended to the named ruleset.  Note that the reserved
name "RSYSLOG_DefaultRuleset" is used to specify rsyslogd's default ruleset.  You can use that name 
wherever you can use a ruleset name, including when binding an input to it.

Inside a ruleset, messages are processed as described above: they start with the first rule and rules
are processed in the order of appearance of the configuration file until either there are no more 
rules or the discard action is executed. Note that with multiple rulesets no longer all rsyslog.conf
rules are executed but only those that are contained within the specific ruleset.

Inputs must explicitly bind to rulesets. If they don't do, the default ruleset is bound.

This brings up the next question:

### What does "To bind to a Ruleset" mean? ###

This term is used in the same sense as "to bind an IP address to an interface": it means that a 
specific input, or part of an input (like a tcp listener) will use a specific ruleset to "pass its
messages to". So when a new message arrives, it will be processed via the bound ruleset. Rule from 
all other rulesets are irrelevant and will never be processed.

This makes multiple rulesets very handy to process local and remote message via separate means: bind
the respective receivers to different rule sets, and you do not need to separate the messages by any
other method.

Binding to rulesets is input-specific. For imtcp, this is done via the following directive:

    input(
        type="imptcp" 
        port="514" 
        ruleset="rulesetname"
    );

Note that "name" must be the name of a ruleset that is already defined at the time the bind
directive is given. There are many ways to make sure this happens, but I personally think that it is 
best to define all rule sets at the top of rsyslog.conf and define the inputs at the bottom. This kind
of reverses the traditional recommended ordering, but seems to be a really useful and straightforward 
way of doing things.

### Why are rulesets important for different parser configurations? ###

Custom message parsers, used to handle different (and potentially otherwise-invalid) message formats, 
can be bound to rulesets. So multiple rulesets can be a very useful way to handle devices sending 
messages in different malformed formats in a consistent way. Unfortunately, this is not uncommon in 
the syslog world. An in-depth explanation with configuration sample can be found at the $RulesetParser
configuration directive.

### Can I use a different Ruleset as the default? ###

This is possible by using the following directive:

    $DefaultRuleset <name>

Please note, however, that this directive is actually global: that is, it does not modify the
ruleset to which the next input is bound but rather provides a system-wide default rule set for those 
inputs that did not explicitly bind to one. As such, the directive can not be used as a work-around to 
bind inputs to non-default rulesets that do not support ruleset binding.

### Examples ###

#### Split local and remote logging ####

Let's say you have a pretty standard system that logs its local messages to the usual bunch of files 
that are specified in the default rsyslog.conf. As an example, your rsyslog.conf might look like this:

    # ... module loading ...
    # The authpriv file has restricted access.
    authpriv.*  /var/log/secure
    # Log all the mail messages in one place.
    mail.*      /var/log/maillog
    # Log cron stuff
    cron.*      /var/log/cron
    # Everybody gets emergency messages
    *.emerg     *
    ... more ...

Now, you want to add receive messages from a remote system and log these to a special file, but you do
not want to have these messages written to the files specified above. The traditional approach is to 
add a rule in front of all others that filters on the message, processes it and then discards it:

    # ... module loading ...
    # process remote messages
    if $fromhost-ip == '192.168.152.137' then {
        action(
            type="omfile"
            file="/var/log/remotefile02"
        )
  	stop
    }

    # only messages not from 192.0.21 make it past this point

    # The authpriv file has restricted access.
    authpriv.*                            /var/log/secure
    # Log all the mail messages in one place.
    mail.*                                /var/log/maillog
    # Log cron stuff
    cron.*                                /var/log/cron
    # Everybody gets emergency messages
    *.emerg                               *
    ... more ...

Note that "stop" is the discard action!. Also note that we assume that 192.0.2.1 is the sole remote 
sender (to keep it simple).

With multiple rulesets, we can simply define a dedicated ruleset for the remote reception case and 
bind it to the receiver. This may be written as follows:

    # ... module loading ...
    # process remote messages
    # define new ruleset and add rules to it:
    ruleset(name="remote"){
	action(
            type="omfile" 
            file="/var/log/remotefile"
        )
    }
    # only messages not from 192.0.21 make it past this point

    # bind ruleset to tcp listener and activate it:
    input(type="imptcp" port="10514" ruleset="remote")

#### Split local and remote logging for three different ports ####

This example is almost like the first one, but it extends it a little bit. While it is very similar,
I hope it is different enough to provide a useful example why you may want to have more than two 
rulesets.

Again, we would like to use the "regular" log files for local logging, only. But this time we set 
up three syslog/tcp listeners, each one listening to a different port (in this example 10514, 
10515, and 10516). Logs received from these receivers shall go into different files. Also, logs 
received from 10516 (and only from that port!) with "mail.*" priority, shall be written into a 
specif file and not be written to 10516's general log file.

This is the config:

    # ... module loading ...
    # process remote messages

    ruleset(name="remote10514"){
	action(
            type="omfile" 
            file="/var/log/remote10514"
        )
    }

    ruleset(name="remote10515"){
	action(
            type="omfile" 
            file="/var/log/remote10515"
        )
    }

    ruleset(name="test1"){
        if prifilt("mail.*") then {
            /var/log/mail10516
            stop
            # note that the stop-command will prevent this message from 
            # being written to the remote10516 file - as usual...	
        }
        /var/log/remote10516
    }

    # and now define listeners bound to the relevant ruleset
    input(
        type="imptcp" 
        port="10514" 
        ruleset="remote10514"
    )
    input(
        type="imptcp"
        port="10515" 
        ruleset="remote10515"
    )
    input(
        type="imptcp" 
        port="10516"
        ruleset="remote10516"
    )

### Performance ###

#### Fewer Filters ####

No rule processing can be faster than not processing a rule at all. As such, it is useful for a 
high performance system to identify disjunct actions and try to split these off to different rule
sets. In the example section, we had a case where three different tcp listeners need to write to 
three different files. This is a perfect example of where multiple rule sets are easier to use 
and offer more performance. The performance is better simply because there is no need to check 
the reception service - instead messages are automatically pushed to the right rule set and can 
be processed by very simple rules (maybe even with "*.*"-filters, the fastest ones available).

#### Partitioning of Input Data ####

Starting with rsyslog 5.3.4, rulesets permit higher concurrency. They offer the ability to run on
their own "main" queue. What that means is that a own queue is associated with a specific rule set.
That means that inputs bound to that ruleset do no longer need to compete with each other when 
they enqueue a data element into the queue. Instead, enqueue operations can be completed in parallel.

**An example:** let us assume we have three TCP listeners. Without rulesets, each of them needs to 
insert messages into the main message queue. So if each of them wants to submit a newly arrived 
message into the queue at the same time, only one can do so while the others need to wait. 
With multiple rulesets, its own queue can be created for each ruleset. If now each listener is 
bound to its own ruleset, concurrent message submission is possible. On a machine with a 
sufficiently large number of cores, this can result in dramatic performance improvement.

It is highly advised that high-performance systems define a dedicated ruleset, with a dedicated 
queue for each of the inputs.

By default, rulesets do not have their own queue. It must be activated via the 
$RulesetCreateMainQueue directive.

