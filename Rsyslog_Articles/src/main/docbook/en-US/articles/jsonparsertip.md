### using mmjsonparse only for select messages ###

Rsyslog's mmjsonparse module permits to parse JSON base data (actually expecting CEE-format). 
This message modification module is implemented via the output plugin interface, which provides 
some nice flexibility in using it.  Most importantly, you can trigger parsing only for a select 
set of messages.

Note that the module checks for the presence of the cee cookie.  Only if it is present, 
json parsing will happen.  Otherwise, the message is left alone.  As the cee cookie was 
specifically designed to signify the presence of JSON data, this is a sufficient check to 
make sure only valid data is processed.

However, you may want to avoid the (small) checking overhead for non-json messages (note, however, 
that the check is *really fast*, so using a filter just to spare it does not gain you too much). 

Another reason for using only a select set might be that you have different types of cee-based 
messages but want to parse (and specifically process just some of them).

With mmjsonparse being implemented via the output module interface, it can be used like a regular action. 
So you could for example do this:

    if ($programname == 'rsyslogd-pstats') then {
        action(type="mmjsonparse")
        action(type="omfwd" target="target.example.net" template="..." ...)
    }

As with any regular action, mmjsonparse will only be called when the filter evaluates to true. Note, 
however, that the modification mmjsonparse makes (most importantly creating the structured data) will 
be kept after the closing if-block. So any other action below that if (in the config file) will also 
be able to see it.
