# 1-Byte Buffer OverRead in RFC 1123 date/time Handling
In order to parse dates in HTTP headers (such as `Warning`, or `Expires`), Squid uses a simple parser written in C, located in `lib/rfc1123.c`, which parses dates/times which conform to RFC 1123. For example: `Sun, 18 Apr 2021 18:40:49 GMT`.

## The Issue
Squid passes dates which are assumed to be at least partially well-formed to the function `make_month`:
```
make_month(const char *s)
{
    int i;
    char month[3];

    month[0] = xtoupper(*s);
    month[1] = xtolower(*(s + 1));
    month[2] = xtolower(*(s + 2));

    for (i = 0; i < 12; i++)
        if (!strncmp(month_names[i], month, 3))
            return i;
    return -1;
}
```

However, it is possible to trick Squid into passing a two-byte buffer to `make_month`, which will make cause a one-byte buffer overread: in `xtolower(*(s + 2));`. This is triggerable via both request headers and response headers:

```
==3041920==ERROR: AddressSanitizer: global-buffer-overflow on address 0x000002dd16e0 at pc 0x000001cac7c8 bp 0x7ffffffea590 sp 0x7ffffffea588

READ of size 1 at 0x000002dd16e0 thread T0
    #0 0x1cac7c7 in parse_date_elements lib/rfc1123.c:88:17
    #1 0x1cac7c7 in parse_date lib/rfc1123.c:153:10
    #2 0x1cac7c7 in parse_rfc1123 lib/rfc1123.c:165:10
    #3 0x8ff0fc in HttpHeader::getTime(Http::HdrType) const src/HttpHeader.cc:1236:17
    #4 0xbe84a1 in clientInterpretRequestHeaders(ClientHttpRequest*) src/client_side_request.cc:1036:29
    #5 0xbe84a1 in ClientHttpRequest::doCallouts() src/client_side_request.cc:1805:13
    #6 0xbfc077 in ClientRequestContext::clientAccessCheckDone(Acl::Answer const&) src/client_side_request.cc:819:11
    #7 0xbf4d52 in ClientRequestContext::clientAccessCheck2() src/client_side_request.cc:724:9
    #8 0xbe69fe in ClientHttpRequest::doCallouts() src/client_side_request.cc:1787:29
    #9 0xbfc077 in ClientRequestContext::clientAccessCheckDone(Acl::Answer const&) src/client_side_request.cc:819:11
    #10 0xbfa044 in clientAccessCheckDoneWrapper(Acl::Answer, void*) src/client_side_request.cc:736:21
    #11 0x114f682 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #12 0xbf3bea in ClientRequestContext::clientAccessCheck() src/client_side_request.cc
    #13 0xbe62e2 in ClientHttpRequest::doCallouts() src/client_side_request.cc:1759:29
    #14 0xbf1567 in ClientRequestContext::hostHeaderVerify() src/client_side_request.cc
    #15 0xbe6065 in ClientHttpRequest::doCallouts() src/client_side_request.cc:1752:29
    #16 0xb50278 in clientProcessRequest(ConnStateData*, RefCount<Http::One::RequestParser> const&, Http::Stream*) src/client_side.cc:1802:11
    #17 0x12f13cb in Http::One::Server::processParsedRequest(RefCount<Http::Stream>&) src/servers/Http1Server.cc:292:5
    #18 0xb04229 in ConnStateData::clientParseRequests() src/client_side.cc:1994:13
```

An example of a request header causing this bug:
```
If-Modified-Since: S000 00 0000000000: 00000000000000000000000000000000000000000 c
```

An example of a response header:
```
Warning: " "W000 000000000000 0000000000000000 000000000:0000000000000000 y"
```
which will be passed to `make_month` as `make_month("y")`.

While this is fairly harmless, a crash may occur if the overread attempts to access protected memory.
