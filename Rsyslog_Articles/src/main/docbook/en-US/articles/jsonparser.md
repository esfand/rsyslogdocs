### JSON Parser Module - mmjsonparse###


#### Description ####

This module provides support for parsing structured log messages that follow the CEE/lumberjack spec. 
The so-called "CEE cookie" is checked and, if present, the JSON-encoded structured message content is parsed. 
The properties are than available as original message properties.

#### Sample ####

This activates the module and applies normalization to all messages:

    module(load="mmjsonparse")
    action(type="mmjsonparse")

The same in legacy format:

    $ModLoad mmjsonparse
    *.* :mmjsonparse:

