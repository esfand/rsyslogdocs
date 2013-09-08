### why disk-assisted queues keep a file open ###

From time to time, someone asks why rsyslog disk-assisted queues keep one file open until shutdown. So it probably is time to elaborate a bit about it.

Let's start with explaining what can be seen: if a disk-assisted queue is configured, rsyslog will normally use the in-memory queue. It will open a disk queue only if there is good reason to do so (because it severely hurts performance). The prime reason to go to disk is when the in memory queue's configured size has been exhausted. Once this happens, rsyslog begins to create spool files, numbered consequtively. This should normally happen in cases where e.g. a remote target is temporarily not reachable or a database engine is responding extremely slow.  Once the situation has been cleared, rsyslog clears out the spool files. However, one spool file always is kept until you shut rsyslog down.

This is expected behaviour, and there is good reason for it. This is what happens technically:

A DA queue is actually "two queues in one": It is a regular in-memory queue, which has a (second) helper disk queue in case it neeeds it. As long as operations run smoothly, the disk queue is never used. When the system starts up, only the in-memory queue is started. Startup of the disk queue is deferred until it is actually needed (as in most cases it will never be needed).

When the in-memory queue runs out of space, it starts that Disk queue, which than allocates its queue file. In order to reclaim space, not a single file is written but a series of files, where old files are deleted when they are processed and new files are created on an as-needed basis. Initially, there is only one file, which is read and written. And if the queue is empty, this single file still exists, because it is the representation of a disk queue (like the in-memory mapping tables for memory queues always exist, which just cannot be seen by the user).

So what happens in the above case is that the disk queue is created, put into action (the part where it writes and deletes files) and is then becoming idle and empty. At that stage, it will keep its single queue file, which holds the queue's necessary mappings.

One may now ask "why not shut down the disk queue if no longer needed"? The short answer is that we anticpiate that it will be re-used and thus we do not make the effort to shut it down and restart when the need again arises. Let me elaborate: experience tells that when a system needs the disk queue once, it is highly likely to need it again in the future. The reason for disk queues to kick into action are often cyclic, like schedule restarts of remote systems or database engines (as part of a backup process, for example). So we assume if we used it once, we will most probably need it again and so keep it ready. This also helps reduce potential message loss during the switchover process to disk - in extreme cases this can happen if there is high traffic load and slim in-memory queues (remember that starting up a disk queue needs comparativley long).

The longer answer is that in the past rsyslog tried to shut down disk queues to "save" ressources. Experience told us that this "saving" often resulted in resource wasting. Also, the need to synchronize disk queue shutdown with the main queue was highly complex (just think about cases where we shut it down at the same time the in-memory queue actually begins to need it again!). This caused quite some overhead even when the disk queue was not used (again, this is the case most of the time - if properly sized). An indication of this effect probably is that we could remove roughly 20% of rsyslog's queue code when we removed the "disk queue shutdown" feature!

Bottom line: keeping the disk queue active once it has been activated is highly desirable and as such seeing a queue file around until shutdown is absolutely correct. Many users will even never see this spool file, because there are no exceptional circumstances that force the queue to go to disk. So they may be surprised if it happens every now and then.

Side-Note: if disk queue files are created without a traget going offline, one should consider increasing the in-memory queue size, as disk queues are obviouly much less efficient than memory queues.