
# Pipeline Prefetch Assertion With Invalid Headers
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
Squid is able to handle `100-continue` requests. It does this in the `Http::One::Server::writeControlMsgAndCall` function. Before this function is called, some checks are made, and some other functions are called. The specifics are left our of this document.

However, the function `Http::One::Server::proceedAfterBodyContinuation` is called, which calls the function `clientProcessRequest`. `clientProcessRequest` does, among other things, the following:
```
clientProcessRequest(ConnStateData *conn, const Http1::RequestParserPointer &hp, Http::Stream *context)
{   
    ClientHttpRequest *http = context->http;

    HttpRequest::Pointer request = http->request;
[...]
    if (!isFtp) {
        // XXX: for non-HTTP messages instantiate a different Http::Message child type
        // for now Squid only supports HTTP requests
        const AnyP::ProtocolVersion &http_ver = hp->messageProtocol();
        assert(request->http_ver.protocol == http_ver.protocol);
        request->http_ver.major = http_ver.major;
        request->http_ver.minor = http_ver.minor;
    }

```
When using pipeline prefetching, there is the incorrect assumption that every request MUST have the same protocol, when using `Expect: 100-continue`. Within the context of this code, that should _always_ be `AnyP::PROTO_HTTP`. However, if a pipelined request does not contain a valid protocol header, it will be set as `AnyP::PROTO_NONE` by default.

It is thus then trivial to cause an assertion in this code with the following request:
```
GET http://1/ HTTP/1.1
Expect: 100-continue

RandomBits
```
which will cause the assertion:
```
assertion failed: client_side.cc:1705: "request->http_ver.protocol == http_ver.protocol"
```
