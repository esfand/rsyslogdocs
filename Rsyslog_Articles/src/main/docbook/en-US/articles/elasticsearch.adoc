### Logging to ElasticSearch ###

This HOWTO should explain the steps of creating a basic setup where host(s) running rsyslog 
is sending logs to host(s) running Elasticsearch.  This would enable you to aggregate logs 
and search for them. Much like Graylog2 does, only not as nice but more flexible and scalable.

I'm running Ubuntu 12.04 x86_64, but I guess on any Linux the steps would be similar.

### Installing Elasticsearch ###
Download it from here: http://www.elasticsearch.org/download/

For Ubuntu there's a nice .deb package which you can simply install. For any other Linux, 
it's as easy as extracting the .tar.gz archive and running bin\elasticsearch

If you have a complex setup, with many logs maybe, you would probably want to build or 
adapt a custom interface. But for now we'll use elasticsearch-head as our GUI. To install
it, simply do:

    git clone git://github.com/mobz/elasticsearch-head.git

Then open index.html in your browser. In the Overview tab you will see your shards and 
replicas, while in the Browser tab you can search for your logs. Trouble is, at this
point we have no shards/replicas and no logs in it.  But that's going to change soon :D

The default settings for Elasticsearch are quite sensible, but if you have a lot of
logs, you might find this tutorial useful:
http://www.elasticsearch.org/tutorials/2012/05/19/elasticsearch-for-logging.html

### Installing rsyslog with omelasticsearch ###
At the time of writing this omelasticsearch is experimental, so you would have to 
download it from the master-elasticsearch branch here:
http://git.adiscon.com/?p=rsyslog.git;a=shortlog;h=refs/heads/master-elasticsearch

Before compiling it, you need libestr:
http://libestr.adiscon.com/download/
and libee:
http://www.libee.org/download/

Here, it was as easy as:
 # tar zxf $PACKAGE_NAME.tar.gz
 # cd $PACKAGE_NAME*
 # ./configure
 # make && make install

When doing the same thing with rsyslog, you would need to add "--enable-elasticsearch" 
when you run the configure script.

### Configuring rsyslog for elasticsearch ###
For a basic setup, you need to add the following lines:

    $ModLoad /usr/local/lib/rsyslog/omelasticsearch.so
    *.*     action(type="omelasticsearch" server="myelasticsearch.mydomain.com")

This would add send all your logs to the specified Elasticsearch server. Your index will be named "system" and your type would be "events".

Now let's suppose you want to add just some specific properties. For that, you would need to define a custom template, that would properly escape the JSON fields for you, and then tell omelasticsearch to use that template.

You can also use templates for defining index names. For example, you might want to have an index per day. This way, for "rotating" logs, you can just remove old indices.

Our config might become something like this:

 $ModLoad /usr/local/lib/rsyslog/omelasticsearch.so
 #
 # the template below will output a JSON like this:
 # {"message":"test","host":"rgheorghe","severity":"6","date":"2012-05-10T10:17:38.045","tag":"test:"}
 $template customSchema,"{\"message\":\"%msg:::json%\",\"host\":\"%HOSTNAME:::json%\",\"severity\":\"%syslogseverity%\",\"date\":\"%timereported:1:19:date-rfc3339%.%timereported:1:3:date-subseconds%\",\"tag\":\"%syslogtag:::json%\"}"
 #
 #the template below outputs something like "2012-05-10" to have our variable index names
 $template srchidx,"%timereported:1:10:date-rfc3339%"
 #
 #now we put everything together
 # "template" is for storing the syslog fields we want
 # dynSearchIndex="on" is for having variable index names
 # searchIndex is for letting rsyslog know where to get these names
 *.*     action(type="omelasticsearch" template="customSchema" searchIndex="srchidx" dynSearchIndex="on" server="myserver")

There are some other nice things you can use:
* searchType="mycustomtype" - to specify a different type than "events". You can have dynSearchType="on" to have it variable, like you can with indices
* serverport="9200" - this is the default setting, but you can specify a different port
* asyncrepl="on" to enable asyncronous replication. That is, Elasticsearch gives an answer imediately after inserting to the main shard(s). It doesn't wait for replicas to be updated as well, which is the default setting
* timeout="1m" - how long to wait for a reply from Elasticsearch. More info here, near the end: http://www.elasticsearch.org/guide/reference/api/index_.html
* basic HTTP authentication. Elasticsearch has no authentication by default, but you can enable it:

Download the http-basic plugin for Elasticsearch from here:
https://github.com/Asquera/elasticsearch-http-basic/downloads

Then, from your Elasticsearch home directory (/usr/share/elasticsearch on Ubuntu):
 # mkdir -p plugins/http-basic
 # cp elasticsearch-http-basic-1.0.3.jar plugins/http-basic/

Then you need to add the following to your config, before restarting Elasticsearch:

 http.basic.enabled: true
 http.basic.user: "myuser"
 http.basic.password: "mypass"

Which is config/elasticsearch.yml if you just extracted the elasticsearch.tar.gz. If you installed it from the .deb package, it's /etc/elasticsearch/elasticsearch.yml

On the rsyslog side, you need to add the following to your "action" line: uid="myuser" pwd="mypass".

Then restart rsyslog and it should work :)

### Using bulk indexing ###
Elasticsearch can index multiple documents at a time (eg: in the same request), which makes this approach faster than indexing one log line at a time. You can make omelasticsearch use this feature by setting bulkmode="on" in your action() line.

The bulk size depends on your queue settings. The default is 16, but, depending on your setup, a value of a few hundred will probably increase the indexing performance.

More infromation about omelasticsearch's bulk indexing here:
http://blog.gerhards.net/2012/06/using-elasticsearch-bulk-mode-with.html

And about queueing in general here:
http://www.rsyslog.com/doc/queues.html