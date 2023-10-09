
# strlen(NULL) Crash Using Digest Authentication

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
When using Digest authentication, it is possible to set the algorithm for hashing, for example, to MD5, by adding `algorithm="md5-sess"` to the `Authorization` header. This is parsed by Squid and used.

Various checks are made to determine which sort of authentication is used, because different RFCs dictate different directives/forms used in Digest authentication. For example, in the `Auth::Digest::Config::decode` function, which parses the `Authorization` header, checks for RFC 2617-specific directives are made:
```
    /* 2617 requirements, indicated by qop */
    if (digest_request->qop) {

        /* check the qop is what we expected. */
        if (strcmp(digest_request->qop, QOP_AUTH) != 0) {
            /* we received a qop option we didn't send */
            debugs(29, 2, "Invalid qop option received");
            rv = authDigestLogUsername(username, digest_request, aRequestRealm);
            safe_free(username);
            return rv;
        }

        /* check cnonce */
        if (!digest_request->cnonce || digest_request->cnonce[0] == '\0') {
            debugs(29, 2, "Missing cnonce field");
            rv = authDigestLogUsername(username, digest_request, aRequestRealm);
            safe_free(username);
            return rv;
        }

        /* check nc */
        if (strlen(digest_request->nc) != 8 || strspn(digest_request->nc, "0123456789abcdefABCDEF") != 8) {
            debugs(29, 2, "invalid nonce count");
            rv = authDigestLogUsername(username, digest_request, aRequestRealm);
            safe_free(username);
            return rv;
        }
    } else {
        /* cnonce and nc both require qop */
        if (digest_request->cnonce || digest_request->nc[0] != '\0') {
            debugs(29, 2, "missing qop!");
            rv = authDigestLogUsername(username, digest_request, aRequestRealm);
            safe_free(username);
            return rv;
        }
    }

```
As we can see, the header is completely rejected if it contains a `qop` directive, but does not have a `cnonce` value, for example.
If there is no `qop` value but there _is_ a `cnonce` value (or there is an `nc` value), the header is also rejected.

However, if there is _no_ `qop` value, no `cnonce` value, and a `nc` valid, this check passes.

As the authentication process continues, the `Auth::Digest::UserRequest::authenticate`  function is called, which does the following:
```
Auth::Digest::UserRequest::authenticate(HttpRequest * request, ConnStateData *, Http::HdrType)
{   
[...]

    DigestCalcHA1(digest_request->algorithm, NULL, NULL, NULL,
                  authenticateDigestNonceNonceHex(digest_request->nonce),
                  digest_request->cnonce,


```
Basically, this will calculate a _digest_ using the parameters `digest_request->cnonce`, `authenticateDigestNonceNonceHex(digest_request->nonce)`, and `digest_request->algorithm`.
`DigestCalcHA1` does the following:
```
calculate H(A1) as per spec */
void
DigestCalcHA1(
    const char *pszAlg,
    const char *pszUserName,
    const char *pszRealm,
    const char *pszPassword,
    const char *pszNonce,
    const char *pszCNonce,
    HASH HA1,
    HASHHEX SessionKey
)
{
    SquidMD5_CTX Md5Ctx;

    if (pszUserName) {
        SquidMD5Init(&Md5Ctx);
        SquidMD5Update(&Md5Ctx, pszUserName, strlen(pszUserName));
        SquidMD5Update(&Md5Ctx, ":", 1);
        SquidMD5Update(&Md5Ctx, pszRealm, strlen(pszRealm));
        SquidMD5Update(&Md5Ctx, ":", 1);
        SquidMD5Update(&Md5Ctx, pszPassword, strlen(pszPassword));
        SquidMD5Final((unsigned char *) HA1, &Md5Ctx);
    }
    if (strcasecmp(pszAlg, "md5-sess") == 0) {
        HASHHEX HA1Hex;
        CvtHex(HA1, HA1Hex);    /* RFC2617 errata */
        SquidMD5Init(&Md5Ctx);
        SquidMD5Update(&Md5Ctx, HA1Hex, HASHHEXLEN);
        SquidMD5Update(&Md5Ctx, ":", 1);
        SquidMD5Update(&Md5Ctx, pszNonce, strlen(pszNonce));
        SquidMD5Update(&Md5Ctx, ":", 1);
        SquidMD5Update(&Md5Ctx, pszCNonce, strlen(pszCNonce));
        SquidMD5Final((unsigned char *) HA1, &Md5Ctx);
    }
    CvtHex(HA1, SessionKey);
}
```
The call within `Auth::Digest::UserRequest::authenticate` to `DigestCalcHA1` is with NULL for `pszUserName`, so the first block of code is skipped. However, the user-supplied `digest_request->algorithm` is passed as the `pszAlg` parameter, which means the second block of code can be executed if the original request header contains `algorithm="md5-sess"`.

We have already established that original checks may be passed if there is "_no_ `qop` value, no `cnonce` value, and a `nc`". Thus, if no `cnonce` value is passed, the following line will cause Squid to crash:
```
SquidMD5Update(&Md5Ctx, pszCNonce, strlen(pszCNonce));
```
because `pszCNonce` is NULL. `strlen(NULL)` causes a segmentation fault on most systems.

This is easy to reproduce, for example with the following header:
```
Proxy-Authorization: Digest username="0",realm="0",nonce="15930256138a2072903bd84d980f485c",uri="0",response="00000000000000000000000000000000", algorithm="md5-sess"
```
Here we can see that the required `algorithm="md5-sess"` is set, and there are no `qop`, `nc`, or `cnonce` values.
