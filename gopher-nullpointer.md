# Null Pointer Dereference in Gopher Response Handling
Gopher, other than being a glorified [*rat*](https://upload.wikimedia.org/wikipedia/commons/thumb/c/cb/Pocket-Gopher_Ano-Nuevo-SP.jpg/440px-Pocket-Gopher_Ano-Nuevo-SP.jpg), is an antiquated protocol described in RFC 1436.

Squid is able to act as a proxy server between HTTP and the Gopher protocol, and can query and retrieve contents from Gopher servers, before sending it to the client in a standard HTML document.

## The Issue
When Squid receives a request to a gopher server, it attempts to connect to the server and parse the response. The response is then placed into a standard HTML response for the user's browsing. This conversion happens in the function `gopherToHTML`.
This function does many things, but most of the important code for this bug is as follows:
```
static void
gopherToHTML(GopherStateData * gopherState, char *inbuf, int len)
{
…
    char *port = NULL;
…
          switch (gopherState->conversion) {
…
          case GopherStateData::HTML_INDEX_RESULT:
          case GopherStateData::HTML_DIR: {
…
                if (host) {
                    *host = '\0';
                    ++host;
                    port = strchr(host, TAB);

                    if (port) {
                        char *junk;
                        port[0] = ':';
                        junk = strchr(host, TAB);

                        if (junk)
                            *junk++ = 0;    /* Chop port */
                        else {
                            junk = strchr(host, '\r');

                            if (junk)
                                *junk++ = 0;    /* Chop port */
                            else {
                                junk = strchr(host, '\n');

                                if (junk)
                                    *junk++ = 0;    /* Chop port */
                            }
                        }

                        if ((port[1] == '0') && (!port[2]))
                            port[0] = 0;    /* 0 means none */
                    }
…
                    if ((gtype == GOPHER_TELNET) || (gtype == GOPHER_3270)) {
                        if (strlen(escaped_selector) != 0)
                            snprintf(tmpbuf, TEMP_BUF_SIZE, "<IMG border=\"0\" SRC=\"%s\"> <A HREF=\"telnet://%s@%s%s%s/\">%s</A>\n",
                                     icon_url, escaped_selector, rfc1738_escape_part(host),
                                     *port ? ":" : "", port, html_quote(name));
                        else
                            snprintf(tmpbuf, TEMP_BUF_SIZE, "<IMG border=\"0\" SRC=\"%s\"> <A HREF=\"telnet://%s%s%s/\">%s</A>\n",
                                     icon_url, rfc1738_escape_part(host), *port ? ":" : "",
                                     port, html_quote(name));
```
In layman terms, this means that if the client requests a Gopher page which is a directory or index, all of this code will be executed.
However, if the response portrays itself as GOPHER_TELNET, and no port is parsed in the response, a null pointer dereference of `*port` will happen in the calls to `snprintf()`.

This is extremely easy to trigger. In order for a server to be identified as GOPHER_TELNET, it simply needs to respond with the number `8` as the first character.  In order for the function to believe it is requesting a directory, the request must end in `/`:
```
gopher_request_parse(const HttpRequest * req, char *type_id, char *request)
{
    ::Parser::Tokenizer tok(req->url.path());

    if (request)
        *request = 0;

    tok.skip('/'); // ignore failures? path could be ab-empty

    if (tok.atEnd()) {
        *type_id = GOPHER_DIRECTORY;
        return;
    }
}

gopherSendComplete(const Comm::ConnectionPointer &conn, char *, size_t size, Comm::Flag errflag, int xerrno, void *data) {
    switch (gopherState->type_id) {

    case GOPHER_DIRECTORY:
        /* we got to convert it first */
        gopherState->conversion = GopherStateData::HTML_DIR;
        gopherState->HTML_header_added = 0;
        break;
  }
```

Thus, to trigger this bug, a client can send the request:
```
GET gopher://example.com:1234/\r\n\r\n
```
while a server located at `example.com` on port `1234` simply responds with `8\t\t\n`.

A Null pointer dereference will cause a crash on most systems.
