
# Assertion in Negotiate/NTLM Authentication Using Pipeline Prefetching
HTTP/1.1 (RFC 2616 / 7231) brought forth one of the largest latency-enhancing features that HTTP was lacking beforehand: pipeling. Pipelining is where one TCP connection from a client to a server could be used for _multiple requests_ at the same time. This meant that instead of taking the time to negotiate a TCP connection, send an HTTP request -- wait for its response -- then send the next request, a client could simply send multiple requests in one burst. More information about pipeling can be found [here](https://developer.mozilla.org/ar/docs/Web/HTTP/Connection_management_in_HTTP_1.x). 

While pipelining concerns the client requests, *pipeline prefetching* concerns the actions that the server may do when it receives a pipelined request. When a proxy server receives a pipelined request, it generally fetches each request one-by-one,  and then responds to the client with all the responses packed together. However, with pipeline prefetching, a proxy server can instead parse every request which has been pipelined, and asynchronously request each page, in order to speed up the process of obtaining each of the upstream responses, in order to pack them together and send them back to the client.
Squid supports pipeline prefetching with the `pipeline_prefetch` configuration. Documentation states:
```
HTTP clients may send a pipeline of 1+N requests to Squid using asingle connection, without waiting for Squid to respond to the firstof those requests.
This option limits the number of concurrentrequests Squid will try to handle in parallel.
If set to N, Squid will try to receive and process up to 1+N requests on the sameconnection concurrently.
```

## The Issue
Squid's documentation states that ``WARNING: pipelining breaks NTLM and Negotiate/Kerberos authentication.``
However, it does not state that any attempt to use the NTLM or the Negotiate authentication methods with pipelining will cause Squid to crash.

When Squid checks the response from the NTLM and/or Negotiate helpers for whether a login is successful or not, it uses the function `Auth::Ntlm::UserRequest::HandleReply` (or `Auth::Negotiate::UserRequest::HandleReply`). Among other things, it does the following:
```
HandleReply(void *data, const Helper::Reply &reply)
{  
    Auth::StateData *r = static_cast<Auth::StateData *>(data);
    
    Auth::UserRequest::Pointer auth_user_request = r->auth_user_request;
    
    Auth::Ntlm::UserRequest *lm_request = dynamic_cast<Auth::Ntlm::UserRequest *>(auth_user_request.getRaw());
    assert(lm_request != NULL);
    assert(lm_request->waiting);

    lm_request->waiting = 0;
```
Here, we can see that the reference-counted user request is defined as `lm_request`. An assertion ensures that this request is waiting for a response from the helper (`assert(lm_request->waiting)`), and if it was not waiting for a response, there would be no reason to be inside this function.
As soon as that initial check is passed, `lm_request->waiting` is set to false, because it is not expected to return to this function -- one request should be accompanied by one helper response, after all.

However, if a pipelined request is received, this request is re-used, and thus, `HandleReply` is called a second time with the same request. As such, the assertion of `lm_request->waiting` being true fails, and Squid crashes.

This is extremely easy to trigger, and is possible both in forwarding proxy mode and reverse proxy mode (however the `Proxy-Authorization` header changes to `Authorization` in the reverse-proxy setup). Two examples are given.
In this case, the website `http://google.com/` requires authentication:
```
GET http://google.com/ HTTP/1.1
Proxy-Authorization: NTLM

GET http://google.com/ HTTP/1.1
Proxy-Authorization: NTLM
```

In another case, the nearly always authentication-required cache manager page is used:
```
GET cache_object://mycache.example.com/info HTTP/1.1
Proxy-Authorization: NTLM

GET cache_object://mycache.example.com/info HTTP/1.1
Proxy-Authorization: NTLM
```
