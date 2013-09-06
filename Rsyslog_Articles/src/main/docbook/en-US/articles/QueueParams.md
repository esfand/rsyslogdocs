Queue Parameters
----------------



queue.filename
--------------

Specifes the base name to be used for queue files.

> Available Since: 6.3.3    
> Format: word    
> Default: none    
> Mandatory: yes (for disk-based queues)    
 	 
Disk-based queues create a set of files for queue content. The value set via queue.filename acts as the basename to be used for filename creation. For actual log data, a number is appended to the file name. There is also a so-called "queue information" (qi) file created, which holds administrative information about the queue status. This file is named with the base name plus ".qi" as suffix.


queue.size
----------
Specifes the maximum number of (in-core) messages a queue can hold.

> Available Since: 6.3.3    
> Format: size    
> Default: 10,000 for ruleset queues, 1,000 for action queues    
> Mandatory: no    
 	 
This setting affects the in-memory queue size. Disk based queues may hold more data inside the queue, but not in main memory but on disk. The size is specified in number of messages. The representation of a typical syslog message object should require less than 1K, but excessively large messages may also result in excessively large objects. Note that not all message types may utilize the full queue. This depends on other queue parameters like the watermark settings. Most importantly, a small amount (seven percent) is reserved for messages with high loss potential (like UDP-received messages) and will not be utilized by messages with lower loss potential (like TCP-received messages).

Warning: do not set the size to extremely small values (like less than 500 messages) unless you know exactly what you do (and why!). This could interfere with other internal settings like watermarks and batch sizes. It is possible to specify very small values in order to support power users who customize the other settings accordingly. Usually there is no need to do that. Queues take only up memory when messages are stored in them. So reducing queue sizes does not reduce memory usage, except in cases where queues are actually full. The default settings permit small message bursts to be buffered without message loss.



queue.dequeuebatchsize
----------------------
Specifies how many messages can be dequeued at once.

> Available Since: 6.3.3    
> Format: number    
> Default:	 
> Mandatory: no
 	 
Specifies the batch size for dequeue operations. This setting affects performance. As a rule of thumb, larger batch sizes (up to a environment-induced upper limit) provide better performance. For the average system, there usually should be no need to adjust batch sizes as the defaults are sufficient.


queue.maxdiskspace
------------------
Specifies maximum amount of disk space a queue may use.

> Available Since: 6.3.3    
> Format: size    
> Default: unlimited    
> Mandatory: no
 	 
This setting permits to limit the maximum amount of disk space the queue data files will use. Note that actual disk allocation may be slightly larger due to block allocation. Also, no partial messages are written to queue, so writing a message is completed even if that means going slightly above the limit. Note that, contrary to queue.size, the size is specified in bytes and not messages. It is recommended to limit queue disk allocation, as otherwise the filesystem free space may be exhausted if the queue needs to grow very large.
If the size limit is hit, messages are discarded until sufficient messages have been dequeued and queue files been deleted


queue.highwatermark
-------------------
Specifies ...

Available Since: 6.3.3    
Format: number    
Default:    
Mandatory: no

queue.lowwatermark
------------------
Specifies ... 

Available Since: 6.3.3    
Format: number    
Default:    
Mandatory: no

queue.fulldelaymark
-------------------
Specifies .

Available Since: 6.3.3    
Format: number    
Default:    
Mandatory: no


queue.discardmark
-----------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no


queue.discardseverity
---------------------
Specifies

Available Since:	6.3.3
Format:	severity
Default:	 
Mandatory:	no

queue.checkpointinterval
------------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no


queue.syncqueuefiles
--------------------
Specifies

Available Since:	6.3.3
Format:	binary
Default:	 
Mandatory:	no

queue.type
----------
Specifies

Available Since:	6.3.3
Format:	queue type
Default: LinkedList for ruleset queues, Direct for action queues
Mandatory:	no


queue.workerthreads
-------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no

queue.timeoutshutdown
---------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no


queue.timeoutactioncompletion
-----------------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no


queue.timeoutenqueue
--------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no


queue.timeoutworkerthreadshutdown
---------------------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no

queue.workerthreadminimummessages
---------------------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no


queue.maxfilesize
-----------------
Specifies

Available Since:	6.3.3
Format:	size
Default:	 
Mandatory:	no


queue.saveonshutdown
--------------------
Specifies

Available Since:	6.3.3
Format:	binary
Default:	no
Mandatory:	no

queue.dequeueslowdown
---------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no

queue.dequeuetimebegin
----------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no

queue.dequeuetimeend
--------------------
Specifies

Available Since:	6.3.3
Format:	number
Default:	 
Mandatory:	no



