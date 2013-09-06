# gtls Network Stream Driver #

This [network stream driver](http://www.rsyslog.com/doc/netstream.html) implements a TLS protected transport via the [GnuTLS library](http://www.gnu.org/software/gnutls/).

**Supported Driver Modes**
* 0 - unencrypted transmission (just like <a href="ns_ptcp.html">ptcp</a> driver)</li>
* 1 - TLS-protected operation</li>

Note: mode 0 does not provide any benefit over the ptcp driver. This
mode exists for technical reasons, but should not be used. It may be
removed in the future.

<span style="font-weight: bold;">Supported Authentication Modes</span><br>
<ul>
<li><span style="font-weight: bold;">anon</span>
- anonymous authentication as
described in IETF's draft-ietf-syslog-transport-tls-12 Internet draft</li>
<li><span style="font-weight: bold;">x509/fingerprint</span>
- certificate fingerprint authentication as
described in IETF's draft-ietf-syslog-transport-tls-12 Internet draft</li>
<li><span style="font-weight: bold;">x509/certvalid</span>
- certificate validation only</li>
<li><span style="font-weight: bold;">x509/name</span>
- certificate validation and subject name authentication as
described in IETF's draft-ietf-syslog-transport-tls-12 Internet draft
</li>
</ul>

Note: "anon" does not permit to authenticate the remote peer. As such,
this mode is vulnerable to man in the middle attacks as well as
unauthorized access. It is recommended NOT to use this mode.

x509/certvalid is a nonstandard mode. It validates the remote
peers certificate, but does not check the subject name. This is 
weak authentication that may be useful in scenarios where multiple
devices are deployed and it is sufficient proof of authenticity when
their certificates are signed by the CA the server trusts. This is
better than anon authentication, but still not recommended.

**Known Problems**

Even in x509/fingerprint mode, both the client and sever
certificate currently must be signed by the same root CA. This is an
artifact of the underlying GnuTLS library and the way we use it. It is
expected that we can resolve this issue in the future.


# Network Stream Drivers #

Network stream drivers are a layer between various parts of rsyslogd (e.g. the imtcp module) 
and the transport layer. They provide sequenced delivery, authentication and confidentiality 
to the upper layers. Drivers implement different capabilities.

Users need to know about netstream drivers because they need to configure the proper driver, 
and proper driver properties, to achieve desired results (e.g. a TLS-protected syslog transmission).

The following drivers exist:
* **ptcp** - the plain tcp network transport (no security)
* **gtls** - a secure TLS transport implemented via the GnuTLS library


# ptcp Network Stream Driver #

This network stream driver implement a plain tcp transport without security properties.

Supported Driver Modes

* **0** - unencrypted trasmission

**Supported Authentication Modes**

* **anon** - no authentication
