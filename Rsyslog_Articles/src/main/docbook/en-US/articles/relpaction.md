### Actions with directives ###

This snippet will show, how Action directives need to be applied to work properly. 
We will show it with the RELP output module. RELP should ensure a safe and loss-free 
transmission between two machines. But if not configured properly, messages may get lost anyway. 

This is mainly meant for any client side configuration.

First of all you have to enable the RELP module.

To load the module use this:

    module(name=omrelp)

To make sure, messages will not get dropped in the event the receiver is not available, 
we basically  need the following directives.  
Additionaly, the queued messages should get saved to the harddrive if the client service needs to shut down. 
It is followed by a forwarding action via RELP to our remote server.

    $ActionQueueType            LinkedList  # use asynchronous processing
    $ActionQueueFileName        srvrfwd     # set file name, also enables disk mode
    $ActionResumeRetryCount     -1          # infinite retries on insert failure
    $ActionQueueSaveOnShutdown  on          # save in-memory data if rsyslog shuts down
    *.* :omrelp:192.168.152.2:20514    
<br/>

    action(
        type                   "omrelp" 
        template               "RSYSLOG_ForwardFormat"
        target                 "192.168.152.2:20514"
        port                   "2514"
        action.resumretrycount -1                     # infinite retries on insert failure
        queue.type             "LinkedList"           # use asynchronous processing
        queue.filename         "srvrfwd"              # set file name, also enables disk mode
        queue.savedonshutdown  "on"                   # save in-memory data if rsyslog shuts down
    )

**Attention**: The directives are only valid for the next configured action!     
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; So you have to set the directives each time you use a new action.

Here is an example with two actions.

    #first action
    $ActionQueueType LinkedList # use asynchronous processing
    $ActionQueueFileName srvrfwd # set file name, also enables disk mode
    $ActionResumeRetryCount -1 # infinite retries on insert failure
    $ActionQueueSaveOnShutdown on # save in-memory data if rsyslog shuts down
    :syslogtag, isequal, “app1″ :omrelp:192.168.152.2:20514
>

    #second action
    $ActionQueueType LinkedList # use asynchronous processing
    $ActionQueueFileName srvrfwd # set file name, also enables disk mode
    $ActionResumeRetryCount -1 # infinite retries on insert failure
    $ActionQueueSaveOnShutdown on # save in-memory data if rsyslog shuts down
    :syslogtag, isequal, “app2″ :omrelp:192.168.152.3:20514

As you can see, we have the whole block of directives mulitple times. But this time, we filter the message 
for the syslogtag and have the diffenrently tagged messages sent to different receivers. 
Now if the receiver is not available, sending the messages will be retried until it is back up again.

If the local rsyslog needs to shut down, all queued messages get written to disk without being lost.
