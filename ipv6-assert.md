# Assertion on IPv6 Host Requests with --disable-ipv6	
## The Issue
When squid is built with the configuration flag `--disable-ipv6`, any request which uses an IPv6 hostname will cause an assertion. This behavior is not documented anywhere, and there are multiple references to the use of this flag scattered throughout official Squid documents.

There is nothing more to say about this, the request to make is simply:
```
GET http://1::/ HTTP/1.1
```
and any server compiled with the flag will crash.
