### Structured Logging with Rsyslog and Elasticsearch ###

[Source](http://blog.sematext.com/2013/05/28/structured-logging-with-rsyslog-and-elasticsearch/)

When your applications generate a lot of logs, you’d probably want to make some sense of them by 
searching and/or statistics.  Here’s when structured logging comes in handy, and I would 
like to share some thoughts and configuration examples of how you could use a popular 
syslog  daemon like rsyslog to handle both structured and unstructured logs.  Then I’m going
to look at how you can take those logs, format them in JSON, and index them with 
Elasticsearch – for some fast and easy searching and statistics.  


### On structured logging ###

If we take an unstructured log message, like:

    Joe bought 2 apples

And compare it with a similar one in JSON, like:

    {“name”: “Joe”, “action”: “bought”, “item”: “apples”, “quantity”: 2}

We can immediately spot a couple of advantages and disadvantages of structured logging:

*  If we index these logs, it will be faster and more precise to search for “apples” in the 
“item” field, rather than in the whole document. 

*  At the same time, the structured log will take up more space than the unstructured one.

*  But in most use-cases there will be more applications that would log the same subset of fields. 
So if you want to search for the same user across those applications, it’s nice to be able to pinpoint 
the “name” field everywhere.  

*  And when you add statistics, like who’s the user buying most of our apples, 
that’s when structured logging really becomes useful.

*  Finally, it helps to have a structure when it comes to maintenance. If a new version of the application 
adds a new field, and your log becomes:    
`Joe bought 2 red apples`    
it might break some log-parsing, while structured logs rarely suffer from the same problem.

Enter **CEE** and **Lumberjack**: structured logging within syslog

With syslog, as defined by RFC3164, there is already a structure in the sense that there’s
* a priority value (severity*8 + facility), 
* a header (timestamp and hostname) and 
* a message.

But this usually isn’t the structure we’re looking for.

CEE and Lumberjack are efforts to introduce structured logging to syslog in a backwards-compatible way. 
The process is quite simple: in the message part of the log, one would start with a cookie string `@cee:`, 
followed by an optional space and then a JSON or XML.  From this point on I will talk about JSON, 
since it’s the format that both rsyslog and Elasticsearch prefer.  Here’s a sample CEE-enhanced syslog message:

    @cee: {“foo”: “bar”}

This makes it quite easy to use CEE-enhanced syslog with existing syslog libraries, although there 
are specific libraries like liblumberlog, which make it even easier.  They’ve also defined a list of 
standard fields, and applications should use those fields where they’re applicable – so that you get
the same field names for all applications.  But the schema is free, so you can add custom fields at will.


### CEE-enhanced syslog with rsyslog ##

rsyslog has a module named mmjsonparse for handling CEE-enhanced syslog messages. It checks for the 
“CEE cookie” at the beginning of the message, and then tries to parse the following JSON.  If all is 
well, the fields from that JSON are loaded and you can then use them in templates to extract whatever
information seems important. Fields from your JSON can be accessed like this: $!field-name. 

An example of how they can be used is shown here.

To get started, you need to have at least rsyslog version 6.6.0, and I’d recommend using version 7 
or higher. If you don’t already have that, check out Adiscon’s repositories for RHEL/CentOS and Ubuntu.

Also, mmjsonparse is not enabled by default. If you use the repositories, install the 
rsyslog-mmjsonparse package. If you compile rsyslog from sources, specify `–enable-mmjsonparse` when 
you run the configure script.  In order for that to work you’d probably have to install libjson and 
liblognorm first, depending on your operating system.

For a proof of concept, we can take this config:

    # load needed modules
    module(load=”imuxsock”)        # provides support for local system logging
    module(load=”imklog”)          # provides kernel logging support
    module(load=”mmjsonparse”)     # for parsing CEE-enhanced syslog messages

    # try to parse structured logs
    *.* :mmjsonparse:

    # define a template to print field “foo”
    template(name=”justFoo” type=”list”) {
        property(name=”$!foo”)
        constant(value=”\n”)       # we’ll separate logs with a newline
    }

    # and now let’s write the contents of field “foo” in a file
    *.* action(
            type=”omfile”
            template=”justFoo”
            file=”/var/log/foo”
        )

To see things, better, you can start rsyslog in foreground and in debug mode:

    rsyslogd -dn

And in another terminal, you can send a structured log, then see the value in your file:

    > logger ‘@cee: {“foo”:”bar”}’
    > cat /var/log/foo
    bar

If we send an unstructured log, or an invalid JSON, nothing will be added

    > logger ‘test’
    > logger ‘@cee: test2′
    > cat /var/log/foo
    bar

But you can see in the debug output of rsyslog why:

    mmjsonparse: no JSON cookie: ‘test’
    [...]
    mmjsonparse: toParse: ‘ test2′
    mmjsonparse: Error parsing JSON ‘ test2′: boolean expected


### Indexing logs in Elasticsearch ###

To index our logs in Elasticsearch, we will use an output module of rsyslog called omelasticsearch. 
Like mmjsonparse, it’s not compiled by default, so you will have to add the –enable-elasticsearch 
parameter to the configure script to get it built when you run make. If you use the repositories, 
you can simply install the rsyslog-elasticsearch package.

omelasticsearch expects a valid JSON from your template, to send it via HTTP to Elasticsearch. 
You can select individual fields, like we did in the previous scenario, but you can also select the 
JSON part of the message via the `$!all-json` property. That would produce the message part of the log, 
without the “CEE cookie”.

The configuration below should be good for inserting the syslog message to an Elasticsearch instance 
running on localhost:9200, under the index “system” and type “events“. These are the default options, 
and you can take a look at this tutorial if you need some info on changing them.

    # load needed modules
    module(load=”imuxsock”)        # provides support for local system logging
    module(load=”imklog”)          # provides kernel logging support
    module(load=”mmjsonparse”)     # for parsing CEE-enhanced syslog messages
    module(load=”omelasticsearch”) # for indexing to Elasticsearch

    # try to parse a structured log
    *.* :mmjsonparse:

    # define a template to print all fields of the message
    template(name=”messageToES” type=”list”) {
        property(name=”$!all-json”)
    }

    # write the JSON message to the local ES node
    *.* action(
            type=”omelasticsearch”
            template=”messageToES”
    )

After restarting rsyslog, you can see your JSON will be indexed:

    > logger ‘@cee: {“foo”: “bar”, “foo2″: “bar2″}’
    > curl -XPOST localhost:9200/system/events/_search?q=foo2:bar2 2>/dev/null | sed s/.*_source//
    ” : { “foo”: “bar”, “foo2″: “bar2″ }}]}}

As for unstructured logs, $!all-json will produce a JSON with a field named “msg”, 
having the message as a value:

    > logger test
    > curl -XPOST localhost:9200/system/events/_search?q=test 2>/dev/null | sed s/.*_source//
    ” : { “msg”: “test” }}]}}

It’s “msg” because that’s rsyslog’s property name for the syslog message.


### Including other properties ###

But the message isn’t the only interesting property. I would assume most would want to 
index other information, like the timestamp, severity, or host which generated that message.

To do that, one needs to play with templates and properties. In the future it might be made easier, 
but at the time of this writing (rsyslog 7.2.3), you need to manually craft a valid JSON to pass it 
to omelasticsearch.  For example, if we want to add the timestamp and the syslogtag, a working 
template might look like this:

    template(name=”customTemplate” type=”list”) {
        #- open the curly brackets,
        #- add the timestamp field surrounded with quotes
        #- add the colon which separates field from value
        #- open the quotes for the timestamp itself
        constant(value=”{\”timestamp\”:\”")

        #- add the timestamp from the log,
        # format it in RFC-3339, so that ES detects it by default
        property(name=”timereported” dateFormat=”rfc3339″)

        #- close the quotes for timestamp,
        #- add a comma, then the syslogtag field in the same manner
        constant(value=”\”,\”syslogtag\”:\”")

        #- now the syslogtag field itself
        # and format=”json” will ensure special characters
        # are escaped so they won’t break our JSON
        property(name=”syslogtag” format=”json”)

        #- close the quotes for syslogtag
        #- add a comma
        #- then add our JSON-formatted syslog message,
        # but start from the 2nd position to omit the left
        # curly bracket
        constant(value=”\”,”)

        property(name=”$!all-json” position.from=”2″)
    }


### Summary ###

If you’re interested in searching or analyzing lots of logs, structured logging might help. 
And you can do it with the existing syslog libraries, via CEE-enhanced syslog.  

If you use a newer version of rsyslog, you can parse these logs with mmjsonparse and index
them in Elasticsearch with omelasticsearch.  

If you want to use Logsene, it will consume your structured logs as described in this post.

