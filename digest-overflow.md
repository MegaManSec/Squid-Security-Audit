# Stack Buffer Overflow in Digest Authentication


Digest Authentication is just one of many HTTP authentication methods available within browsers. Unlike the 'Basic' authentication method (which you may be familiar with in the format of accessing a website in the form http://username:password@website.com/), which sends a username and password over the internet in cleartext, Digest Authentication uses so-called 'nonces', which are used to hash the username and password before they are sent to the server for authentication.

Simplified, RFC 2617 defines authentication as followed:
```
Hash1=MD5(username:realm:password)
Hash2=MD5(method:digestURI)
response=MD5(Hash1:nonce:nc:cnonce:qop:Hash2)
```
with the nonce, nonceCount, cnonce, and qop, each referring to the server-provided nonce, nonce-count (incremented by the client), client-provided nonce, and the "quality of protection", respectively.

Digest Authentication is a so-called challenge-response authentication method, meaning that a browser must first attempt to access a protected realm (page) in order to receive the challenge by the server, which contains the server-provided nonce. An example HTTP *response* within the context of Squid is given:
```
HTTP/1.1 407 Proxy Authentication Required
Server: squid/6.0.0-VCS
Mime-Version: 1.0
Date: Sat, 17 Apr 2021 21:53:19 GMT
Content-Type: text/html;charset=utf-8
Content-Length: 3502
X-Squid-Error: ERR_CACHE_ACCESS_DENIED 0
Vary: Accept-Language
Content-Language: en
Proxy-Authenticate: Digest realm="My Super Secret Page!", nonce="cab92d09fff10dfb679a279447eba301", qop="auth", stale=false
Connection: keep-alive
```

After this initial response is received when trying to access the authenticated page, the browser then uses the provided nonce, and sends the request:

```
GET /SecretPage HTTP/1.1
Host: website.com
Proxy-Authorization: Digest username="Username", realm="My Super Secret Page!", nonce="cab92d09fff10dfb679a279447eba301", uri="/SecretPage", cnonce="ZjU3NDlkMWY1NmZmMTFmNDIyNzJlMDM4Yzk3OTY2MDE=", nc=00000001, qop=auth, response="43194ca186e1bc93ecb7e846725bbee1"
User-Agent: curl/7.68.0
Accept: */*
Proxy-Connection: Keep-Alive
```

Within the context of Squid, this authentication can be used for both proxy authentication (in forwarding mode), and standard WWW authentication (in reverse proxy mode).

## The Issue

When parsing the second request from the browser for the provided Digest values, Squid loops between each comma in order to extract information from the request line. It extracts, for example, `username`, `realm`, and so on, and dynamically allocates memory on the heap to hold each token.

Squid  assumes that the user-provided nonce-count (*nc*) is a legitimate value of exactly 8-digits long (i.e. 00000000-99999999). This assumption is made because RFC 7616 dictates that: "*for historical reasons, the nc value MUST be exactly 8 hexadecimal digits.*" As such, a 9-character long (the extra one-byte for the null pointer) char buffer is allocated on the stack:
```
char nc[9]; /* = "00000001" */
```

However, a malicious client can pass anything it wants in this token.
Squid's Digest authentication parsing does the following:
```
if (value.size() != 8) {
     debugs(29, 9, "Invalid nc '" << value << "' in '" << temp << "'");
}
xstrncpy(digest_request->nc, value.rawBuf(), value.size() + 1);
debugs(29, 9, "Found noncecount '" << digest_request->nc << "'");
```
Here, `value.rawBuf()` contains what is passed in the request, and `digest_request->nc` is our 9-character buffer. xstrcpy will copy a maximum length of whatever is passed in the request to the 9-byte character buffer, rather than a maximum of 9-bytes, as was probably intended. From here, it is as simple as sending a request with the HTTP header
```
Proxy-Authorization: Digest nc="AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA.."
```
in order to cause a stack buffer overflow. There is no limit to the length of this memory violation, and it is trivial to cause remote code execution.

