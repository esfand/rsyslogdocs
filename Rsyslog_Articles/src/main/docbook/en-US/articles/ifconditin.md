## Filter Conditions ##
[Source](http://www.rsyslog.com/doc/rsyslog_conf_filter.html)

Rsyslog offers three different types "filter conditions":
* RainerScript</a>-based filters
* "traditional" severity and facility based selectors
* property-based filters


### RainerScript-Based Filters ###

RainerScript based filters are the prime means of creating complex rsyslog configuration.
The permit filtering on arbitrary complex expressions, which can include boolean,
arithmetic and string operations. They also support full nesting of filters, just
as you know from other scripting environments.    
Scripts based filters are indicated by the keyword "if", as usual.
They have this format:
    
    if expr then block else block

"If" and "then" are fixed keywords that mus be present. "expr" is a (potentially quite complex) expression. 
So the <a href="expression.html">expression documentation</a> for details.
The keyword "else" and its associated block is optional. Note that a block can contain either
a single action (chain), or an arbitrary complex script enclosed in curly braces, e.g.:

    if $programname == 'prog1' then {
        action(type="omfile" file="/var/log/prog1.log")
        if $msg contains 'test' then
            action(type="omfile" file="/var/log/prog1test.log")
        else
            action(type="omfile" file="/var/log/prog1notest.log")
    }

Other types of filtes can also be combined with the pure RainerScript ones. This makes
it particularly easy to migrate from early config files to RainerScript. Also, the traditional
syslog PRI-based filters are a good and easy to use addition. While they are legacy, we still
recommend there use where they are up to the job. We do NOT, however, recommend property-based
filters any longer. As an example, the following is perfectly valid:

    if $fromhost == 'host1' then {
        mail.* action(type="omfile" file="/var/log/host1/mail.log")
        *.err /var/log/host1/errlog # this is also still valid
        # 
        # more "old-style rules" ...
        #
    } else {
        mail.* action(type="omfile" file="/var/log/mail.log")
        *.err /var/log/errlog
        # 
        # more "old-style rules" ...
        #
    }

Right now, you need to specify numerical values if you would like to check for facilities 
and severity. These can be found  in [RFC 3164](http://www.ietf.org/rfc/rfc3164.txt)
If you don't like that, you can of course also use the textual property - just be sure to use the right one.  
As expression support is enhanced, this will change. For example, if you would like to filter on message
that have facility local0, start with "DEVNAME" and have either
"error1" or "error0" in their message content, you could use the following filter:

    if $syslogfacility-text == 'local0' and 
       $msg startswith 'DEVNAME'        and 
       ($msg contains 'error1' or $msg contains 'error0')
        then /var/log/somelog<br>

Please note that the above **must all be on one line**! And if you would like to store all
messages except those that contain "error1" or "error0", you just need
to add a "not":

    if  $syslogfacility-text == 'local0' and 
        $msg startswith 'DEVNAME' and not 
        not ($msg contains 'error1' or $msg contains 'error0') 
    then 
        /var/log/somelog<br>

If you would like to do case-insensitive comparisons, use
"contains_i" instead of "contains" and "startswith_i" instead of "startswith".

Regular expressions are supported via functions (see function list).

### Selectors ###

**Selectors are the traditional way of filtering syslog messages.** 
They have been kept in rsyslog with their original syntax, because it is well-known, highly 
effective and also needed for compatibility with stock syslogd configuration files. 
If you just need to filter based on priority and facility, you should do this with
selector lines. They are <b>not</b> second-class citizens
in rsyslog and offer the best performance for this job.

The selector field itself again consists of two parts, a
facility and a priority, separated by a period (".''). 
Both parts are
case insensitive and can also be specified as decimal numbers, but
don't do that, you have been warned. Both facilities and priorities are
described in syslog(3). The names mentioned below correspond to the
similar LOG_-values in /usr/include/syslog.h.

The facility is one of the following keywords:  auth, authpriv, cron, daemon, kern, lpr, 
mail, mark, news, security (same as auth), syslog, user, uucp and local0 through local7.

The keyword security should not
be used anymore and mark is only for internal use and therefore should
not be used in applications. Anyway, you may want to specify and
redirect these messages here. The facility specifies the subsystem that
produced the message, i.e. all mail programs log with the mail facility
(LOG_MAIL) if they log using syslog.

The priority is one of the following keywords, in ascending order:
debug, info, notice, warning, warn (same as warning), err, error (same as err), 
crit, alert, emerg, panic (same as emerg). 
The keywords error, warn and panic are deprecated and should not be used anymore. 
The priority defines the severity of the message.

The behavior of the original BSD syslogd is that all messages of the
specified priority and higher are logged according to the given action.
Rsyslogd behaves the same, but has some extensions.

In addition to the above mentioned names the rsyslogd(8) understands
the following extensions: An asterisk ("*'') stands for all facilities
or all priorities, depending on where it is used (before or after the
period). The keyword none stands for no priority of the given facility.

You can specify multiple facilities with the same priority pattern in
one statement using the comma (",'') operator. You may specify as much
facilities as you want. Remember that only the facility part from such
a statement is taken, a priority part would be skipped.

Multiple selectors may be specified for a single action using
the semicolon (";'') separator. Remember that each selector in the
selector field is capable to overwrite the preceding ones. Using this
behavior you can exclude some priorities from the pattern.

Rsyslogd has a syntax extension to the original BSD source,
that makes its use more intuitively. You may precede every priority
with an equals sign ("='') to specify only this single priority and
not any of the above. You may also (both is valid, too) precede the
priority with an exclamation mark ("!'') to ignore all that
priorities, either exact this one or this and any higher priority. If
you use both extensions than the exclamation mark must occur before the
equals sign, just use it intuitively.

### Property-Based Filters ###

Property-based filters are unique to rsyslogd. They allow to
filter on any property, like HOSTNAME, syslogtag and msg. A list of all
currently-supported properties can be found in the <a href="property_replacer.html">property replacer documentation</a>
(but keep in mind that only the properties, not the replacer is
supported). With this filter, each properties can be checked against a
specified value, using a specified compare operation.

A property-based filter must start with a colon in column 0.
This tells rsyslogd that it is the new filter type. The colon must be
followed by the property name, a comma, the name of the compare
operation to carry out, another comma and then the value to compare
against. This value must be quoted. There can be spaces and tabs
between the commas. Property names and compare operations are
case-sensitive, so "msg" works, while "MSG" is an invalid property
name. In brief, the syntax is as follows:

    :property, [!]compare-operation, "value"

The following **compare-operations** are currently supported:

* **contains**    
Checks if the string provided in value is contained in
the property. There must be an exact match, wildcards are not supported.


* **isempty**    
Checks if the property is empty. The value is discarded. This is
especially useful when working with normalized data, where some fields
may be populated based on normalization result.
Available since 6.6.2.


* **isequal**    
Compares the "value" string provided and the property contents.
These two values must be exactly equal to match. 
The difference to contains is that contains searches for the value anywhere
inside the property value, whereas all characters must be identical for isequal. 
As such, isequal is most useful for fields like syslogtag or
FROMHOST, where you probably know the exact contents.


* **startswith**    
Checks if the value is found exactly at the beginning
of the property value. For example, if you search for "val" with

    :msg, startswith, "val"

    it will be a match if msg contains "values are in this
message" but it won't match if the msg contains "There are values in
this message" (in the later case, contains would match). Please note
that "startswith" is by far faster than regular expressions. So
it makes very much sense (performance-wise) to use "startswith".

    Note: when processing syslog messages, please note that $msg usually
starts with a space. The reason for this is RFC3164. Please read the
<a href="http://www.rsyslog.com/log-normalization-and-the-leading-space/">detail
description</a> of what that means to you. In short, you need to make sure
that you include the first space if you use "startswith", otherwise you will
not get matches.



* **regex**    
Compares the property against the provided POSIX BRE regular expression.


* **ereregex**    
Compares the property against the provided POSIX ERE regular expression.


You can use the bang-character (!) immediately in front of a
compare-operation, the outcome of this operation is negated. For
example, if msg contains "This is an informative message", the
following sample would not match:

    :msg, contains, "error"

but this one matches:

    :msg, !contains, "error"    

Using negation can be useful if you would like to do some
generic processing but exclude some specific events. You can use the
discard action in conjunction with that. A sample would be:

    *.* /var/log/allmsgs-including-informational.log
    :msg, contains, "informational" ~
    *.* /var/log/allmsgs-but-informational.log

Do not overlook the red tilde in line 2! In this sample, all
messages are written to the file allmsgs-including-informational.log.
Then, all messages containing the string "informational" are discarded.
That means the config file lines below the "discard line" (number 2 in
our sample) will not be applied to this message. Then, all remaining
lines will also be written to the file allmsgs-but-informational.log.

**Value** is a quoted string. It supports some escape sequences:</p>

\" - the quote character (e.g. "String with \"Quotes\"")    
\\ - the backslash character (e.g. "C:\\tmp")

Escape sequences always start with a backslash. Additional
escape sequences might be added in the future. Backslash characters <b>must</b>
be escaped. Any other sequence then those outlined above is invalid and
may lead to unpredictable results.

<p>Probably, "msg" is the most prominent use case of property
based filters. It is the actual message text. If you would like to
filter based on some message content (e.g. the presence of a specific
code), this can be done easily by:</p>

    :msg, contains, "ID-4711"

This filter will match when the message contains the string
"ID-4711". Please note that the comparison is case-sensitive, so it
would not match if "id-4711" would be contained in the message.

    :msg, regex, "fatal .* error"

This filter uses a POSIX regular expression. It matches when the
string contains the words "fatal" and "error" with anything in between
(e.g. "fatal net error" and "fatal lib error" but not "fatal error" as
two spaces are required by the regular expression!).

Getting property-based filters right can sometimes be challenging. 
In order to help you do it with as minimal effort as
possible, rsyslogd spits out debug information for all property-based
filters during their evaluation. To enable this, run rsyslogd in
foreground and specify the "-d" option.

Boolean operations inside property based filters (like
'message contains "ID17" or message contains "ID18"') are currently not
supported (except for "not" as outlined above). Please note that while
it is possible to query facility and severity via property-based
filters, it is far more advisable to use classic selectors (see above)
for those cases.
