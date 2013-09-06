### General Queue Parameters ###

Queues need to be configured in the action or ruleset it should affect. 
If nothing is configured, default values will be used. Thus, the default ruleset has 
only the default main queue. Specific Action queues are not set up by default.

* **queue.filename**  name

* **queue.size**  number

* **queue.dequeuebatchsize**  number    
    default 16
* **queue.maxdiskspace**  number    

* **queue.highwatermark**  number    
    default 8000
* **queue.lowwatermark**  number    
    default 2000
* **queue.fulldelaymark**  number    

* **queue.lightdelaymark**  number     

* **queue.discardmark**  number    
    default 9750
* **queue.discardseverity**  number    
    numerical severity default 8 (nothing discarded)
* **queue.checkpointinterval**  number    

* **queue.syncqueuefiles**  on/off    

* **queue.type**  [FixedArray/LinkedList/Direct/Disk]    

* **queue.workerthreads**  number    
    number of worker threads, default 1, recommended 1
* **queue.timeoutshutdown**  number    
    number is timeout in ms, default 0 (indefinite)
* **queue.timeoutactioncompletion**  number    
    number is timeout in ms, default 1000, 0 means immediate!
* **queue.timeoutenqueue**  number    
    number is timeout in ms, default 2000, 0 means indefinite
* **queue.timeoutworkerthreadshutdown**  number    
    number is timeout in ms, default 60000 (1 minute)
* **queue.workerthreadminimummessages**  number    
    default 100
* **queue.maxfilesize**  size_nbr    
    default 1m
* **queue.saveonshutdown**   on/off    

* **queue.dequeueslowdown**  number    
    number is timeout in microseconds, default 0 (no delay). Simple rate-limiting.
* **queue.dequeuetimebegin**  number    

* **queue.dequeuetimeend**  number    

**Sample:**

The following is a sample of a TCP forwarding action with its own queue.

    module (load="builtin:omfwd")
    *.* action(
            type="omfwd" 
            target="192.168.2.11"
            port="10514"
            protocol="tcp"
            queue.filename="forwarding"
            queue.size="1000000"
            queue.type="LinkedList"
        )
