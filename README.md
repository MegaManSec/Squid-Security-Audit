# Squid Caching Proxy Security Audit

In February 2021, I started looking for vulnerabilities in forward-proxies, and found various issues in Squid.

Explanations and reproducers for each of the vulnerabilities are documented in each of the markdown files. IDs are assigned where possible, however since the majority of these remain unfixed, there are no identifiers.

|  Vulnerability| ID |
|--|--|
| [Stack Buffer Overflow in Digest Authentication](digest-overflow.md)| |
| [Use-After-Free in TRACE Requests](trace-uaf.md)| |
| [Partial Content Parsing Use-After-Free](range-uaf.md)|[CVE-2021-31807](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31807) |
| [X-Forwarded-For Stack Overflow](xff-stackoverflow.md)| |
| [Chunked Encoding Stack Overflow](chunked-stackoverflow.md)| |
| [Use-After-Free in Cache Manager Errors](cache-uaf.md)| |
| [Cache Poisoning by Large Stored Response Headers (With Bonus XSS)](cache-headers.md)| |
| [Memory Leak in CacheManager URI Parsing](cachemanager-memleak.md)|[CVE-2021-28652](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28652) |
| [RFC 2141 / 2169 (URN) Response Parsing Memory Leak](urn-memleak.md)| [CVE-2021-28651](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28651)|
| [Memory Leak in HTTP Response Parsing](response-memleaks.md)| |
| [Memory Leak in ESI Error Processing](esi-memleak.md)| |
| [1-Byte Buffer OverRead in RFC 1123 date/time Handling](datetime-overflow.md)| |
| [Null Pointer Dereference in Gopher Response Handling](gopher-nullpointer.md)| [GHSA-cg5h-v6vc-w33f](https://github.com/squid-cache/squid/security/advisories/GHSA-cg5h-v6vc-w33f) |
| [One-Byte Buffer OverRead  in HTTP Request Header Parsing](garbage-overflow.md)| |
| [strlen(NULL) Crash Using Digest Authentication](digest-strlen-null.md)| |
| [Assertion in ESI Header Handling](esi-assert-header.md)| |
| [Integer Overflow in Range Header](range-assert-int.md)|[CVE-2021-31808](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31808) |
| [Gopher Assertion Crash](gopher-assert-entry.md)| |
| [Whois Assertion Crash](whois-assert-entry.md)| |
| [Assertion in Gopher Response Handling](gopher-assert.md)| |
| [RFC 2141 / 2169 (URN) Assertion Crash](urn-assert.md)| |
| [Vary: Other HTTP Response Assertion Crash](vary-other-assert.md)|[CVE-2021-28662](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28662) |
| [Assertion in Negotiate/NTLM Authentication Using Pipeline Prefetching](ntlm-negotiate-assert.md)| |
| [Assertion on IPv6 Host Requests with --disable-ipv6](ipv6-assert.md)| |
| [Assertion Crash on Unexpected "HTTP/1.1 100 Continue" Response Header](100-continue-entry-assert.md)| |
| [Pipeline Prefetch Assertion With Double 'Expect:100-continue' Request Headers](expect-100-assert.md)| |
| [Pipeline Prefetch Assertion With Invalid Headers](expect-100-invalid-headers-assert.md)| |
| [Assertion Crash in Deferred Requests](defer-assert.md)| |
| [Assertion in Digest Authentication](digest-assert.md)| |
| [FTP URI Assertion](ftp-assert.md)| |
| [FTP Authentication Crash](ftp-fatal.md)| |
| [Unsatisfiable Range Requests Assertion](range-assert.md)|[CVE-2021-31806](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31806) |
| [Crash in Content-Range Response Header Logic](range-fatal.md)|[CVE-2021-33620](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-33620) |
| [Assertion Crash In HTTP Response Headers Handling](response-assertion.md)| |
| [Implicit Assertion in Stream Handling](stream-assert.md)| |
| [Buffer UnderRead in SSL CN Parsing](ssl-bufferunderread.md)| |
| [Use-After-Free in ESI 'Try' (and 'Choose') Processing ](esi-uaf-crash.md)| |
| [Use-After-Free in ESI Expression Evaluation ](esi-uaf.md)| |
| [Buffer Underflow in ESI ](esi-underflow.md)| |
| [Assertion in Squid "Helper" Process Creator](ipc-assert.md)| |
| [Assertion Due to 0 ESI 'when' Checking ](esi-when-assert-0.md)| |
| [Assertion Using ESI's When Directive ](esi-when-assert-1.md)| |
| [Assertion in ESI Variable Assignment (String)](esi-assignassert-2.md)| |
| [Assertion in ESI Variable Assignment](esi-assignassert.md)| |
| [Null Pointer Dereference In ESI's esi:include and esi:when ](esi-nullpointer.md)| |
