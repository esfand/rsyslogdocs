### parsing JSON-enhanced syslog ###

Strucuted logging is cool. A couple of month ago, I added support for log normalization and 
the 0.5 draft CEE standard to rsyslog. At last weeks Fedora Developer's Conference, there was 
a huge agreement that CEE-like JSON is a great way to enhance syslog logging. To follow up on 
this concept, I have integrated a JSON decoder into libee, so that it can now decode JSON with 
a single method call. It's a proof of concept, and for serious use performance optimization 
needs to be done. Besides that, it's already quite solid.

Also, I just added the mmjsonparse message modification module to rsyslog (available now in 
git master branch!). It checks if the message contains an "@JSON: " cookie and, if so, tries 
to parse the resulting string as JSON. If that succeeds, we obviously have a JSON-enhanced 
message and the individual name/value pairs are stored and can be used both in filters and 
output templates. This provides some really great opportunities when it comes to processing 
the structured data. Just think about RESTful interfaces and such!

Right now, everything is at proof of concept level, but works well enough for you to try it. 
I'll smoothen some edges but will release the versions rather soon. Probably the biggest drawback 
is that the JSON processor currently flattens the event, with structure being conveyed via 
field names. That means if you have a JSON object "SUPER" containing a number of fields "field1" 
to "fieldn", the current implementation will be a single level and the names are "SUPER.field1",... 
I did this in order to have a quick solution and one that fits into the existing framework. 
I'll work on creating real structure soon. It's not really hard, but I probably do some other PoCs first ;)

I considered several approaches, among them moving over to libcollection (part of ding-libs) or 
a pure JSON parser. The more I worked with the code, the more it turned out that libee already has 
a lot of the necessary plumbing and could simply been enhanced/modified under the hood. 

The big plus 
in that approach is that is immediately plugs in into rsyslog and the other solutions that already 
built on it. This even enables using the new functionality in the v6 context (I originally thought 
I'd need to move on to rsyslog v7 for the name-value pair changes). 

Now that I have written mmjsonparse, 
this really seems to work out. No engine change was required, and I expect little need for change even
for the final version. As such, I'll proceed in that direction. Actually, what I now use is kind of
a hybrid approch: I use a lot of philosophy of libcollection, which showed me the right route to take. 
Then, I use cJSON, which is a really nice JSON parser. 

In the proof of concept, I use both 
cJSON's object model and libee's own. I expect to merge them, actually tightly integrating cJSON.
The reason is that CEE has evolved quite a bit in the mean time, and many complex constructs are 
no longer required. As such, I can streamline the library as well, what not only reduces complexity 
but speeds up the whole process.

