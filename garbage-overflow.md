# One-Byte Buffer OverRead  in HTTP Request Header Parsing
HTTP/1.1 (RFC 7230) recommends expecting an unserviceable CRLF before a request is made:
```
In the interest of robustness, a server that is expecting to receive
and parse a request-line SHOULD ignore at least one empty line (CRLF)
received prior to the request-line.
```
This means that a **valid** HTTP/1.1 request *may* send a frivolous CRLF before its first line such as: `\r\nGET / HTTP/1.1\r\n\r\n`. In this case, the first `\r\n` should be discarded.

Squid conforms to this *SHOULD* parameter of the RFC.

## The Issue
Upon receiving an HTTP/1.1 request, Squid calls the function `skipGarbageLines`, which is defined as the following:
```
void
Http::One::RequestParser::skipGarbageLines()
{
    if (Config.onoff.relaxed_header_parser) {
        if (Config.onoff.relaxed_header_parser < 0 && (buf_[0] == '\r' || buf_[0] == '\n'))
            debugs(74, DBG_IMPORTANT, "WARNING: Invalid HTTP Request: " <<
                   "CRLF bytes received ahead of request-line. " <<
                   "Ignored due to relaxed_header_parser.");
        // Be tolerant of prefix empty lines
        // ie any series of either \n or \r\n with no other characters and no repeated \r
        while (!buf_.isEmpty() && (buf_[0] == '\n' || (buf_[0] == '\r' && buf_[1] == '\n'))) {
            buf_.consume(1);
        }
    }
}

```
The issue here is in the following lines:
```
        while (!buf_.isEmpty() && (buf_[0] == '\n' || (buf_[0] == '\r' && buf_[1] == '\n'))) {
            buf_.consume(1);
        }
```
While there is a check that `buf_` is not empty, it does not check that there is a second byte allocated. If `buf_[0] == '\r'` is true,`buf_[1] == '\n'` will cause a one-byte buffer overread. Note that `buf_` is not a NUL-terminated buffer. If this condition (incorrectly) returns true, a `consume` (which is akin to `memmove`) will occur.

The worst case scenario here is a simple crash. 

Due to Squid's mempooling, it is difficult to trigger this bug to actually cause a memory violation (rather than a *memory pool* violation), but not impossible. For example, the following request can be made:
```
GET http://000000000000000000000000000\n\r
```
which is detected by ASAN:
```
==351608==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x629000004200 at pc 0x000000b47d50 bp 0x7ffffffecc40 sp 0x7ffffffecc38
READ of size 1 at 0x629000004200 thread T0
    #0 0xb47d4f in SBuf::operator[](unsigned int) const src/../src/sbuf/SBuf.h:231:68
    #1 0x12b3d8e in Http::One::RequestParser::skipGarbageLines() src/http/one/RequestParser.cc:48:75
    #2 0x12b297d in Http::One::RequestParser::doParse(SBuf const&) src/http/one/RequestParser.cc:358:9
    #3 0x12b1fe2 in Http::One::RequestParser::parse(SBuf const&) src/http/one/RequestParser.cc:340:25
    #4 0xcd06b7 in ConnStateData::parseHttpRequest(RefCount<Http::One::RequestParser> const&) src/client_side.cc:1337:35
    #5 0x12679b9 in Http::One::Server::parseOneRequest() src/servers/Http1Server.cc:88:29
    #6 0xcabad6 in ConnStateData::clientParseRequests() src/client_side.cc:1984:43
```
