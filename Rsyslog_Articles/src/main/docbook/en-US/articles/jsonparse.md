### Log Message Normalization Module ###

Module Name: mmjsonparse

Description:

This module provides support for parsing structured log messages that follow the CEE/lumberjack spec. The so-called "CEE cookie" is checked and, if present, the JSON-encoded structured message content is parsed. The properties are than available as original message properties.

Sample:

This activates the module and applies normalization to all messages:

    module(load="mmjsonparse")
    action(type="mmjsonparse")
    
The same in legacy format:

    $ModLoad mmjsonparse
    *.* :mmjsonparse:
    
    
    
### how to use mmjsonparse only for select messages ###

Rsyslog's mmjsonparse module permits to parse JSON base data (actually expecting CEE-format). 
This message modification module is implemented via the output plugin interface, which 
provides some nice flexibility in using it.  Most importantly, you can trigger parsing only 
for a select set of messages.

Note that the module checks for the presence of the cee cookie. Only if it is present, json 
parsing will happen. Otherwise, the message is left alone. As the cee cookie was specifically 
designed to signify the presence of JSON data, this is a sufficient check to make sure only 
valid data is processed.

However, you may want to avoid the (small) checking overhead for non-json messages (note, however, 
that the check is *really fast*, so using a filter just to spare it does not gain you too much). 
Another reason for using only a select set might be that you have different types of 
cee-based messages but want to parse (and specifically process just some of them).

With mmjsonparse being implemented via the output module interface, it can be used like a 
regular action. So you could for example do this:

    if ($programname == 'rsyslogd-pstats') then {
          action(type="mmjsonparse")
          action(type="omfwd" target="target.example.net" template="..." ...)
    }

As with any regular action, mmjsonparse will only be called when the filter evaluates to true. 
Note, however, that the modification mmjsonparse makes (most importantly creating the structured data) 
will be kept after the closing if-block. So any other action below that if (in the config file) will 
also be able to see it.

### CEE-enhanced syslog defined ###

CEE-enhanced syslog is an upcoming standard for expressing structured data inside syslog messages. 
It is a cross-platform effort that aims at making log analysis (and log processing in general) 
much more easy both for log producers and consumers. 

The idea was originally born as part of MITRE's CEE effort. It has been adopted by a larger set 
of logging stakeholders in an initiative that was named "project lumberjack". Under this project, 
cee-enhanced syslog, and a framework to make full use of it, is being openly advanced. 
It is hoped (and planned) that the outcome will flow back to the CEE standard.

In a nutshell cee-enhanced syslog is very simple and powerful: inside the syslog message, a 
special cookie ("@cee:") is followed by a JSON representation of the data. The cookie tells 
processors that the format is actually cee-enhanced. 

If you are interested in a more technical coverage, have a look at my 
[cee-enhanced syslog howto presentation[().


### JSON and rsyslog templates ###

Rsyslog already supports JSON parsing and formatting (for all cee properties). 
However, the way formatting currently is done is unsatisfactory to me. Right now, 
we just take the cee properties as they are and format them into JSON format. 
In this mode, we do not have any way to specify which fields to use and we also 
do not have a way to modify the field contents (e.g. pick substrings or do case 
conversions). Exactly these are the use cases rsyslog invented templates for.

One way to handle the situation is to have the user write the JSON code inside the 
template and just inject the data field where desired. This almost works (and I 
know Brian Knox tries to explore that route).  IT just works "almost" as there is 
currently no property replacer option to ensure proper JSON escaping. Adding this 
option is not hard. However, I don't feel this approach is the right route to take: 
making the admin craft the JSON string is error-prone and very user-unfriendly.

So I wonder what would be a good way to specify fields that shall go into a JSON format. 
As a limiting factor, the method should be possible within the limits of the current 
template system - otherwise it will probably take too long to implement it. 
The same question also arises for outputs like MongoDB: how best to specify the fields 
(and structure!) to be passed to the output module?

Of course, both questions are closely related. One approach would be to solve the
JSON encoding and say that to outputs like MongoDB JSON is passed. 
Unfortunately, this has strong performance implications. In a nutshell, it would mean 
formatting the data to JSON, and then re-parsing it inside the plugin. 
This process could be be somewhat simplified by passing the data structure 
(the underlaying tree) itself rather than the JSON encoding.  However, this would still 
mean, that a data structure specific for this use would need to be created. 
That obviously involves a lot of data-copying.
So it would probably be useful to have a capability to specify fields (and replacement 
options) that are just passed down to the module for its use (that would probably limit
the required amount of data copying, at least in common cases). Question again: what 
would be a decent syntax to specify this?

Suggestions are highly welcome. I need to find at least an interim solution urgently, 
as this is an important building block for the MongoDB driver and all work that will 
depend on it. So please provide feedback (note that I may try out a couple of things 
to finally settle on one - so any idea is highly welcome ;)).


