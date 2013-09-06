
Using the Text File Input Module
---------------------------------

Friday, March 25th, 2011

Log files should be processed by rsyslog. Here is some information on how the file monitor works. 
This will only describe setting up the Text File Input Module. Further configuration like processing 
rules or output methods will not be described.

Things to think about

The configuration given here should be placed on top of the rsyslog.conf file.

Config Statements

    module(load="imfile" PollingInterval="10")
    # needs to be done just once. PollingInterval is a module directive and is only 
    # set once when loading the module
    # File 1
    input(
        type="imfile" 
        File="/path/to/file1" 
        Tag="tag1" 
        StateFile="/var/spool/rsyslog/statefile1" 
        Severity="error" 
        Facility="local7"
    )
    # File 2
    input(
        type="imfile" File="/path/to/file2" 
        Tag="tag2" 
        StateFile="/var/spool/rsyslog/statefile2")
        # ... and so on ...
        #
        
How it works

The configuration for using the Text File Input Module is very extensive. At the beginning of your 
rsyslog configuration file, you always load the modules. There you need to load the module for 
Text File Input as well. Like all other modules, this has to be made just once. Please note that 
the directive PollingInterval is a module directive which needs to be set when loading the module.

    module(load="imfile" PollingInterval="10")

Next up comes the input and its parameters. We configure a input of a certain type and then set the 
parameters to be used by this input. This is basically the same principle for all inputs:

    # File 1
    input(type="imfile" File="/path/to/file1" 
        Tag="tag1" 
        StateFile="/var/spool/rsyslog/statefile1" 
        Severity="error" 
        Facility="local7"
    )
        
File specifies, the path and name of the text file that should be monitored. The file name must 
be absolute.

Tag will set a tag in front of each message pulled from the file. If you want a colon after the 
tag you must set it as well, it will not be added automatically.

StateFile will create a file where rsyslog keeps track of the position it currently is in a file. 
You only need to set the filename. This file always is created in the rsyslog working directory 
(configurable via $WorkDirectory). This file is important so rsyslog will not pull messages from 
the beginning of the file when being restarted.

Severity will give all log messages of a file the same severity. This is optional. 
By default all mesages will be set to “notice”.

Facility gives alle log messages of a file the same facility. Again, this is optional. 
By default all messages will be set to “local0″.

These statements are needed for monitoring a file. There are other statements described in the doc, 
which you might want to use. If you want to monitor another file the statements must be repeated.

Since the files cannot be monitored in genuine real time (which generates too much processing effort) 
you need to set a polling interval:

PollingInterval 10

This is a module setting and it defines the interval in which the log files will be polled. 
By default this value is set to 10 seconds. If you want this to get more near realtime, you 
can decrease the value, though this is not suggested due to increasing processing load. 
Setting this to 0 is supported, but not suggested. Rsyslog will continue reading the file 
as long as there are unprocessed messages in it. The interval only takes effect once rsyslog 
reaches the end of the file.

Important

The StateFile needs to be unique for every file that is monitored. If not, strange things could happen.

