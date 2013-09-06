### How to bind a template ###

This little FAQ describe how to bind a template.
First with the new template format “list” and then with the old “legacy” format.

First off all you have to define a template for example for specify output.

Here is an example template in the list format:

    template(
        name=”FileFormat”
        type=”list”
    ) 
    {
        property(name=”timestamp” dateFormat=”rfc3339″)
        constant(value=” “)
        property(name=”hostname”)
        constant(value=” “)
        property(name=”syslogtag”)
        constant(value=” “)
        property(name=”msg” spifno1stsp=”on” )
        property(name=”msg” droplastlf=”on” )
        constant(value=”\n”)
    }

Then you have to bind the template to an action. The Syntax to bind a template is:

    Action;name-of-template

Here is an example action with a example-template:

    *.* action(type=”omfile” file=”/var/log/all-messages.log”);Name-of-your-template

In the configuration it should looks like this:

    template(name=”FileFormat” type=”list”) {
        property(name=”timestamp” dateFormat=”rfc3339″)
        constant(value=” “)
        property(name=”hostname”)
        constant(value=” “)
        property(name=”syslogtag”)
        constant(value=” “)
        property(name=”msg” spifno1stsp=”on” )
        property(name=”msg” droplastlf=”on” )
        constant(value=”\n”)
    }
    action(
        type=”omfile” 
        file=”/var/log/all-msgs.log”
    );FileFormat
    “

Here is an example for the legacy format

Here is an example template in the legacy format:

    $template ExampleFormat,”%timereported:::date-rfc3339% %HOSTNAME% %msg%”

Here is an example action with a example-template:

    *.* /var/log/all-messages.log;Your-Template-Name

In the Configuration it looks like this:

    “$template ExampleFormat,”%timereported:::date-rfc3339% %HOSTNAME% %msg%”
    *.* /var/log/all-messages.log;ExampleFormat”

