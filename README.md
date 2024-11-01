# Squid Caching Proxy Security Audit: 55 vulnerabilities and 35 0days

In February 2021, I started looking for vulnerabilities in forward-proxies, and found various issues in Squid. Some more information about what's here can be found on my blog: [https://joshua.hu/squid-security-audit-35-0days-45-exploits](https://joshua.hu/squid-security-audit-35-0days-45-exploits)

Explanations and reproducers for each of the vulnerabilities are documented in each of the markdown files. IDs are assigned where possible, however since the majority of these remain unfixed, there are no identifiers.

The Squid Team have been helpful and supportive during the process of reporting these issues. However, they are effectively understaffed, and simply do not have the resources to fix the discovered issues. Hammering them with demands to fix the issues won't get far.

With any system or project, it is important to reguarly review solutions used in your stack to determine whether they are still appropriate. If you are running Squid in an environment which may suffer from any of these issues, then it is up to you to reassess whether Squid is the right solution for your system.


|  Vulnerability | CVE | GHSA |
|--|--|--|
| [Buffer Overflow in Digest Authentication](digest-overflow.md)| [CVE-2023-46847](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-46847) | [GHSA-phqj-m8gv-cq4g](https://github.com/squid-cache/squid/security/advisories/GHSA-phqj-m8gv-cq4g) |
| [Use-After-Free in TRACE Requests](trace-uaf.md)| [CVE-2023-49288](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-49288) | [GHSA-rj5h-46j6-q2g5](https://github.com/squid-cache/squid/security/advisories/GHSA-rj5h-46j6-q2g5) |
| [Partial Content Parsing Use-After-Free](range-uaf.md)|[CVE-2021-31807](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31807) | [GHSA-pxwq-f3qr-w2xf](https://github.com/squid-cache/squid/security/advisories/GHSA-pxwq-f3qr-w2xf) |
| [X-Forwarded-For Stack Overflow](xff-stackoverflow.md)| [CVE-2023-50269](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-50269) | [GHSA-wgq4-4cfg-c4x3](https://github.com/squid-cache/squid/security/advisories/GHSA-wgq4-4cfg-c4x3) |
| [Chunked Encoding Stack Overflow](chunked-stackoverflow.md)| [CVE-2024-25111](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-25111) | [GHSA-72c2-c3wm-8qxc](https://github.com/squid-cache/squid/security/advisories/GHSA-72c2-c3wm-8qxc) |
| [Use-After-Free in Cache Manager Errors](cache-uaf.md)| [CVE-2024-23638](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-23638) | [GHSA-j49p-553x-48rx](https://github.com/squid-cache/squid/security/advisories/GHSA-j49p-553x-48rx) |
| [Cache Poisoning by Large Stored Response Headers (With Bonus XSS)](cache-headers.md)| [CVE-2023-5824](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-5824)| [GHSA-543m-w2m2-g255](https://github.com/squid-cache/squid/security/advisories/GHSA-543m-w2m2-g255) |
| [Memory Leak in CacheManager URI Parsing](cachemanager-memleak.md)|[CVE-2021-28652](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28652) | [GHSA-m47m-9hvw-7447](https://github.com/squid-cache/squid/security/advisories/GHSA-m47m-9hvw-7447) |
| [RFC 2141 / 2169 (URN) Response Parsing Memory Leak](urn-memleak.md)| [CVE-2021-28651](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28651) | [GHSA-ch36-9jhx-phm4](https://github.com/squid-cache/squid/security/advisories/GHSA-ch36-9jhx-phm4) |
| [Memory Leak in HTTP Response Parsing](response-memleaks.md)| [CVE-2024-25617](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-25617) | [GHSA-h5x6-w8mv-xfpr](https://github.com/squid-cache/squid/security/advisories/GHSA-h5x6-w8mv-xfpr) |
| [Memory Leak in ESI Error Processing](esi-memleak.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
| [1-Byte Buffer OverRead in RFC 1123 date/time Handling](datetime-overflow.md)| [CVE-2023-49285](https://cve.mitre.org/cvename.cgi?name=CVE-2023-49285) | [GHSA-8w9r-p88v-mmx9](https://github.com/squid-cache/squid/security/advisories/GHSA-8w9r-p88v-mmx9) |
| [Null Pointer Dereference in Gopher Response Handling](gopher-nullpointer.md)| [CVE-2023-46728](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-46728) | [GHSA-cg5h-v6vc-w33f](https://github.com/squid-cache/squid/security/advisories/GHSA-cg5h-v6vc-w33f) |
| [One-Byte Buffer OverRead  in HTTP Request Header Parsing](garbage-overflow.md)| | |
| [strlen(NULL) Crash Using Digest Authentication](digest-strlen-null.md)| | [GHSA-254c-93q9-cp53](https://github.com/squid-cache/squid/security/advisories/GHSA-254c-93q9-cp53) |
| [Assertion in ESI Header Handling](esi-assert-header.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
| [Integer Overflow in Range Header](range-assert-int.md)|[CVE-2021-31808](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31808) | [GHSA-pxwq-f3qr-w2xf](https://github.com/squid-cache/squid/security/advisories/GHSA-pxwq-f3qr-w2xf)|
| [Gopher Assertion Crash](gopher-assert-entry.md)| | |
| [Whois Assertion Crash](whois-assert-entry.md)| | |
| [Assertion in Gopher Response Handling](gopher-assert.md)| [CVE-2021-46784](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-46784) | [GHSA-f5cp-6rh3-284w](https://github.com/squid-cache/squid/security/advisories/GHSA-f5cp-6rh3-284w) |
| [RFC 2141 / 2169 (URN) Assertion Crash](urn-assert.md)| | |
| [Vary: Other HTTP Response Assertion Crash](vary-other-assert.md)|[CVE-2021-28662](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28662) | [GHSA-jjq6-mh2h-g39h](https://github.com/squid-cache/squid/security/advisories/GHSA-jjq6-mh2h-g39h) |
| [Assertion in Negotiate/NTLM Authentication Using Pipeline Prefetching](ntlm-negotiate-assert.md)| | |
| [Assertion on IPv6 Host Requests with --disable-ipv6](ipv6-assert.md)| | |
| [Assertion Crash on Unexpected "HTTP/1.1 100 Continue" Response Header](100-continue-entry-assert.md)| | |
| [Pipeline Prefetch Assertion With Double 'Expect:100-continue' Request Headers](expect-100-assert.md)| | |
| [Pipeline Prefetch Assertion With Invalid Headers](expect-100-invalid-headers-assert.md)| | |
| [Assertion Crash in Deferred Requests](defer-assert.md)| | |
| [Assertion in Digest Authentication](digest-assert.md)| | |
| [FTP URI Assertion](ftp-assert.md)| [CVE-2023-46848](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-46848) | [GHSA-2g3c-pg7q-g59w](https://github.com/squid-cache/squid/security/advisories/GHSA-2g3c-pg7q-g59w) |
| [FTP Authentication Crash](ftp-fatal.md)| | |
| [Unsatisfiable Range Requests Assertion](range-assert.md)|[CVE-2021-31806](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-31806) | [GHSA-pxwq-f3qr-w2xf](https://github.com/squid-cache/squid/security/advisories/GHSA-pxwq-f3qr-w2xf) |
| [Crash in Content-Range Response Header Logic](range-fatal.md)|[CVE-2021-33620](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-33620) |[GHSA-572g-rvwr-6c7f](https://github.com/squid-cache/squid/security/advisories/GHSA-572g-rvwr-6c7f) |
| [Assertion Crash In HTTP Response Headers Handling](response-assertion.md)| | |
| [Implicit Assertion in Stream Handling](stream-assert.md)| | |
| [Buffer UnderRead in SSL CN Parsing](ssl-bufferunderread.md)| [CVE-2023-46724](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-46724)| [GHSA-73m6-jm96-c6r3](https://github.com/squid-cache/squid/security/advisories/GHSA-73m6-jm96-c6r3) |
| [Use-After-Free in ESI 'Try' (and 'Choose') Processing ](esi-uaf-crash.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
| [Use-After-Free in ESI Expression Evaluation ](esi-uaf.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
| [Buffer Underflow in ESI ](esi-underflow.md)| [CVE-2024-37894](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-37894) | [GHSA-wgvf-q977-9xjg](https://github.com/squid-cache/squid/security/advisories/GHSA-wgvf-q977-9xjg)
| [Assertion in Squid "Helper" Process Creator](ipc-assert.md)| [CVE-2023-49286](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-49286) | [GHSA-xggx-9329-3c27](https://github.com/squid-cache/squid/security/advisories/GHSA-xggx-9329-3c27)|
| [Assertion Due to 0 ESI 'when' Checking ](esi-when-assert-0.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
| [Assertion Using ESI's When Directive ](esi-when-assert-1.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
| [Assertion in ESI Variable Assignment (String)](esi-assignassert-2.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
| [Assertion in ESI Variable Assignment](esi-assignassert.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
| [Null Pointer Dereference In ESI's esi:include and esi:when ](esi-nullpointer.md)| [CVE-2024-45802](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2024-45802)| [GHSA-f975-v7qw-q7hj](https://github.com/squid-cache/squid/security/advisories/GHSA-f975-v7qw-q7hj)|
