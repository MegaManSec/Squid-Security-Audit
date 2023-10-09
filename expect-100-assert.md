# Pipeline Prefetch Assertion With Double 'Expect:100-continue' Request Headers
HTTP/1.1 (RFC 2616 / 7231) brought forth one of the largest latency-enhancing features that HTTP was lacking beforehand: pipeling. Pipelining is where one TCP connection from a client to a server could be used for _multiple requests_ at the same time. This meant that instead of taking the time to negotiate a TCP connection, send an HTTP request -- wait for its response -- then send the next request, a client could simply send multiple requests in one burst. More information about pipeling can be found [here](https://developer.mozilla.org/ar/docs/Web/HTTP/Connection_management_in_HTTP_1.x). 

While pipelining concerns the client requests, *pipeline prefetching* concerns the actions that the server may do when it receives a pipelined request. When a proxy server receives a pipelined request, it generally fetches each request one-by-one,  and then responds to the client with all the responses packed together. However, with pipeline prefetching, a proxy server can instead parse every request which has been pipelined, and asynchronously request each page, in order to speed up the process of obtaining each of the upstream responses, in order to pack them together and send them back to the client.
Squid supports pipeline prefetching with the `pipeline_prefetch` configuration. Documentation states:
```
HTTP clients may send a pipeline of 1+N requests to Squid using asingle connection, without waiting for Squid to respond to the firstof those requests.
This option limits the number of concurrentrequests Squid will try to handle in parallel.
If set to N, Squid will try to receive and process up to 1+N requests on the sameconnection concurrently.
```

The `Expect: 100-continue` request header has a special meaning within the context of HTTP/1.1 (RFC 7231). It basically asks the server about the status of the client and its request, and more or less, whether "_everything is already_". If there are no problems, the server is expected to reply with the `100 Continue` response status.

## The Issue
Squid is able to handle `100-continue` requests. It does this in the `Http::One::Server::writeControlMsgAndCall` function, which, among other things, writes a response to the client nearly immediately:
```
Comm::Write(clientConnection, mb, call);
```

Once a client connection is written to, it is usually marked as 'active', until an 'asynchronous' handler unmarks it as active after the write is finished.
However, when using pipelining, the 'asynchronous' handler does not have time to run before the next pipelined request is parsed due to a loop, and thus, the same function is run again, and the `Comm::Write` function is called again. The ``Comm::Write`` function has an assertion:

```
Comm::Write(const Comm::ConnectionPointer &conn, const char *buf, int size, AsyncCall::Pointer &callback, FREE * free_func)
{   
    debugs(5, 5, HERE << conn << ": sz " << size << ": asynCall " << callback);

    /* Make sure we are open, not closing, and not writing */
    assert(fd_table[conn->fd].flags.open);
    assert(!fd_table[conn->fd].closing());
    Comm::IoCallback *ccb = COMMIO_FD_WRITECB(conn->fd);
    assert(!ccb->active());
```

Since the connection is marked as being written to in the first request and will only be marked as 'not active (`ccb->active()`)' after the request has been parsed, the pipelining 'loop' will precede the asynchronous function which will mark the write as inactive.
Thus, an assertion will occur if a pipelined request with two `100-continue` headers is sent to Squid.

An example is given:
```
GET http://0 HTTP/1.1
Expect:100-continue

GET http://0 HTTP/1.1
Expect:100-continue
```
which will cause the assertion: 
```
assertion failed: Write.cc:43: "!ccb->active()"
```
