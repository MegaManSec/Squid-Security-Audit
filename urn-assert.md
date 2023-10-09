# RFC 2141 / 2169 (URN) Assertion Crash

RFC 2141 concerns the so-called 'Uniform Resource Name' (URN) scheme.  While similar to standard URLs, URNs are concerned about identifiers within a specific namespace for long-term availability. One example is `urn:ietf:rfc:2648`, which corresponds to The IETF's RFC 2648.

RFC 2169 concerns the "Trivial Convention for using HTTP in URN Resolution", and can be thought of as an interface between HTTP and URN; effectively, a proxy between the two protocols.

Upon initial connection to Squid, the client is sent a `HTTP/1.1 200 Gatewaying` response, while the connection to the URN resolver is made "asynchronously". Once the response is received from the URN resolver, it is sent to the client.

## The Issue
When a client opens a connection with Squid and subsequently closes it, its so-called 'store entry' is marked as 'pending' (STORE_PENDING) and 'completed' (STORE_OK) respectively. If an attempt is subsequently made to write to that store entry after it has been set to STORE_OK, Squid will crash with an assertion, as it generally means something is 'wrong'.

Due to Squid marking a store as 'completed' upon a client disconnecting, it is possible to trick Squid into requesting an URN resolution, then quickly disconnecting, resulting in a write to a store which has been marked as STORE_OK.
In fact, this process does not even need to be done quickly, and as long as the 'urn server' holds the connection open longer than the client is connected, this assertion will happen.

For example, using a basic `nc` command to simulate a server, it is possible to cause this crash.
A response is set up on port 80 of localhost (for example):
```
printf "ICY/1.1 OK\r\n\r\n" | sudo nc -l 80
```
and a request is made:
```
printf "GET urn:localhost:2041210X:media:mee312659:mee312659-math HTTP/1.1\r\n\r\n" | nc localhost 3128
```

The request is made to the 'server', which receives the following:
```
GET /uri-res/N2L?urn:localhost:2041210X:media:mee312659:mee312659-math HTTP/1.1
Accept: text/plain
Host: localhost
Cache-Control: max-age=3600
Connection: keep-alive
```
However, the connection is not closed because a full response has not been received. 

In the meantime, the client can simply cancel their request. This in of itself will not cause a crash, but once the 'server '(i.e. `nc`) is either canceled or simply times out, an assertion will happen.

