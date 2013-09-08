## Filter optimization with arrays

[Thursday, January 10th, 2013](http://www.rsyslog.com/tag/template/)

If you are using a lot of filters and templates in rsyslog, this can not only be affecting the 
performance drastically, but it is also a hassle to set up all the different actions and templates. 
It is always worthy to check, if there isn’t a shortcut somewhere, which might not only save you 
time for creating the configuration, but also make it much simpler in the end to keep track of all
the actions.

In our example, we have several programnames. The log messages should be sorted by programname and then be stored in a specific file and be sorted by host. After storing the log messages, the message should be discarded, so it won’t be processed by the following filters, thus saving otherwise wasted processing time. 

This example is applicable to rsyslog v7.2.x and above.

Here are some sample config lines.

### apache_access

    template(
        name="DailyPerHost_apache_access" 
        type="string" 
        string="/syslog/%FROMHOST%/apache_access.log"
    )

    :programname, isequal, "apache_access" { 
        action(
            type="omfile"
            DynaFile="DailyPerHost_apache_access"
        ) 
        stop 
    }

### apache_error

    template(
        name="DailyPerHost_apache_error"
        type="string" 
        string="/syslog/%FROMHOST%/apache_error.log"
    )

    :programname, isequal, "apache_error" { 
        action(
            type="omfile" 
            DynaFile="DailyPerHost_apache_error"
        ) 
        stop 
    }

### mysql

    template(
        name="DailyPerHost_mysql" 
        type="string"
        string="/syslog/%FROMHOST%/mysql.log"
    )
    :programname, isequal, "mysql" { 
        action(
            type="omfile" 
            DynaFile="DailyPerHost_mysql"
        ) 
        stop
    }

### php

    template(
        name="DailyPerHost_php"
        type="string" 
        string="/syslog/%FROMHOST%/php.log"
    )
    :programname, isequal, "php" { 
        action(
            type="omfile" 
            DynaFile="DailyPerHost_php"
        ) 
        stop
    }

These are some basic services, which are often run together. Please note, that these 
are just a few examples. As you can see here, the template is created first. 
It is given a name, type and format. Templates of type string are usually used for file names. 
Here the log messages get stored in the folder /syslog a subfolder for the host 
where the message occured and then a filename which reflects the type of message 
that occured.

The second line holds the actions. First you see the property based filter (programname) 
and the condition. After that the actions get chained with the curly braces. 
The log messages where the filter evaluates to true get stored in a file. 
The filename and path is generated dynamically with the DynaFile parameter. 
Through this, the above written template will be used to generate the path and filename. 
The second action is represented by stop. Please note that this is case sensitive. 
Basically, stop means to stop the message processing. No further processing of the 
message will take place.

If we look closely at the sample config lines, we see, that the filter condition is 
basically always the same. It will always filter the programname property for a certain value. 
This is a predestinated case for using an array for simplification. 
We can use the property programname in the file template as well and filter an array of values. 
This will greatly save the overhead for all the seperate filter, not only in the configuration, 
but also in processing the messages.

    template(
        name="DailyPerHost_app" 
        type="string" 
        string="/syslog/%FROMHOST%/%programname%.log"
    )
    if $programname == ["apache_access",
                        "apache_error",
                        "mysql",
                        "php"]
    then {
        action(
            type="omfile" 
            DynaFile="DailyPerHost_app"
        )
        stop
    }

Again, we first create the template. Please note the difference in the filename where the hardcoded text has been replaced by the property programname. In the next lines, we see the filter and the array of values. This is just to reflect the example. Virtually, the array can have near-infinite values. The filter is also a common if/then construct. After the then we see our chain of commands. First the action which writes the log messages into a file where the filename is created by the above template and then a stop as second action.

This case is applicable in many forms. It is also most useful if you are filtering and the discarding a lot of messages with very common filter settings. You could use it to filter for an array of property values and even chain comparison operations.

