# Assertion Crash In HTTP Response Headers Handling
When Squid receives a request, it passes the request upstream. It then must handle the response appropriately and pass it back to the client.

Squid has multiple ways to handle character buffers, which it uses interchangeably. Notably, one of its methods to handle character buffers, is known as a `String` type. This type holds a maximum of 65534-bytes, reserving one byte for the null-character. The `String` type is used for *some* parsing of HTTP headers. `String`types are used in these cases because there is an overall assumption that the response headers will be smaller than 65535-bytes. If a `String` type is assigned a length of more than 65534-bytes (or is appended to which makes the object such a large length), an assertion will occur.

Some response headers must be parsed in special ways, which can sometimes mean the resulting `String` is either expanded or shrunk.

In some cases, Squid may inappropriately handle response headers, and cause an assertion.

## The Issue
When Squid parses a `Warning` response header, it gets sent to the `HttpReply::removeStaleWarningValues` function to be parsed. This function loops around each comma and, effectively, appends what has already been dealt with in the loop with `, `.
For example, the response header `Warning: 0,0,0,0,0` will be converted to `Warning: 0, 0, 0, 0, 0`.

The issue here is that initial checks (*which, if they fail, will cause a different vulnerability identified: a memory leak*) for the length of the `String` type may be successful when the header is first parsed,  but this expansion may cause the string to grow over the limit of 65535-bytes, thus causing an assertion.

The growth rate of the expansion given in this example is: ![Growth Rate](https://render.githubusercontent.com/render/math?math=%5Cfrac%7B%28-2%20%2b%203%20n%29%7D%7B%28-1%20%2b%202%20n%29%7D) (*where n is the number of '0' characters*), which, for large values of `n`, is 1.5. This means that (for large values) the contents of the `Warning` header is expanded 1.5x from what it was to begin with. Furthermore, this means that a response `Warning` header of length `65535*2/3 = 43690` can cause an assertion (corresponding to `n = 21846` 0-characters (why yes, I *did* study Mathematics at University).

It is thus trivial to set up a remote server which responds with the HTTP headers "Warning 0,0,0,0....[repeat ~44000 times]" to cause an assertion when a Squid user accesses the page. This will work on a default installation because the initial header length checks will pass:
```
2021/04/19 01:19:57.207| assertion failed: String.cc:172: "canGrowBy(len)"
    current master transaction: master57
```

## Other Issues
This issue also affects the `Cache-Control` header in a very similar manner. When a `Cache-Control` response header is received, it is parsed by the `HttpHdrCc::parse` function:
```
    while (strListGetItem(&str, ',', &item, &ilen, &pos)) {
        const HttpHdrCcType type = ccLookupTable.lookup(SBuf(item,nlen));
        switch (type) {
        case HttpHdrCcType::CC_OTHER:
            if (other.size())
                other.append(", ");

            other.append(item, ilen);
            break;  
    }
```
An assertion is thus possible in the example given in the `Warning` case, but replacing `Warning` with `Cache-Control`.
