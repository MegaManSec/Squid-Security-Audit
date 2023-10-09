
# Pipeline Prefetch Assertion With Invalid Headers
HTTP/1.1 (RFC 2616 / 7231) brought forth one of the largest latency-enhancing features that HTTP was lacking beforehand: pipeling. Pipelining is where one TCP connection from a client to a server could be used for _multiple requests_ at the same time. This meant that instead of taking the time to negotiate a TCP connection, send an HTTP request -- wait for its response -- then send the next request, a client could simply send multiple requests in one burst. More information about pipeling can be found [here](https://developer.mozilla.org/ar/docs/Web/HTTP/Connection_management_in_HTTP_1.x). 

While pipelining concerns the client requests, *pipeline prefetching* concerns the actions that the server may do when it receives a pipelined request. When a proxy server receives a pipelined request, it generally fetches each request one-by-one,  and then responds to the client with all the responses packed together. However, with pipeline prefetching, a proxy server can instead parse every request which has been pipelined, and asynchronously request each page, in order to speed up the process of obtaining each of the upstream responses, in order to pack them together and send them back to the client.
Squid supports pipeline prefetching with the `pipeline_prefetch` configuration. Documentation states:
```
HTTP clients may send a pipeline of 1+N requests to Squid using asingle connection, without waiting for Squid to respond to the firstof those requests.
This option limits the number of concurrentrequests Squid will try to handle in parallel.
If set to N, Squid will try to receive and process up to 1+N requests on the sameconnection concurrently.
```

## The Issue
Normally, a tunnel (i.e. CONNECT) should not be handled with pipeline prefetching. This is not always the case, however, and a user can forcefully create a tunnel using an unsupported type of request, which will make Squid assert after the request is made.

An example request:
```
GET http://0

0
```

The assertion:
```
assertion  failed: client_side.cc:1589: "conn->pipeline.front() == context"
```

No further information is provided in this document.
