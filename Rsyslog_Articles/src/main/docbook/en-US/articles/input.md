### input() statement: a quick look ###

The new input() config statement is released. This concludes the major part of the new config 
format for v6 (v7 will also support an enhanced ruleset() statement). This article gives you 
some quick ideas of how the new format looks in practice.  Following is a small test 
rsyslog.conf with the old-style directives commented out and followed by the new style ones. 
Here it is:

    #$ModLoad imfile
    #$inputfilepollinterval 1

    module(
        load="imfile" 
        pollingInterval="1"
    )
>

    #input(type="imuxsock" )

    module(
        load="imuxsock" 
        syssock.use="off"
    )
    input(
        type="imuxsock" 
        socket="/home/rgerhards/testsock"
    )
>

    #$ModLoad imfile
    #$InputFileName /tmp/inputfile
    #$InputFileTag tag1:
    #$InputFileStateFile inputfile-state
    #$InputRunFileMonitor

    module(load="imfile")
    input( type="imfile" file="/tmp/inputfile" tag="tag1:" statefile="inputfile-state")
>

    #$ModLoad imtcp
    #$InputPTCPServerRun 13514
    module(load="imptcp")
    input(type="imptcp" port="13514")
>

    module(load="imtcp" keepalive="on")
    #$InputTCPServerSupportOctetCountedFraming off
    #$InputTCPServerInputName tcpname
    #$InputTCPServerRun 13515

    input(type="imtcp" port="13515" name="tcpname" supportOctetCountedFraming="off")
>

    #$UDPServerRun 13514
    #$UDPServerRun 13515

    input(type="imudp" port="13514")
    input(type="imudp" port="13515")
>
