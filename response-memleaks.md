# Memory Leak in HTTP Response Parsing
When Squid receives a request, it passes the request upstream. It then must handle the response appropriately and pass it back to the client. 
Squid has multiple ways to handle character buffers, which it uses interchangeably. Notably, one of its methods to handle character buffers, is known as a `String` type. This type holds a maximum of 65534-bytes, reserving one byte for the null-character. The `String` type is used for *some* parsing of HTTP headers.

`String` is used in these cases because there is an overall assumption that the response headers will be smaller than 65535-bytes. However, this is not always the case, as the config option `reply_header_max_size` may allow for higher values.

In some cases, Squid may inappropriately handle reply headers of extremely long length, and leak large amounts of memory.

## The Issue
When Squid receives a response from an upstream server, the function `HttpStateData::processReplyHeader` handles the parsing and processing of the response headers, and prepares a reply to be sent to the client. This function does, for the purpose of this bug, the following:
```
void
HttpStateData::processReplyHeader()
{
    /* Attempt to parse the first line; this will define where the protocol, status, reason-phrase and header begin */
    {
        if (hp == NULL)
            hp = new Http1::ResponseParser;

        bool parsedOk = hp->parse(inBuf);
        if (!parsedOk) {
            return;
        }
    }

    HttpReply *newrep = new HttpReply;
    newrep->sline.set(hp->messageProtocol(), hp->messageStatus() /* , hp->reasonPhrase() */);

    // parse headers
    if (!newrep->parseHeader(*hp)) {
        newrep->sline.set(hp->messageProtocol(), Http::scInvalidHeader);
        debugs(11, 2, "error parsing response headers mime block");
    }
}
```

When the headers are parsed using ``parseHeader``, a special function is eventually called in order to evaluate the `Cache-Control` header, called `HttpHeader::getCc`, which does the following:
```
HttpHeader::getCc() const {
    String s;
    getList(Http::HdrType::CACHE_CONTROL, &s);
```
`getList` turns a header into a list (with comma delimiters).
However, getList will cause an exception if the `Cache-Control` header is greater than 65535-bytes long, as this is the maximum length of a `String` type. This is easily achievable is the config option `reply_header_max_size` is set to greater than 65535.

When this happens, the memory allocated in `HttpStateData::processReplyHeader` -- specifically from `HttpReply *newrep = new HttpReply` -- will be leaked.

It is thus trivial to cause memory exhaustion by continuously requesting a page which presents a `Cache-Control` header of more than 65535-bytes long.

## Other Issues

Similar issues exist for the response headers `Warning`, `Via`, `Transfer-Encoding`, `Connection`, `Etag`, `Content-MD5`, `X-Forwarded-For`, `X-Accelerator-Vary`, `Proxy-Connection`, and in reverse proxy mode (using ESI), `Cookie`, `Accept-Language`, and `Vary`.
There may be other cases where this causes problems and this list should not be assumed to be exhaustive.
