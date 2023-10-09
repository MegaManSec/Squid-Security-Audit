# Cache Poisoning by Large Stored Response Headers (With Bonus XSS)
Squid offers many [storage types](http://www.squid-cache.org/Doc/config/cache_dir/) to save cached pages/files. These include standard in-memory caching, as well as other methods which all eventually deal with on-disk storage; the latter of which are used for keeping caches in the case of Squid shutting down.

Cached responses are saved in so-called 'client stores', which are saved in either memory or on-disk. When they must be retrieved, they are read, and the cached response is sent to the requesting client.

## The Issue
Squid usually handles response headers using 4096-byte buffering. This means that any response which has headers of greater than 4096-bytes long, is actually multiple buffers appended to each other.

However, when a response is saved to the disk and then subsequently retrieved, Squid does not appropriately handle saved responses which pertain to multiple 4096-byte long buffers. In other words, Squid is unable to handle large response headers that have been saved to the disk.

The main issue is in the function ``store_client::readBody``, which handles parsing the response from the disk:
```
    if (copyInto.offset == 0 && len > 0 && rep && rep->sline.status() == Http::scNone) {
        /* Our structure ! */
        if (!entry->mem_obj->adjustableBaseReply().parseCharBuf(copyInto.data, headersEnd(copyInto.data, len))) {
            debugs(90, DBG_CRITICAL, "Could not parse headers from on disk object");
        } else {
            parsed_header = 1;
        }
    }
```
The critical line here is:
```
        if (!entry->mem_obj->adjustableBaseReply().parseCharBuf(copyInto.data, headersEnd(copyInto.data, len))) {
```
The function `headersEnd` is used to determine where in the request the headers end and the body, if applicable, begins. If `headersEnd` is passed an invalid request, it returns zero as an error.
While STORE_OK (i.e. known 'good' and 'well-formed') responses are normally passed to headersEnd without any other checks, in the case of stored responses, data is truncated to 4096-bytes, and thus most stored responses with headers greater than 4096-bytes will be corrupted. 
When the function `parseCharBuf` is called with a length of zero (which will happen in this case), an error will occur, and the headers will not be marked as parsed. As this failure propagates 'down the line', Squid eventually realises there is a problem, and returns an HTTP "500 internal service error" page to the requesting client. This error page's body contains the full saved response (body headers and body), completely unfiltered.

This is problematic for multiple reasons:
- It is trivial to create a page which responds with large response headers which will be cached and stored to the disk. However, the impact of this is fairly low, because the only cached object you will be poisoning is of a page you already control.
- It is *sometimes* trivial to 'trick' a page which we *do not* control into responding with large response headers. One such method could be the nearly omnipresent Host Header Injection bug which is possible nearly everywhere (which is a non-issue by itself). Other methods exist, too (for example: response splitting).
- The error response which Squid returns to the client on this error occurring is unescaped, and contains what was in the response's headers, in the body. This means that any response which contained headers which, say, contained HTML, would be parsed as HTML. In a situation where we can control the response headers of a web page, it is then trivial to cause JavaScript execution in the browser, because those *headers* now become part of the *body*.

For example, let's say we have a 'target' which returns the (response) headers:
```
HTTP/1.1 200 OK
X-Requested-User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.90 Safari/537.36 OPR/75.0.3969.149
```
It is clear that it is possible to, then, change our user-agent to an arbitrary value for it to be placed within the response *headers*, such as:
```
HTTP/1.1 200 OK
X-Requested-User-Agent: Mozilla/5.0 <script>alert(1);</script>AAAAAAAA……
```
which would normally not be an issue, as response headers are never parsed as a request (assuming no issues with crlf).

However, if those `A` characters were to cause the response headers to be greater than 4096-bytes, once the response is cached and saved onto the disk, the response will be unable to be retrieved. Subsequent visits to the same page will cause the following *full* (i.e. header and body) response:
```
HTTP/1.1 500 Internal Server Error
Age: <Something>
Date: <Something>
Cache-Status: hit;detail=match

HTTP/1.1 200 OK
X-Requested-User-Agent: Mozilla/5.0 <script>alert(1);</script>AAAAAAAA……
```

As such, the original response *headers* are placed into the response *body*, which can then be executed by the browser.

Of course, such 'attacks' are completely theoretical and are only considered for entertainment purposes. The main issue here is that any cached object with large headers will become inaccessible (which can only be recovered by clearing the cache) after they are written to the on-file cache.
