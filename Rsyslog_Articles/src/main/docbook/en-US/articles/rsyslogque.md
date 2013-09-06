# Main Queue #
This object is available since 7.5.3. It permits to specify parameters for the main message queue. Note that only queue-parameters are permitted for this config object. This permits to set the same options like in ruleset and action queues. A special statement is needed for the main queue, because it is a different object and cannot be configured via any other object.

Note that when the main_queue() object is configured, the legacy `$MainMsgQ...` statements are ignored.
    
## A Simple Example ##

    main_queue(    
        queue.size="100000"      # how many messages (messages, not bytes!) to hold in memory
        queue.type="LinkedList"  # allocate memory dynamically for the queue. Better for handling spikes
    )
>    

    # location for storing queue files on disk
    $WorkDirectory /var/lib/elasticsearch/rsyslog_queues

    main_queue(
        # file name template, also enables disk mode for the memory queue
        queue.fileName incoming_queue

        # allocate memory dynamically for the queue. Better for handling spikes
        Queue.Type LinkedList

        # when to start discarding messages
        queue.DiscardMark 2000000

        # limit of messages in the memory queue. When this is reached, it starts to write to disk
        queue.HighWaterMark 1000000

        # memory queue size at which it stops writing to the disk
        queue.LowWaterMark 800000

        # maximum disk space used for the disk part of the queue
        queue.MaxDiskSpace 5g

        # how many messages (messages, not bytes!) to hold in memory
        queue.Size 8000000

        # don't throttle receiving messages when the queue gets full
        queue.TimeoutEnqueue 0

        # save the queue contents when stopping rsyslog
        queue.SaveOnShutdown on
    )

    
# Action Queues #
In v6+, queue parameters are set directly within the action:

    *.* action(type="omfile"
               queue.filename="nfsq"
               queue.type="linkedlist"
               file="/var/log/log1")    
> 

    *.* action(type="omfile"
               queue.filename="diskq"
               queue.type="linkedlist"
               file="/var/log/log2")

>  

    *.* action(type="omfile"
               file="/var/log/log3") 
>    


    $template miniSchema,"{\"message\":\"%msg:::json%\",\"host\":\"%HOSTNAME:::json%\",\"severity\":\"%syslogseverity%\",\"date\":\"%timereported:1:19:date-rfc3339%.%timereported:1:3:date-subseconds%\",\"tag\":\"%syslogtag:::json%\"}"
    $template srchidx,"%timereported:1:10:date-rfc3339%"
    *.*     action(type="omelasticsearch" 
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


    
# Queue Parameters #

Queues need to be configured in the action or ruleset it should affect. If nothing is configured, default values will be used. Thus, the default ruleset has only the default main queue. Specific Action queues are not set up by default.

*  **queue.filename** name

*  **queue.size** number

*  **queue.dequeuebatchsize** number    
default 16

*  **queue.maxdiskspace** number

*  **queue.highwatermark** number    
default 8000

*  **queue.lowwatermark** number    
default 2000

*  **queue.fulldelaymark** number

*  **queue.lightdelaymark** number

*  **queue.discardmark** number    
default 9750

*  **queue.discardseverity** number    
numerical severity! default 8 (nothing discarded)

*  **queue.checkpointinterval** number

*  **queue.syncqueuefiles** on/off

*  **queue.type** [FixedArray/LinkedList/Direct/Disk]

*  **queue.workerthreads** number    
number of worker threads, default 1, recommended 1

*  **queue.timeoutshutdown** number     
number is timeout in ms,   
default 0 (indefinite)

*  **queue.timeoutactioncompletion** number    
number is timeout in ms (1000ms is 1sec!),    
default 1000, 0 means immediate!

*  **queue.timeoutenqueue** number    
number is timeout in ms (1000ms is 1sec!),    
default 2000, 0 means indefinite

*  **queue.timeoutworkerthreadshutdown** number    
number is timeout in ms,    
default 60000 (1 minute)

*  **queue.workerthreadminimummessages** number    
default 100

*  **queue.maxfilesize** size_nbr    
default 1m

*  **queue.saveonshutdown** on/off
 
*  **queue.dequeueslowdown** number    
number is timeout in microseconds,    
default 0 (no delay). Simple rate-limiting!

*  **queue.dequeuetimebegin** number    

*  **queue.dequeuetimeend** number    
    

###Sample###
The following is a sample of a TCP forwarding action with its own queue.

    Module (load="builtin:omfwd")
    *.* action(Type="omfwd" 
               Target="192.168.2.11"
               Port="10514"
               Protocol="tcp" 
      
               queue.filename="forwarding"
               queue.size="1000000"
               queue.type="LinkedList"
        )

This is the end