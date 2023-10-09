

# Assertion Crash on Unexpected "HTTP/1.1 100 Continue" Response Header

Note: This issue is similar to that of `RFC 2141 / 2169 (URN) Assertion Crash`, and thus a detailed description is not given. However, this issue pertains to how Squid handles receiving "HTTP/1.1 100 Continue" response headers when the client is not expecting such a header.

## The Issue
When a client opens a connection with Squid and subsequently closes it, its so-called 'store entry' is marked as 'pending' (STORE_PENDING) and 'unsuccessful' (ENTRY_ABORTED) respectively. If an attempt is subsequently made to write to that store entry after it has been set to ENTRY_ABORTED, Squid will crash with an assertion, as it generally means something is 'wrong' (since it should be STORE_PENDING).

Due to Squid marking a store as ENTRY_ABORTED upon a client disconnecting, it is possible to trick Squid into requesting a Gopher page, then quickly disconnecting, resulting in a write to a store which has not been marked as ENTRY_ABORTED.

In theory, this issue could affect HTTP too, however there are protections in place within the HTTP end-points to stop the processing of asynchronous replies when an entry has been aborted (such as due to a client closing the connection before the reply has been read) For example in `HttpStateData::readReply` and `HttpStateData::processReplyBody`, there is a check before the entry writing:
```
    if (EBIT_TEST(entry->flags, ENTRY_ABORTED)) {
        abortTransaction("store entry aborted while reading reply");
        return;
    }
```
The issue is that despite this protection being in place for `HttpStateData::readReply`, which stops `HttpStateData::processReply` from being called (which attempts to write to the entry), it is possible to bypass the `HttpStateData::readReply` function altogether. This happens if a client requests a website, and the website responses with a `"HTTP/1.1 100 Continue"` header:
```
void
HttpStateData::proceedAfter1xx()
{
 [...]

    debugs(11, 2, "continuing with " << payloadSeen << " bytes in buffer after 1xx");
    CallJobHere(11, 3, this, HttpStateData, HttpStateData::processReply);
}

```
We can see here that after a call to `HttpStateData::processReply` is scheduled directly, it is then possible for the client to disconnect, thus setting the store entry to STORE_ABORTED. Since there are no checks within the `HttpStateData::processReply` function for an aborted entry directly (since it is assumed that this function is only called via `HttpStateData::readReply`), an assertion will happen.

It is thus possible to cause an assertion using the following proof of concept.
A client requests and disconnects quickly:
```
printf "POST http://localhost/ HTTP/1.0\r\n\r\n" | timeout 0.1 nc localhost 2222 2>&1 > /dev/null
```
And a server replies with:
```
HTTP/1.0 100 00
Date: F000 00 Mar 0000 00:00:4000000
```

Squid then asserts:
```
2021/04/08 03:37:21.950| assertion failed: store.cc:834: "store_status == STORE_PENDING"
    current master transaction: master254
```
