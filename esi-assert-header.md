
# Assertion in ESI Header Handling
Squid is able to process Edge Side Includes (ESI) pages when it acts as a reverse proxy. By default, this processing is enabled by default, and can be 'triggered' by any page whose response can be manipulated to return a single response header, and content.

ESI is an xml-based markup language which, in layman terms, can be used to cache *_specific parts_* of a single web page, thus allowing for the caching of static content on an otherwise dynamically created page.

## The Issue
ESIs are able to make use of certain HTTP headers, such as through boolean operators, checking of certain attributes, and Cookie parsing.

However, Squid's ESI handling incorrectly assumes that headers will always be less than 65535-bytes long, and uses the `String`-type to parse them in some places. This means that, for example, if `request_header_max_size` is set to greater than 65535-bytes, Squid will assert when parsing these headers in ESI's handling, because the `String`-type has a hard-coded maximum of 65535-bytes.

There are multiple places this is an issue in Squid's ESI handling. Notably:
```
String S (state.header().getList (Http::HdrType::ACCEPT_LANGUAGE));
String strVary (rep->header.getList (Http::HdrType::VARY));
```
This means that a rogue client can send an extremely long `Vary` or `Accept-Language` header to Squid when it is in reverse-proxy mode, and if it is using ESIs (or can be tricked into thinking it is using ESIs), it will crash with an assertion.
