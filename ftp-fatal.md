# FTP Authentication Crash
FTP (File Transfer Protocol) is a 50-year-old protocol which is used for the transfer of files across networks. Unlike HTTP, FTP is stateful, meaning that (generally) a connection is held open for multiple file transfers rather than a single file. Likewise, unlike HTTP which generally requires sending a single request, FTP uses multiple commands and responses which must be dealt with before any file transfer actually happens.
There are some interesting features of FTP which go beyond the scope of this post, however the tunneling or proxying of FTP through HTTP is an interesting topic worth reading more about, especially how to *efficiently* deal with each command and response.

Squid is able to act as both a [native FTP relay](https://wiki.squid-cache.org/Features/FtpRelay), and a proxy. In the latter case, bridging an HTTP-user to an FTP-server.  In the former case, a client can send an *FTP* command to Squid, which squid interprets (and actually converts to HTTP, then re-converts to FTP), and subsequently sends it to the destination FTP server. The reply is eventually sent back to the client.

## The Issue
When Squid is acting as an FTP *proxy* (gateway), which is does by default on the HTTP-port which is enabled, the client's request headers are parsed for any authentication which may be used. This is started in the ``Ftp::Gateway::start`` function, which calls `Ftp::Gateway::checkAuth`:
```
int
Ftp::Gateway::checkAuth(const HttpHeader * req_hdr)
{   
    /* default username */
    xstrncpy(user, "anonymous", MAX_URL);
    
    const auto auth(req_hdr->getAuthToken(Http::HdrType::AUTHORIZATION, "Basic"));
    if (!auth.isEmpty()) {
        flags.authenticated = 1;
        loginParser(auth, false);
    }
…
    /* name is missing. that's fatal. */
    if (!user[0])
        fatal("FTP login parsing destroyed username info");
    if (password[0])
        return 1;
…
    return 0;           /* different username */
}
```
As we can see, this function first fills the `user` buffer with the text `anonymous`, and then attempts to parse the `Authorization` request header for a `Basic`-type authentication header, and sends it to the `loginParser` function. The `HttpHeader::getAuthToken` function can more-or-less be described as a simple HTTP header parser which will base64-decode the contents of a header of type `Authorization: Basic […]`. 

`Ftp::Gateway::loginParser` is then concerned about the (hopefully) `username:password` combination which has been base64-decoded from the header, and fills the char buffer `user` and `pass`. The assumption here is that it should be impossible for `user` to be empty, since it has been initialized as `anonymous`, and `loginParser` may only *change* the contents of this buffer, assuming a successfully parsed and base64-decoded `Authorization` header.

However, this assumption is incorrect, because, for example, the base64-encoded string `AG00` can be decoded as three characters: `\00`, `\109`, `\52` (in octal); or rather, in ASCII: `\000m4` where `\000` is a NULL character. Once this `AG00` field is decoded, it is placed into the `user` variable. From here, the condition `if (!user[0])` becomes true, and `fatalf()`, which causes squid to (gracefully) exit.

A very easy reproducer is given:
```
GET ftp://ftp.hp.com/ HTTP/1.1
Authorization: Basic AG00
```
which will cause Squid to crash:
```
2021/04/19 14:33:36.467| FATAL: FTP login parsing destroyed username info
current master transaction: master54
2021/04/19 14:33:36.467| Squid Cache (Version 6.0.0-VCS): Terminated abnormally.
current master transaction: master54
```

This bug is slightly interesting because, in fact, an *empty* (rather than *no*)  username is a *valid* username when it comes to FTP:
From RFC 1738:
```
   Note that an empty user name or password is different than no user
   name or password; there is no way to specify a password without
   specifying a user name. E.g., <URL:ftp://@host.com/> has an empty
   user name and no password, <URL:ftp://host.com/> has no user name,
   while <URL:ftp://foo:@host.com/> has a user name of "foo" and an
   empty password.
```
But, hey, c'est la vie.
