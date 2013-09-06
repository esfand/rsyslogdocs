### Queues and omelasticsearch ###

The following text gives some explained configuration snippets about how you can use 
omelasticsearch to get your logs to Elasticsearch, and how to use queues in the process 
for enhancing performance and reliability.

First of all, some references about queues:
* general info: http://www.rsyslog.com/doc/queues.html
* v6-specific: http://www.rsyslog.com/doc/node32.html

But in short, queues are good for chaching logs when the recipient is not as fast as the 
sender. Or when it's down. Or, when you simply want to process them faster, because 
rsyslog can make batches of messages from the queue and process them in bulk.

You get a "MainMsgQueue", where rsyslog puts the received messages in order to be processed. 
And for each action, you can define a queue.

Queues can be in memory, on disk, or a smart combination of both, named Disk-Assisted. 
I don't like pure disk queues because they're slow, so I'll concentrate on the other two.

Elasticsearch is a distributed search engine build on top of Lucene. It's nice to collect 
your logs there in order to search them. At the time of this writing, omelasticsearch 
(rsyslog plugin for Elasticsearch) can be found in the latest beta - v6.3.12. A HOWTO on how 
to get it working can be found here: 
http://wiki.rsyslog.com/index.php/HOWTO:_rsyslog_%2B_elasticsearch


### Main message queue ###

With 6.3.12, you need to specify Main message queue options in the old, pre-v6 format. 
Here's an example of a disk-assisted queue here:

    # location for storing queue files on disk
    $WorkDirectory /var/lib/elasticsearch/rsyslog_queues
 
    main_queue {

        # file name template, also enables disk mode for the memory queue
        queue.filename="incoming_queue"
 
        # allocate memory dynamically for the queue. Better for handling spikes
        queue.type="LinkedList"
 
        # when to start discarding messages
        queue.discardMark=2000000
 
        # limit of messages in the memory queue. When this is reached, it starts to write to disk
        queue.highWaterMark=1000000
 
        # memory queue size at which it stops writing to the disk
        queue.lowWaterMark=800000
 
        # maximum disk space used for the disk part of the queue
        queue.maxfilesize="5g"
 
        # how many messages (messages, not bytes!) to hold in memory
        $queue.size=8000000
 
        # don't throttle receiving messages when the queue gets full
        queue.timeoutenqueue 0
 
        # save the queue contents when stopping rsyslog
        queue.saveonshutdown=on
    }

You can also use an in-memory queue here. The advantages are that it will be consistently good 
in terms of performance, and it will fail fast - things that you normally want in a distributed,
scalable system, where reliability isn't very important. The downside is that you can keep a 
limited number of messages in that queue, less than you normally can with a disk-assisted queue. 
Example config snippet:

    main_queue {

        # allocate memory dynamically for the queue. Better for handling spikes
        queue.type="LinkedList"
 
        # how many messages (messages, not bytes!) to hold in memory
        $queue.size=8000000
 
        # don't throttle receiving messages when the queue gets full
        queue.timeoutenqueue=0
    }


### Action queue ###

For each action you can define a queue. And you can do that with the new format. 
Here's a config snippet that you might find useful for using disk assisted queues 
with omelasticsearch:

    template(name="miniSchema"  type="list" option.json="on") {

        #- open the curly brackets,
        constant(value=”{")

        #- add the timestamp field surrounded with quotes
        #- add the colon which separates field from value
        #- open the quotes for the timestamp itself
        constant(value="\”timestamp\”:\”")
        #- add the timestamp from the log,
        # format it in RFC-3339, so that ES detects it by default
        property(name=”timereported” dateFormat=”rfc3339″ outname="timestamp")

        #- close the quotes for timestamp,
        #- add a comma, then the syslogtag field in the same manner
        constant(value="\",\”syslogtag\”:\”")




        constant(value="{\"message\":\"")
        property(name=%msg:::json%\",
        \"host\":\"%HOSTNAME:::json%\",
        \"severity\":\"
        %syslogseverity%
        \",\"date\":\"
        %timereported:1:19:date-rfc3339%
        .
        %timereported:1:3:date-subseconds%
        \",\"tag\":\"
        %syslogtag:::json%
        \"}"
        constant(value=”\n”) #we’ll separate logs with a newline
    }

    $template srchidx,"%timereported:1:10:date-rfc3339%"

    *.* action(
            type="omelasticsearch" 
                template="miniSchema"
                searchIndex="srchidx"
                dynSearchIndex="on" 
                searchType="logs"
                server="localhost"
                serverport="9200"
                bulkmode="on" 
                queue.dequeuebatchsize="200"
                queue.type="linkedlist"
                queue.timeoutenqueue="0"
                queue.filename="dbq"
                queue.highwatermark="500000"
                queue.lowwatermark="400000"
                queue.discardmark="5000000"
                queue.timeoutenqueue="0"
                queue.maxdiskspace="5g"
                queue.size="2000000"
                queue.saveonshutdown="on"
                action.resumeretrycount="-1")

Now let's take things one line at a time:

    $template miniSchema,"{
        \"message\":\"
        %msg:::json%
        \",\"host\":\"
        %HOSTNAME:::json%
        \",\"severity\":\"
        %syslogseverity%
        \",\"date\":\"
        %timereported:1:19:date-rfc3339%
        .
        %timereported:1:3:date-subseconds%
        \",\"tag\":\"
        %syslogtag:::json%
        \"}"

This is to describe how your log line will end up as a document in Elasticsearch. 
For example, at the beginning, we have \"message\":\"%msg:::json%\". This means that
"message" will be the name of the field, and the value will be whatever the syslog %msg% 
property contains. Putting %msg:::json% there is to tell rsyslog to make that value 
a proper JSON field, by escaping quotes for example. Some fields don't need such escaping,
so you can save your CPU some time by omitting that, like it's done here
with \"severity\":\"%syslogseverity%\".

Note about quotes: in general, you need to have quotes to describe the template, like this: 

    $template TemplateName,"template contents go here"

JSON fields and values are delimited by double-quotes, so you need to escape them with a 
backslash. If you need to check out more about rsyslog templates, here are two useful links:
* general info about templates: http://www.rsyslog.com/doc/rsyslog_conf_templates.html
* what sort of variables you can put there: http://www.rsyslog.com/doc/property_replacer.html

Finally, you can use the full power of templates when describind your JSON. For example, you 
might have noticed the hack with this one: 

    \"date\":\"%timereported:1:19:date-rfc3339%.%timereported:1:3:date-subseconds%\"

It's because I always want my timestamp with miliseconds, and I want 000 for those who don't
report sub-second timestamps. There might be a better way to do this (please fill in here if
you got a nicer approach), but it works for me.

    $template srchidx,"%timereported:1:10:date-rfc3339%"

This is a template that we'll use to make one index of logs per day. You can choose to keep
all your logs in one index, and remove old ones by 
using TTL: http://www.elasticsearch.org/guide/reference/mapping/ttl-field.html

But in general it's recommended to roll your indices based on time. Depending on how long 
you want to keep your logs, you might want to choose another interval (eg: one per month)
to avoid having too many indices. But other than that, you will be able to change your number
of shards and replicas, and other index setting pretty easily, just because a new index will
be created every now and then. Most importantly, you can backup, optimize, remove and restore
"old" indices quite easily. Here are a couple of scripts that do all 
that (except for deleting): https://gist.github.com/3180985

    *.*     action(type="omelasticsearch"
                template="miniSchema"

Here we make all logs go through omelasticsearch. We use the template defined earlier to 
make the JSON document. You don't *need* to use such a template, you can rely on the default 
one which looks like this: 

    $template JSONDefault, "{
                             \"message\":\"
                             %msg:::json%
                             \",\"fromhost\":\"
                             %HOSTNAME:::json%
                             \",\"facility\":\"
                             %syslogfacility-text%
                             \",\"priority\":\"
                             %syslogpriority-text%\",\"timereported\":\"
                             %timereported:::date-rfc3339%\",\"timegenerated\":\"
                             %timegenerated:::date-rfc3339%\"}"

                searchIndex="srchidx"
                dynSearchIndex="on" 
                searchType="logs"

Here we use the "srchidx" template defined earlier to make index names like "2012-07-25". In order to tell rsyslog to use the template with that name instead of just the given string, you need to specify dynSearchIndex="on". By contrast, we leave it to the default "off" for dynSearchType, because here we want to name our type simply "logs".

                server="localhost"
                serverport="9200"

Here we define the Elasticsearch node where we send the logs (host and port). In case the node goes down, you can specify a failover node, much like you do with any other action, as described here: http://wiki.rsyslog.com/index.php/FailoverSyslogServer

But stay tuned! In future versions this might be done automatically, since a list of Elasticsearch nodes can be retrieved at startup, if multicasting works in your network. Take a look here for details: http://www.gossamer-threads.com/lists/rsyslog/users/7161

                bulkmode="on" 
                queue.dequeuebatchsize="200"

By default, rsyslog inserts your logs to Elasticsearch one by one. You can make things much faster by using a queue and inserting them in bulk. queue.dequeuebatchsize is the maximum number of elements in the bulk. "Maximum" means that, if there isn't much load, rsyslog will not wait for the queue to get to that size in order to do a bulk insert. Which is good. More info about bulk inserts in Elasticsearch can be found here: http://www.elasticsearch.org/guide/reference/api/bulk.html

                queue.type="linkedlist"
                queue.timeoutenqueue="0"
                queue.filename="dbq"
                queue.highwatermark="500000"
                queue.lowwatermark="400000"
                queue.discardmark="5000000"
                queue.timeoutenqueue="0"
                queue.maxdiskspace="5g"
                queue.size="2000000"
                queue.saveonshutdown="on"

These are the queue settings we have looked at in the Main Message Queue section, only in the new, v6 configuration format. Each action can have it's own queue settings.

                action.resumeretrycount="-1")

This enables rsyslog to retry indefinitely if your Elasticsearch node is down, which will build up a queue during that time. When Elasticsearch goes back up, it will start inserting from the queue.

You don't have to worry what happens if a message gets malformed in a way (eg: not a proper JSON). Elasticsearch gives an error in this case, and the message is discarded from the queue.

### Some messages are already JSON ###

Let's take a more complex scenario: some applications can put JSONs in some of their syslog messages. It would be awesome (eg: faster and more precise searches, better statistics using facets) to use that JSON and insert it in Elasticsearch as it is.

Now let's assume that the logging application will identify those JSON messages in a way. For example, tag them by making JSON messages look like this: 'TAG {"name": "Radu", "class": "paladin", "level": 9001}'

Let's also assume that we want to take the JSON from there, and add the timestamp for when the message was reported, and insert the resulting JSON in a given Elasticsearch index. We might also want to put logs in separate types based on hostnames. For example, logs from webserver01, webserver02, etc will go to webserver_logs and those from db01, db02, etc will go to db_logs.

We should be able to achieve all this with the following:

    $template getType,"%hostname:R,ERE,0,DFLT:[A-Za-z-]+--end%_logs"

    $template myJSONs,"{\"date\":\"%timereported:1:19:date-rfc3339%\", %msg:7:$::"

    if $msg startswith ' TAG' then action(type="omelasticsearch"
                                    template="myJSONs"
                                    searchIndex="testindex"
                                    searchType="getType"
                                    dynSearchType="on"
                                    server="localhost"
                                    serverport="9200"
                                    bulkmode="on"
                                    queue.dequeuebatchsize="100"
                                    queue.type="linkedlist"
                                    queue.size="200000"
                                    queue.timeoutenqueue="0"
                                    action.resumeretrycount="-1")

Let's take them one by one:

    $template getType,"%hostname:R,ERE,0,DFLT:[A-Za-z-]+--end%_logs"

This will take hostnames like myserver01 or my-server23 and generate strings like 
myserver_logs or my-server_logs. Which is what we want for our Elasticsearch types in this scenario.

    $template myJSONs,"{\"date\":\"%timereported:1:19:date-rfc3339%\", %msg:7:$::"

This will take a syslog message that goes like 

    'TAG {"name": "Radu", "class": "paladin", "level": 9001}'

but only from the 7th character until the end `(%msg:7:$::)`. This will be added to the first part, which is our reported timestamp. The end result will be the the proper, date-containing JSON that we really want: '{"date": "2012-07-26T13:59:36", "name": "Radu", "class": "paladin", "level": 9001}'.

    if $msg startswith ' TAG' then 
        action(
            type="omelasticsearch"
            template="myJSONs"
            searchIndex="testindex"

This will take all messages that start with "TAG", use the defined template to make them a proper JSON and insert them in the specified index. Note the leading space in the condition: ' TAG'. It's there because in normal conditions the %msg% variable gets a leading space. Some details about why that happens can be found here: http://www.rsyslog.com/log-normalization-and-the-leading-space/

                                    searchType="getType"
                                    dynSearchType="on"
                                    server="localhost"
                                    serverport="9200"

Here we define our Elasticsearch server and port, and also the dynamic type (eg: generic-server-name_logs) that we defined earlier.

                                    bulkmode="on"
                                    queue.dequeuebatchsize="100"

You probably want bulk indexing here as well, so we configure this again.

                                    queue.type="linkedlist"
                                    queue.size="200000"
                                    queue.timeoutenqueue="0"
                                    action.resumeretrycount="-1")

Here we configure an in-memory queue, of type "linkedlist" that will hold up to 200K messages. 
If the queue gets full it will not throttle, and it will retry indefinitely if the Elasticsearch
server is unreachable.



### outname Support for omelasticsearch ###

**Question**    
I'm trying out rsyslog 7.2.2 with omelasticsearch, and I'm struggling a bit
with the new template format. What I got working is something like this:

    template(name="myTemplate" type="list" option.json="on") {
        constant(value="{")
        constant(value="\"myMessage\":\"")       property(name="msg")
        constant(value="\",\"myTimestamp\":\"")  property(name="timereported" dateFormat="rfc3339")
        constant(value="\"}")
    }


But from reading the documentation I understand that I could use the
"outname" option to make it look less ugly. Something like this:

    template(name="myTemplate" type="list" option.json="on") {
        property(name="msg" outname="message")
        property(name="timereported" outname="timestamp" dateFormat="rfc3339")
    }

Which doesn't work, it seems to work as if "outname" just isn't there.

Are there any nicer ways to define my template? How does "outname" work? Is
it not supported for omelasticsearch?

**Answer**    
just a quick note: I need to look some more in-depth, will do asap.

I think outname does not affect string generation. 
It's primarily for plugins that are structure-aware, and omelasticsearch has not been upgraded 
to that new interface.  But I am not 100% sure (too much changed in the past weeks ;)).

