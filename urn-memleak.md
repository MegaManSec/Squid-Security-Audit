# RFC 2141 / 2169 (URN) Response Parsing Memory Leak
RFC 2141 concerns the so-called 'Uniform Resource Name' (URN) scheme.  While similar to standard URLs, URNs are concerned about identifiers within a specific namespace for long-term availability. One example is `urn:ietf:rfc:2648`, which corresponds to The IETF's RFC 2648.

RFC 2169 concerns the "Trivial Convention for using HTTP in URN Resolution", and can be thought of as an interface between HTTP and URN; effectively, a proxy between the two protocols.

Squid is able to attempt to retrieve URN responses using the method outlined in RFC 2169, which, in layman terms, simply means making an HTTP request to `/uri-res/N2L?<service>?<uri>` of an URN resolver running the well-known (at the time) [`N2L`](https://github.com/MegaManSec/N2L) script. Squid then attempts to parse the response from `N2L` and return an HTML page to the client.

## The Issue
When Squid parses the response from an URN resolver (which it does on *default* configurations), it uses the function `urnParseReply`.  The 'URN resolver' in this case can be an arbitrary host, and is simply extracted from the urn requested (i.e. `urn:example.com:s:s` will connect to `http://example.com/uri-res/N2L`).
This function allocates enough memory on the heap to hold the response:
```
static url_entry *
urnParseReply(const char *inbuf, const HttpRequestMethod& m)
{
    char *buf = xstrdup(inbuf);
…
}
```
at a maximum of 4096-bytes per allocation.
However, this memory is never freed. It is therefore trivial to send requests (and thus replies) to Squid over and over, until the host's memory is exhausted.

As explained by Amos (Squid's core developer), it is possible for a single *request* to cause memory exhaustion:
>Squid will receive the initial URN request, and attempt to resolve the request from the remote server by request the API `/uri-res/N2L`. A malicious server may respond to the `/uri-res/N2L` request with a 302-redirect to another URN, which will cause Squid to first leak memory in the urnParseReply function, before following the redirect. This redirection can then be repeated by the malicious server for infinite many times, eventually exhausting all the memory on the Squid instance's host.

The simple fix is to simply free the memory at the end of this function.

This was assigned [CVE-2021-28651](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28651).
