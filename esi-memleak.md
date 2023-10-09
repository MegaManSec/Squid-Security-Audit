# Memory Leak in ESI Error Processing
Squid is able to process Edge Side Includes (ESI) pages when it acts as a reverse proxy. By default, this processing is enabled by default, and can be 'triggered' by any page whose response can be manipulated to return a single response header, and content.

ESI is an xml-based markup language which, in layman terms, can be used to cache *specific parts* of a single webpage, thus allowing for the caching of static content on an otherwise dynamically created page.

ESI processing (especially in Squid) is fragile, and must be scripted with care. Errors can (and are expected to) occur which are gracefully dealt with. When an error occurs in ESI processing, Squid returns a generic error page to the requesting client.

## The Issue
When some sort of error must be communicated to the client, such as a requested page not being alive for example, Squid prepares a standard HTML response to be sent to the requester. This is done using the `clientBuildError` function. This function does the following:
```
ErrorState *
clientBuildError(err_type page_id, Http::StatusCode status, char const *url,
                 const ConnStateData *conn, HttpRequest *request, const AccessLogEntry::Pointer &al) {
    const auto err = new ErrorState(page_id, status, request, al);
    err->src_addr = conn && conn->clientConnection ? conn->clientConnection->remote : Ip::Address::NoAddr();
    if (url)
        err->url = xstrdup(url);
    return err;
}
```
As we can see, quite a lot of memory is allocated on the heap.

When and `ErrorState` is created, it is normally passed to the function ``errorAppendEntry``, which does the following:
```
void
errorAppendEntry(StoreEntry * entry, ErrorState * err)
{
…
    if (entry->store_status != STORE_PENDING) {
        delete err;
        return;
    }
    …
    entry->storeErrorResponse(err->BuildHttpReply());
    delete err;
}
```

However, when an error occurs in ESI processing, ``errorAppendEntry`` is not called, and the `ErrorState` is not cleaned up with memory freed:
```
void
ESIContext::fail ()
{
 ….
    const auto err = clientBuildError(errorpage, errorstatus, nullptr, http->getConn(), http->request, http->al);
    err->err_msg = errormessage;
    errormessage = NULL;
    rep = err->BuildHttpReply();
    // XXX: Leaking err!
    assert (rep->body.hasContent());
    size_t errorprogress = rep->body.contentSize();
    send ();
 }

```

Funnily enough, there is even a comment mentioning that the `err` is leaked :smile:

It is trivial for this error page to be generated over-and-over until the memory of the system is exhausted.
