# Crash in Content-Range Response Header Logic
The `Content-Range` header is an HTTP/1.1 response header which is used to inform the client which bytes of a specific page (or file) that is transferred corresponds to. This is generally used for partial-content retrieval and distribution. For example, a valid header may look like: `Content-Range: bytes 9-100/200`, which means that the content being sent pertains to bytes 9 to 100, of a total file being of size 200. The asterisk(`*`) character has a special meaning in this context: it means unknown. For example, `Content-Range: bytes 10-100/*` means that the bytes 10 to 100 are being sent, but it is unknown how large the full file is.

Squid is able to handle `Content-Range` response headers, and combines much of the logic for this header with that of the `Range` *request* header (as one would expect). It can use this header for more complicated cases of caching, for example, the combining of multiple ranges into one.

## The Issue
Squid does not properly handle unsatisfiable `Content-Range` headers, and fails to reject them correctly, if a response is of type `HTTP/1.1 206 Partial Content`.

The issue is slightly complicated, but it can be exemplified in the function `Client::haveParsedReplyHeaders`:
```
void
Client::haveParsedReplyHeaders()
{   
    const bool partial = theFinalReply->contentRange();
    currentOffset = partial ? theFinalReply->contentRange()->spec.offset : 0;
}
```
In this case, if a `Content-Range` header is received, the `currentOffset` variable is set to the `spec.offset` variable.
This variable was previously set by the `httpHdrContRangeParseInit` function, which effectively does the following:
```
    if (strncasecmp(str, "bytes ", 6))
        return 0;

    str += 6;

    /* split */
    if (!(p = strchr(str, '/')))
        return 0;
    if (*str == '*')
        range->spec.offset = range->spec.length = range_spec_unknown;
```
Note: `range_spec_unknown` is equal to `-1`.
This means that if a `Content-Range` header begins with `*`, the offset of the set to `-1`. Of course, it is not possible to access content of length `-1`, but due to invalid (further) checks, this is kept as `-1` and assumed to be a valid offset.

This negative value for `currentOffset` propagates through Squid, until the 'actual' content of the response is parsed and written to memory. Eventually, Squid will crash due to the invalid offset being detected
```
2021/04/19 16:04:08.283| mem_hdr::write: writeBuffer: [95,96)
    current master transaction: master53
2021/04/19 16:04:08.283| mem_hdr::debugDump: lowest offset: 0 highest offset + 1: 96.
    current master transaction: master53
2021/04/19 16:04:08.283| mem_hdr::debugDump: Current available data is: [0,96) - .
    current master transaction: master53
2021/04/19 16:04:08.283| FATAL: Attempt to overwrite already in-memory data. Preceding this there should be a mem_hdr::write output that lists the attempted write, and the currently present data. Please get a 'backtrace full' from this error - using the generated core, and file a bug report with the squid developers including the last 10 lines of cache.log and the backtrace.
```

It is simple to reproduce this crash, with the following response:
```
HTTP/1.1 206 OK
Content-Length: 1
Content-Range: bytes */1
Connection: close

1
```

This bug was assigned CVE-2021-33620.
