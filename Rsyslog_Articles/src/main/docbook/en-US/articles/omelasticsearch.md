# Elasticsearch Output Module #

This module provides native support for logging to <a href="http://www.elasticsearch.org/">Elasticsearch</a>.</p>

**Action Parameters:**

* **server**    
Host name or IP address of the Elasticsearch server. Defaults to "localhost"</li>
* **serverport**    
HTTP port to connect to Elasticsearch. Defaults to 9200</li>
* **searchIndex**    
<a href="http://www.elasticsearch.org/guide/appendix/glossary.html#index">Elasticsearch index</a> to send your logs to. Defaults to "system"
* **dynSearchIndex** on off    
Whether the string provided for <strong>searchIndex</strong> should be taken as a <a href="http://www.rsyslog.com/doc/rsyslog_conf_templates.html">template</a>. Defaults to "off", which means the index name will be taken literally. Otherwise, it will look for a template with that name, and the resulting string will be the index name. For example, let's assume you define a template named "date-days" containing "%timereported:1:10:date-rfc3339%". Then, with dynSearchIndex="on", if you say searchIndex="date-days", each log will be sent to and index named after the first 10 characters of the timestamp, like "2013-03-22".

* **searchType**
<a href="http://www.elasticsearch.org/guide/appendix/glossary.html#type">Elasticsearch type</a> to send your index to. 
Defaults to "events"

* **dynSearchType** <on|**off**>
Like <strong>dynSearchIndex</strong>, it allows you to specify a <a href="http://www.rsyslog.com/doc/rsyslog_conf_templates.html">template</a> for <strong>searchType</strong>, instead of a static string.

* **asyncrepl** <on|**off>
By default, an indexing operation returns after 
all <a href="http://www.elasticsearch.org/guide/appendix/glossary.html#replica_shard">replica shards</a> 
have indexed the document. With asyncrepl="on" it will return after it was indexed on 
the <a href="http://www.elasticsearch.org/guide/appendix/glossary.html#primary_shard">primary shard</a> 
only - thus trading some consistency for speed.
* **timeout**
How long Elasticsearch will wait for a primary shard to be available for indexing your log before sending 
back an error. Defaults to "1m".
* **template**
This is the JSON document that will be indexed in Elasticsearch. 
The resulting string needs to be a valid JSON, otherwise Elasticsearch will return an error. Defaults to:

		<pre>$template JSONDefault, "{\"message\":\"%msg:::json%\",\"fromhost\":\"%HOSTNAME:::json%\",\"facility\":\"%syslogfacility-text%\",\"priority\":\"%syslogpriority-text%\",\"timereported\":\"%timereported:::date-rfc3339%\",\"timegenerated\":\"%timegenerated:::date-rfc3339%\"}"
</pre>

<p>Which will produce this sort of documents (pretty-printed here for readability):</p>


<pre>{
&nbsp;&nbsp;&nbsp; "message": " this is a test message",
&nbsp;&nbsp;&nbsp; "fromhost": "test-host",
&nbsp;&nbsp;&nbsp; "facility": "user",
&nbsp;&nbsp;&nbsp; "priority": "info",
&nbsp;&nbsp;&nbsp; "timereported": "2013-03-12T18:05:01.344864+02:00",
&nbsp;&nbsp;&nbsp; "timegenerated": "2013-03-12T18:05:01.344864+02:00"
}</pre>

* **bulkmode** <on|**off**>
The default "off" setting means logs are shipped one by one. Each in its own HTTP request, using the <a href="http://www.elasticsearch.org/guide/reference/api/index_.html">Index API</a>. Set it to "on" and it will use Elasticsearch's <a href="http://www.elasticsearch.org/guide/reference/api/bulk.html">Bulk API</a> to send multiple logs in the same request. The maximum number of logs sent in a single bulk request depends on your queue settings - usually limited by the <a href="http://www.rsyslog.com/doc/node35.html">dequeue batch size</a>. More information about queues can be found <a href="http://www.rsyslog.com/doc/node32.html">here</a>.</li>
			<li>
				<strong>parent</strong><br>
				Specifying a string here will index your logs with that string the parent ID of those logs. Please note that you need to define the <a href="http://www.elasticsearch.org/guide/reference/mapping/parent-field.html">parent field</a> in your <a href="http://www.elasticsearch.org/guide/reference/mapping/">mapping</a> for that to work. By default, logs are indexed without a parent.</li>
			<li>
				<strong>dynParent </strong>&lt;on/<strong>off</strong>&gt;<br>
				Using the same parent for all the logs sent in the same action is quite unlikely. So you'd probably want to turn this "on" and specify a <a href="http://www.rsyslog.com/doc/rsyslog_conf_templates.html">template</a> that will provide meaningful parent IDs for your logs.</li>
			<li>
				<strong>uid</strong><br>
				If you have basic HTTP authentication deployed (eg: through the <a href="https://github.com/Asquera/elasticsearch-http-basic">elasticsearch-basic plugin</a>), you can specify your user-name here.</li>
			<li>
				<strong>pwd</strong><br>
				Password for basic authentication.</li>
		</ul>
		<p>
			<b>Samples:</b></p>
		<p>
			The following sample does the following:</p>
		<ul>
			<li>
				loads the omelasticsearch module</li>
			<li>
				outputs all logs to Elasticsearch using the default settings</li>
		</ul>
		<pre>module(load="omelasticsearch")
*.*     action(type="omelasticsearch")</pre>
		<p>
			The following sample does the following:</p>
		<ul>
			<li>
				loads the omelasticsearch module</li>
			<li>
				defines a template that will make the JSON contain the following properties (more info about what properties you can use <a href="http://www.rsyslog.com/doc/property_replacer.html">here</a>):
				<ul>
					<li>
						RFC-3339 timestamp when the event was generated</li>
					<li>
						the message part of the event</li>
					<li>
						hostname of the system that generated the message</li>
					<li>
						severity of the event, as a string</li>
					<li>
						facility, as a string</li>
					<li>
						the tag of the event</li>
				</ul>
			</li>
			<li>
				outputs to Elasticsearch with the following settings
				<ul>
					<li>
						host name of the server is myserver.local</li>
					<li>
						port is 9200</li>
					<li>
						JSON docs will look as defined in the template above</li>
					<li>
						index will be "test-index"</li>
					<li>
						type will be "test-type"</li>
					<li>
						activate bulk mode. For that to work effectively, we use an in-memory queue that can hold up to 5000 events. The maximum bulk size will be 300</li>
					<li>
						retry indefinitely if the HTTP request failed (eg: if the target server is down)</li>
				</ul>
			</li>
		</ul>
		<pre>module(load="omelasticsearch")
template(name="testTemplate"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; type="list"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; option.json="on") {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; constant(value="{")
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; constant(value="\"timestamp\":\"")&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; property(name="timereported" dateFormat="rfc3339")
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; constant(value="\",\"message\":\"")&nbsp;&nbsp;&nbsp;&nbsp; property(name="msg")
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; constant(value="\",\"host\":\"")&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; property(name="hostname")
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; constant(value="\",\"severity\":\"")&nbsp;&nbsp;&nbsp; property(name="syslogseverity-text")
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; constant(value="\",\"facility\":\"")&nbsp;&nbsp;&nbsp; property(name="syslogfacility-text")
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; constant(value="\",\"syslogtag\":\"")&nbsp;&nbsp; property(name="syslogtag")
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; constant(value="\"}")
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }
*.* action(type="omelasticsearch"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; server="myserver.local"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; serverport="9200"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; template="testTemplate"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; searchIndex="test-index"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; searchType="test-type"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; bulkmode="on"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; queue.type="linkedlist"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; queue.size="5000"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; queue.dequeuebatchsize="300"
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; action.resumeretrycount="-1")</pre>


