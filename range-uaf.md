# Partial Content Parsing Use-After-Free
The HTTP request header 'Range' is used when a client wishes to partially download (also known as *byte serving*) a file from a compliant HTTP server (RFC 7233 states that Range header handling is optional in HTTP/1.1). Examples for usage include resuming downloads, segmented updates, and video playback (being able to skip to the middle of a video without downloading the whole video first).

Squid is able to handle both multi-part ranges (such as, bytes 0-50 AND 100-150), and single part ranges (bytes 0-50 ONLY), and their subsequent '206 Partial Content' responses.

## The Issue
When a client requests a multi-part range, checks are incorrectly delayed for whether they are valid and well-formed until after the response has been received.
This means that once a response is received, invalid ranges are deleted:
```
    for (iterator pos (begin()); pos != end(); ++pos) {
        if ((*pos)->canonize(clen))
            copy.push_back (*pos);
        else
            delete (*pos);
    }

```
Here, `pos` is a pointer to a range, as specified by the client in the request.

However, despite being deleted, they are once again used in ``HttpHdrRangeIter::updateSpec``:
```
void
HttpHdrRangeIter::updateSpec()
{ 
    assert (debt_size == 0);
    assert (valid);

    if (pos != end) {
        debt(currentSpec()->length);
    }
}

void HttpHdrRangeIter::debt(int64_t newDebt)
{   
    debugs(64, 3, "HttpHdrRangeIter::debt: was " << debt_size << " now " << newDebt);
    debt_size = newDebt;
}

```

Assuming no malice in the UaF, an assert will happen right afterwards in `Http::Stream::canPackMoreRanges`:
``assert(!http->range_iter.debt() == !http->range_iter.currentSpec());``.

The following request can be made to Squid to cause this bug:
```
GET https://i.imgur.com/p9CXk3M.png HTTP/1.0
Range:bytes=0-0,-4,-0
```


This bug was assigned [CVE-2021-31807](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31807).

