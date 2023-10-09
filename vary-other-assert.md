# Vary: Other HTTP Response Assertion Crash
The HTTP/1.1 `Vary` header is used to determine which request headers should be matched against future requests, in order to determine whether a cached response should be used or a fresh one should be requested from the origin server.

The `Vary` header is parsed as a comma-delimited list and matched against 'known' headers for future use.

## The Issue
When Squid parses a response with a `Vary` header, it uses the function `assembleVaryKey` to loop through each separated header and assemble an array of headers which it will use for caching purposes in the future. This function does the following:
```
String hdr(request.header.getByName(name));
```
which eventually calls : `HttpHeader::hasNamed`, which is defined as:
```
bool
HttpHeader::hasNamed(const char *name, unsigned int namelen, String *result) const
{   
    Http::HdrType id;
    HttpHeaderPos pos = HttpHeaderInitPos;
    HttpHeaderEntry *e;

    assert(name);

    /* First try the quick path */
    id = Http::HeaderLookupTable.lookup(name,namelen).id;
    if (id != Http::HdrType::BAD_HDR) {
        if (getByIdIfPresent(id, result))
        ….
```
The issue here is that there is no restriction to what header is sent to `Http::HeaderLookupTable.lookup`, or `getByIdIfPresent`. If `Http::HeaderLookupTable.lookup` is passed an `Other` header, it will successfully return the type `Http::HdrType::OTHER`. This will then be passed to `HttpHeader::getByIdIfPresent`.
The issue here is that `HttpHeader::getByIdIfPresent` calls a function, `HttpHeader::has`, is defined as follows:
```
/* test if a field is present */
int
HttpHeader::has(Http::HdrType id) const
{  
    assert(any_registered_header(id));
    debugs(55, 9, this << " lookup for " << id);
    return CBIT_TEST(mask, id);
} 
```
Subsequently, `any_registered_header` is defined as:
```
/// match any registered header type (headers squid knows how to handle),
///  thus excluding OTHER and BAD
inline bool
any_registered_header (const Http::HdrType id)
{   
    return (id >= Http::HdrType::ACCEPT && id < Http::HdrType::OTHER);
}
```

The issue is clear: if the `Vary` response header contains `Other`,  this assertion will trigger because prior checks are are only for `Http::HdrType::BAD_HDR`, not `Http::HdrType::OTHER`, but the assertion is for both.

The exact response which will trigger this is:
```
Vary: Other:
```

I do not know why the final colon is required, but it is.


This was assigned [CVE-2021-28662](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28662).
