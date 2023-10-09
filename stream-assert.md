
# Implicit Assertion in Stream Handling

## The Issue
The `clientReplyContext::pushStreamData` function is used for passing data from the client to the server, or vice-versa.
This function calls `clientStreamCallback` as so:
```
clientStreamCallback((clientStreamNode*)http->client_stream.head->data, http, NULL,localTempBuffer);
```
This function then calls the function `clientStreamCallback` as follows:
```
next->callback(next, http, rep, replyBuffer);
```
with the same order of the variables (i.e. `next` = `(clientStreamNode*)http->client_stream.head->data`,  `http` = `http`, etc.).
`callback` is an alias for the `clientSocketRecipient` function, which has the following code:
```
        http->getConn()->handleReply(rep, receivedData);
```

We can see that from the first call to `clientStreamCallback`, `rep` will be NULL.

The issue here is that `handleReply` asserts when `rep` is NULL. When this happens, Squid will crash:
```
#0  __GI_raise (sig=sig@entry=6) at ../sysdeps/unix/sysv/linux/raise.c:50
#1  0x00007ffff7482859 in __GI_abort () at abort.c:79
#2  0x0000555555e23043 in xassert (msg=0x5555565a06dc "rep", file=0x5555565a047d "Http1Server.cc", line=330) at debug.cc:624
#3  0x00005555561256bd in Http::One::Server::handleReply (this=0x55555ccf1428, rep=0x0, receivedData=...) at Http1Server.cc:330
#4  0x0000555555db36a4 in clientSocketRecipient (node=0x55555d789e98, http=0x55555c3ca0f8, rep=0x0, receivedData=...) at client_side.cc:870
#5  0x0000555555da54e6 in clientStreamCallback (thisObject=0x55555d789e08, http=0x55555c3ca0f8, rep=0x0, replyBuffer=...) at clientStream.cc:159
#6  0x0000555555e011a7 in clientReplyContext::pushStreamData (this=0x55555ce542d8, result=..., source=0x55555c5664d0 '0' <repeats 110 times>, "\r\nf\r\n T0 05 Nov 1990 00:00:00 GMT\r\n00n00000t00n: 00\r\nContent-Type: 00\r\nETag: 00\r\nLast-Modi"...) at client_side_reply.cc:1858
#7  0x0000555555e043bd in clientReplyContext::sendMoreData (this=0x55555ce542d8, result=...) at client_side_reply.cc:2144
#8  0x0000555555e0083c in clientReplyContext::SendMoreData (data=0x55555ce542d8, result=...) at client_side_reply.cc:1804
#9  0x0000555555f49f12 in store_client::callback (this=0x55555bb73aa8, sz=115, error=false) at store_client.cc:178
#10 0x0000555555f4d08b in store_client::scheduleMemRead (this=0x55555bb73aa8) at store_client.cc:474
#11 0x0000555555f4c71d in store_client::scheduleRead (this=0x55555bb73aa8) at store_client.cc:440
#12 0x0000555555f4c13a in store_client::doCopy (this=0x55555bb73aa8, anEntry=0x555559a1d7d0) at store_client.cc:401
#13 0x0000555555f4b6db in storeClientCopy2 (e=0x555559a1d7d0, sc=0x55555bb73aa8) at store_client.cc:355
#14 0x0000555555f4ad2b in store_client::copy (this=0x55555bb73aa8, anEntry=0x555559a1d7d0, copyRequest=..., callback_fn=0x555555e007fa <clientReplyContext::SendMoreData(void*, StoreIOBuffer)>, data=0x55555ce542d8) at store_client.cc:278
#15 0x0000555555f4a6be in storeClientCopy (sc=0x55555bb73aa8, e=0x555559a1d7d0, copyInto=..., callback=0x555555e007fa <clientReplyContext::SendMoreData(void*, StoreIOBuffer)>, data=0x55555ce542d8) at store_client.cc:230
#16 0x0000555555dfffff in clientGetMoreData (aNode=0x55555d789e08, http=0x55555c3ca0f8) at client_side_reply.cc:1719
#17 0x0000555555da58ee in clientStreamRead (thisObject=0x55555d789e98, http=0x55555c3ca0f8, readBuffer=...) at clientStream.cc:182
#18 0x0000555556145adc in Http::Stream::pullData (this=0x55555c5664b0) at Stream.cc:127
#19 0x00005555561452cd in Http::Stream::writeComplete (this=0x55555c5664b0, size=0) at Stream.cc:85
#20 0x0000555555db6861 in ConnStateData::afterClientWrite (this=0x55555ccf1428, size=0) at client_side.cc:1057
#21 0x000055555612e2ad in Server::clientWriteDone (this=0x55555ccf1428, io=...) at Server.cc:208
#22 0x000055555612fae7 in CommCbMemFunT<Server, CommIoCbParams>::doDial (this=0x55555cc9e128) at ../../src/CommCalls.h:205
#23 0x000055555612fc71 in JobDialer<Server>::dial (this=0x55555cc9e128, call=...) at ../../src/base/AsyncJobCalls.h:175
#24 0x000055555612f8e5 in AsyncCallT<CommCbMemFunT<Server, CommIoCbParams> >::fire (this=0x55555cc9e0f0) at ../../src/base/AsyncCall.h:145
#25 0x000055555616779f in AsyncCall::make (this=0x55555cc9e0f0) at AsyncCall.cc:44
#26 0x00005555561696cc in AsyncCallQueue::fireNext (this=0x555557270080) at AsyncCallQueue.cc:60
#27 0x0000555556169062 in AsyncCallQueue::fire (this=0x555557270080) at AsyncCallQueue.cc:43
#28 0x0000555555c31205 in EventLoop::dispatchCalls (this=0x7fffffffdde0) at EventLoop.cc:144
#29 0x0000555555c31003 in EventLoop::runOnce (this=0x7fffffffdde0) at EventLoop.cc:121
#30 0x0000555555c30c95 in EventLoop::run (this=0x7fffffffdde0) at EventLoop.cc:83
#31 0x0000555555ecbc0d in SquidMain (argc=6, argv=0x7fffffffdfc8) at main.cc:1716
#32 0x0000555555ec9426 in SquidMainSafe (argc=6, argv=0x7fffffffdfc8) at main.cc:1403
#33 0x0000555555ec93ad in main (argc=6, argv=0x7fffffffdfc8) at main.cc:1391
```

No more information is given in this document; it can happen randomly.
