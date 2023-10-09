
# Use-After-Free in ESI Expression Evaluation 
Squid is able to process Edge Side Includes (ESI) pages when it acts as a reverse proxy. By default, this processing is enabled by default, and can be 'triggered' by any page whose response can be manipulated to return a single response header, and content.
ESI is an xml-based markup language which, in layman terms, can be used to cache *specific parts* of a single webpage, thus allowing for the caching of static content on an otherwise dynamically created page.

## The Issue
The ESI syntax includes a directive called `<esi:when>`. This directive is used to test for boolean conditions. For example, ``<esi:when test="$(HTTP_USER_AGENT{'os'})=='iPhone'">`` could be used to check whether a request's user agent is identifying itself as an iPhone.

The issue is that the function which actually *tests* the `test` directive in terms of its boolean logic is broken in Squid's ESI system. The `esiSequence::process` function is the entry point for evaluating the `test` directive:
```
esiProcessResult_t
esiSequence::process (int inheritedVarsFlag)
{   
 
 [...]
    if (processedcount == elements.size() || provideIncrementalData) {
        ESISegment::Pointer temp(new ESISegment);
        render (temp);

        if (temp->next.getRaw() || temp->len)
            parent->provideData(temp, this);
        else
            ESISegmentFreeList (temp);
    }

    /* Depends on full parsing before processing */
    if (processedcount == elements.size())
        parent = NULL;

    processing = false;

    return processingResult;
}
```
The issue here is that the `provideData` function frees the memory of `this`. This means that every single line after `parent->provideData(temp, this);` causes numerous (both read and write) use-after-frees.
**Note: This means that *every* usage of <esi:when test> causes these use-after-free bugs***
 
 With even basic directives, such as this 100% valid directive:
```
<esi:when test="0!0">
True?
</esi:when>
```
the following use-after-frees happen:

```
Sequence.cc:298:9: runtime error: member access within address 0x60b000056ba0 which does not point to an object of type 'esiSequence'
0x60b000056ba0: note: object has invalid vptr
 00 00 00 00  1f 00 00 77 00 00 00 00  30 2b 21 00 20 60 00 00  30 2b 21 00 20 60 00 00  38 2b 21 00
              ^~~~~~~~~~~~~~~~~~~~~~~
              invalid vptr
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior Sequence.cc:298:9 in 
=================================================================
==1704956==ERROR: AddressSanitizer: heap-use-after-free on address 0x60b000056bc0 at pc 0x00000228c81d bp 0x7fffffff70d0 sp 0x7fffffff70c8
READ of size 8 at 0x60b000056bc0 thread T0
    #0 0x228c81c in esiSequence::process(int) src/esi/Sequence.cc:298:9
    #1 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #2 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #3 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #4 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #5 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #6 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #7 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #8 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #9 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #10 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #11 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #12 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #13 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #14 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #15 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #16 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #17 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #18 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #19 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #20 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #21 0x93837b in EventLoop::runOnce() src/EventLoop.cc:109:23
    #22 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #23 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #24 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #25 0x10f4819 in main src/main.cc:1391:12
    #26 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #27 0x71dbad in _start (src/squid+0x71dbad)

0x60b000056bc0 is located 32 bytes inside of 112-byte region [0x60b000056ba0,0x60b000056c10)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3084 in free_const compat/xalloc.cc:175:5
    #2 0x232e897 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232e270 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2317e45 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x23123d6 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222ef6f in esiWhen::operator delete(void*) src/esi/Esi.cc:194:5
    #7 0x222a4c4 in esiWhen::~esiWhen() src/esi/Esi.cc:2202:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d1fb in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220217 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1996:13
    #11 0x2282bff in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x228c675 in esiSequence::process(int) src/esi/Sequence.cc:290:21
    #13 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #14 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #15 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #16 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #17 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #18 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #19 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #20 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #21 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #22 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #23 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #24 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #25 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #26 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #27 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #28 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #29 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c2d5f in xmalloc compat/xalloc.cc:114:15
    #2 0x232dd2e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2317c06 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2311f06 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222eef5 in esiWhen::operator new(unsigned long) src/esi/Esi.cc:194:5
    #6 0x21f3645 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1061:19
    #7 0x22b061e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/Sequence.cc:298:9 in esiSequence::process(int)
Shadow bytes around the buggy address:
  0x0c1680002d20: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c1680002d30: fa fa fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c1680002d40: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c1680002d50: fd fd fd fd fd fd fa fa fa fa fa fa fa fa fd fd
  0x0c1680002d60: fd fd fd fd fd fd fd fd fd fd fd fd fa fa fa fa
=>0x0c1680002d70: fa fa fa fa fd fd fd fd[fd]fd fd fd fd fd fd fd
  0x0c1680002d80: fd fd fa fa fa fa fa fa fa fa fd fd fd fd fd fd
  0x0c1680002d90: fd fd fd fd fd fd fd fa fa fa fa fa fa fa fa fa
  0x0c1680002da0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fa fa
  0x0c1680002db0: fa fa fa fa fa fa fd fd fd fd fd fd fd fd fd fd
  0x0c1680002dc0: fd fd fd fd fa fa fa fa fa fa fa fa fd fd fd fd
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
Sequence.cc:298:27: runtime error: member access within address 0x60b000056ba0 which does not point to an object of type 'esiSequence'
0x60b000056ba0: note: object has invalid vptr
 00 00 00 00  1f 00 00 77 00 00 00 00  30 2b 21 00 20 60 00 00  30 2b 21 00 20 60 00 00  38 2b 21 00
              ^~~~~~~~~~~~~~~~~~~~~~~
              invalid vptr
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior Sequence.cc:298:27 in 
=================================================================
==1704956==ERROR: AddressSanitizer: heap-use-after-free on address 0x60b000056bb0 at pc 0x000002230561 bp 0x7fffffff7080 sp 0x7fffffff7078
READ of size 8 at 0x60b000056bb0 thread T0
    #0 0x2230560 in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::size() const /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:919:40
    #1 0x228c911 in esiSequence::process(int) src/esi/Sequence.cc:298:36
    #2 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #3 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #4 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #5 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #6 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #7 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #8 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #9 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #10 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #11 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #12 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #13 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #14 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #15 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #16 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #17 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #18 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #19 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #20 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #21 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #22 0x93837b in EventLoop::runOnce() src/EventLoop.cc:109:23
    #23 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #24 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #25 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #26 0x10f4819 in main src/main.cc:1391:12
    #27 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #28 0x71dbad in _start (src/squid+0x71dbad)

0x60b000056bb0 is located 16 bytes inside of 112-byte region [0x60b000056ba0,0x60b000056c10)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3084 in free_const compat/xalloc.cc:175:5
    #2 0x232e897 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232e270 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2317e45 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x23123d6 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222ef6f in esiWhen::operator delete(void*) src/esi/Esi.cc:194:5
    #7 0x222a4c4 in esiWhen::~esiWhen() src/esi/Esi.cc:2202:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d1fb in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220217 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1996:13
    #11 0x2282bff in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x228c675 in esiSequence::process(int) src/esi/Sequence.cc:290:21
    #13 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #14 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #15 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #16 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #17 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #18 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #19 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #20 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #21 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #22 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #23 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #24 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #25 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #26 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #27 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #28 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #29 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c2d5f in xmalloc compat/xalloc.cc:114:15
    #2 0x232dd2e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2317c06 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2311f06 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222eef5 in esiWhen::operator new(unsigned long) src/esi/Esi.cc:194:5
    #6 0x21f3645 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1061:19
    #7 0x22b061e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:919:40 in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::size() const
Shadow bytes around the buggy address:
  0x0c1680002d20: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c1680002d30: fa fa fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c1680002d40: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c1680002d50: fd fd fd fd fd fd fa fa fa fa fa fa fa fa fd fd
  0x0c1680002d60: fd fd fd fd fd fd fd fd fd fd fd fd fa fa fa fa
=>0x0c1680002d70: fa fa fa fa fd fd[fd]fd fd fd fd fd fd fd fd fd
  0x0c1680002d80: fd fd fa fa fa fa fa fa fa fa fd fd fd fd fd fd
  0x0c1680002d90: fd fd fd fd fd fd fd fa fa fa fa fa fa fa fa fa
  0x0c1680002da0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fa fa
  0x0c1680002db0: fa fa fa fa fa fa fd fd fd fd fd fd fd fd fd fd
  0x0c1680002dc0: fd fd fd fd fa fa fa fa fa fa fa fa fd fd fd fd
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
=================================================================
==1704956==ERROR: AddressSanitizer: heap-use-after-free on address 0x60b000056ba8 at pc 0x000002230642 bp 0x7fffffff7080 sp 0x7fffffff7078
READ of size 8 at 0x60b000056ba8 thread T0
    #0 0x2230641 in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::size() const /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:919:66
    #1 0x228c911 in esiSequence::process(int) src/esi/Sequence.cc:298:36
    #2 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #3 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #4 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #5 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #6 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #7 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #8 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #9 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #10 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #11 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #12 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #13 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #14 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #15 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #16 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #17 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #18 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #19 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #20 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #21 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #22 0x93837b in EventLoop::runOnce() src/EventLoop.cc:109:23
    #23 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #24 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #25 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #26 0x10f4819 in main src/main.cc:1391:12
    #27 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #28 0x71dbad in _start (src/squid+0x71dbad)

0x60b000056ba8 is located 8 bytes inside of 112-byte region [0x60b000056ba0,0x60b000056c10)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3084 in free_const compat/xalloc.cc:175:5
    #2 0x232e897 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232e270 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2317e45 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x23123d6 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222ef6f in esiWhen::operator delete(void*) src/esi/Esi.cc:194:5
    #7 0x222a4c4 in esiWhen::~esiWhen() src/esi/Esi.cc:2202:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d1fb in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220217 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1996:13
    #11 0x2282bff in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x228c675 in esiSequence::process(int) src/esi/Sequence.cc:290:21
    #13 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #14 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #15 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #16 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #17 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #18 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #19 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #20 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #21 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #22 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #23 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #24 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #25 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #26 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #27 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #28 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #29 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c2d5f in xmalloc compat/xalloc.cc:114:15
    #2 0x232dd2e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2317c06 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2311f06 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222eef5 in esiWhen::operator new(unsigned long) src/esi/Esi.cc:194:5
    #6 0x21f3645 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1061:19
    #7 0x22b061e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:919:66 in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::size() const
Shadow bytes around the buggy address:
  0x0c1680002d20: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c1680002d30: fa fa fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c1680002d40: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c1680002d50: fd fd fd fd fd fd fa fa fa fa fa fa fa fa fd fd
  0x0c1680002d60: fd fd fd fd fd fd fd fd fd fd fd fd fa fa fa fa
=>0x0c1680002d70: fa fa fa fa fd[fd]fd fd fd fd fd fd fd fd fd fd
  0x0c1680002d80: fd fd fa fa fa fa fa fa fa fa fd fd fd fd fd fd
  0x0c1680002d90: fd fd fd fd fd fd fd fa fa fa fa fa fa fa fa fa
  0x0c1680002da0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fa fa
  0x0c1680002db0: fa fa fa fa fa fa fd fd fd fd fd fd fd fd fd fd
  0x0c1680002dc0: fd fd fd fd fa fa fa fa fa fa fa fa fd fd fd fd
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
Sequence.cc:299:9: runtime error: member access within address 0x60b000056ba0 which does not point to an object of type 'esiSequence'
0x60b000056ba0: note: object has invalid vptr
 00 00 00 00  1f 00 00 77 00 00 00 00  30 2b 21 00 20 60 00 00  30 2b 21 00 20 60 00 00  38 2b 21 00
              ^~~~~~~~~~~~~~~~~~~~~~~
              invalid vptr
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior Sequence.cc:299:9 in 
=================================================================
==1704956==ERROR: AddressSanitizer: heap-use-after-free on address 0x60b000056bd0 at pc 0x000002236b28 bp 0x7fffffff6ff0 sp 0x7fffffff6fe8
READ of size 8 at 0x60b000056bd0 thread T0
    #0 0x2236b27 in RefCount<esiTreeParent>::dereference(esiTreeParent const*) src/esi/../../src/base/RefCount.h:96:28
    #1 0x222ff3b in RefCount<esiTreeParent>::operator=(RefCount<esiTreeParent>&&) src/esi/../../src/base/RefCount.h:63:13
    #2 0x228ca2a in esiSequence::process(int) src/esi/Sequence.cc:299:16
    #3 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #4 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #5 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #6 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #7 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #8 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #9 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #10 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #11 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #12 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #13 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #14 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #15 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #16 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #17 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #18 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #19 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #20 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #21 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #22 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #23 0x93837b in EventLoop::runOnce() src/EventLoop.cc:109:23
    #24 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #25 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #26 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #27 0x10f4819 in main src/main.cc:1391:12
    #28 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #29 0x71dbad in _start (src/squid+0x71dbad)

0x60b000056bd0 is located 48 bytes inside of 112-byte region [0x60b000056ba0,0x60b000056c10)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3084 in free_const compat/xalloc.cc:175:5
    #2 0x232e897 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232e270 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2317e45 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x23123d6 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222ef6f in esiWhen::operator delete(void*) src/esi/Esi.cc:194:5
    #7 0x222a4c4 in esiWhen::~esiWhen() src/esi/Esi.cc:2202:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d1fb in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220217 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1996:13
    #11 0x2282bff in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x228c675 in esiSequence::process(int) src/esi/Sequence.cc:290:21
    #13 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #14 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #15 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #16 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #17 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #18 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #19 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #20 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #21 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #22 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #23 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #24 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #25 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #26 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #27 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #28 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #29 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c2d5f in xmalloc compat/xalloc.cc:114:15
    #2 0x232dd2e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2317c06 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2311f06 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222eef5 in esiWhen::operator new(unsigned long) src/esi/Esi.cc:194:5
    #6 0x21f3645 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1061:19
    #7 0x22b061e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/../../src/base/RefCount.h:96:28 in RefCount<esiTreeParent>::dereference(esiTreeParent const*)
Shadow bytes around the buggy address:
  0x0c1680002d20: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c1680002d30: fa fa fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c1680002d40: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c1680002d50: fd fd fd fd fd fd fa fa fa fa fa fa fa fa fd fd
  0x0c1680002d60: fd fd fd fd fd fd fd fd fd fd fd fd fa fa fa fa
=>0x0c1680002d70: fa fa fa fa fd fd fd fd fd fd[fd]fd fd fd fd fd
  0x0c1680002d80: fd fd fa fa fa fa fa fa fa fa fd fd fd fd fd fd
  0x0c1680002d90: fd fd fd fd fd fd fd fa fa fa fa fa fa fa fa fa
  0x0c1680002da0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fa fa
  0x0c1680002db0: fa fa fa fa fa fa fd fd fd fd fd fd fd fd fd fd
  0x0c1680002dc0: fd fd fd fd fa fa fa fa fa fa fa fa fd fd fd fd
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
=================================================================
==1704956==ERROR: AddressSanitizer: heap-use-after-free on address 0x60b000056bd0 at pc 0x000002236b55 bp 0x7fffffff6ff0 sp 0x7fffffff6fe8
WRITE of size 8 at 0x60b000056bd0 thread T0
    #0 0x2236b54 in RefCount<esiTreeParent>::dereference(esiTreeParent const*) src/esi/../../src/base/RefCount.h:97:12
    #1 0x222ff3b in RefCount<esiTreeParent>::operator=(RefCount<esiTreeParent>&&) src/esi/../../src/base/RefCount.h:63:13
    #2 0x228ca2a in esiSequence::process(int) src/esi/Sequence.cc:299:16
    #3 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #4 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #5 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #6 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #7 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #8 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #9 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #10 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #11 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #12 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #13 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #14 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #15 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #16 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #17 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #18 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #19 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #20 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #21 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #22 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #23 0x93837b in EventLoop::runOnce() src/EventLoop.cc:109:23
    #24 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #25 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #26 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #27 0x10f4819 in main src/main.cc:1391:12
    #28 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #29 0x71dbad in _start (src/squid+0x71dbad)

0x60b000056bd0 is located 48 bytes inside of 112-byte region [0x60b000056ba0,0x60b000056c10)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3084 in free_const compat/xalloc.cc:175:5
    #2 0x232e897 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232e270 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2317e45 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x23123d6 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222ef6f in esiWhen::operator delete(void*) src/esi/Esi.cc:194:5
    #7 0x222a4c4 in esiWhen::~esiWhen() src/esi/Esi.cc:2202:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d1fb in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220217 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1996:13
    #11 0x2282bff in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x228c675 in esiSequence::process(int) src/esi/Sequence.cc:290:21
    #13 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #14 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #15 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #16 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #17 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #18 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #19 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #20 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #21 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #22 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #23 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #24 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #25 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #26 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #27 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #28 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #29 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c2d5f in xmalloc compat/xalloc.cc:114:15
    #2 0x232dd2e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2317c06 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2311f06 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222eef5 in esiWhen::operator new(unsigned long) src/esi/Esi.cc:194:5
    #6 0x21f3645 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1061:19
    #7 0x22b061e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/../../src/base/RefCount.h:97:12 in RefCount<esiTreeParent>::dereference(esiTreeParent const*)
Shadow bytes around the buggy address:
  0x0c1680002d20: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c1680002d30: fa fa fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c1680002d40: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c1680002d50: fd fd fd fd fd fd fa fa fa fa fa fa fa fa fd fd
  0x0c1680002d60: fd fd fd fd fd fd fd fd fd fd fd fd fa fa fa fa
=>0x0c1680002d70: fa fa fa fa fd fd fd fd fd fd[fd]fd fd fd fd fd
  0x0c1680002d80: fd fd fa fa fa fa fa fa fa fa fd fd fd fd fd fd
  0x0c1680002d90: fd fd fd fd fd fd fd fa fa fa fa fa fa fa fa fa
  0x0c1680002da0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fa fa
  0x0c1680002db0: fa fa fa fa fa fa fd fd fd fd fd fd fd fd fd fd
  0x0c1680002dc0: fd fd fd fd fa fa fa fa fa fa fa fa fd fd fd fd
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
Sequence.cc:303:5: runtime error: member access within address 0x60b000056ba0 which does not point to an object of type 'esiSequence'
0x60b000056ba0: note: object has invalid vptr
 00 00 00 00  1f 00 00 77 00 00 00 00  30 2b 21 00 20 60 00 00  30 2b 21 00 20 60 00 00  38 2b 21 00
              ^~~~~~~~~~~~~~~~~~~~~~~
              invalid vptr
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior Sequence.cc:303:5 in 
=================================================================
==1704956==ERROR: AddressSanitizer: heap-use-after-free on address 0x60b000056bdb at pc 0x00000228d0fb bp 0x7fffffff70d0 sp 0x7fffffff70c8
WRITE of size 1 at 0x60b000056bdb thread T0
    #0 0x228d0fa in esiSequence::process(int) src/esi/Sequence.cc:303:16
    #1 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #2 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #3 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #4 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #5 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #6 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #7 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #8 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #9 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #10 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #11 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #12 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #13 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #14 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #15 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #16 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #17 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #18 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #19 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #20 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #21 0x93837b in EventLoop::runOnce() src/EventLoop.cc:109:23
    #22 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #23 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #24 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #25 0x10f4819 in main src/main.cc:1391:12
    #26 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #27 0x71dbad in _start (src/squid+0x71dbad)

0x60b000056bdb is located 59 bytes inside of 112-byte region [0x60b000056ba0,0x60b000056c10)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3084 in free_const compat/xalloc.cc:175:5
    #2 0x232e897 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232e270 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2317e45 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x23123d6 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222ef6f in esiWhen::operator delete(void*) src/esi/Esi.cc:194:5
    #7 0x222a4c4 in esiWhen::~esiWhen() src/esi/Esi.cc:2202:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d1fb in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220217 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1996:13
    #11 0x2282bff in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x228c675 in esiSequence::process(int) src/esi/Sequence.cc:290:21
    #13 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #14 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #15 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #16 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #17 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #18 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #19 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #20 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #21 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #22 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #23 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #24 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #25 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #26 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #27 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #28 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #29 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c2d5f in xmalloc compat/xalloc.cc:114:15
    #2 0x232dd2e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2317c06 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2311f06 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222eef5 in esiWhen::operator new(unsigned long) src/esi/Esi.cc:194:5
    #6 0x21f3645 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1061:19
    #7 0x22b061e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/Sequence.cc:303:16 in esiSequence::process(int)
Shadow bytes around the buggy address:
  0x0c1680002d20: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c1680002d30: fa fa fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c1680002d40: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c1680002d50: fd fd fd fd fd fd fa fa fa fa fa fa fa fa fd fd
  0x0c1680002d60: fd fd fd fd fd fd fd fd fd fd fd fd fa fa fa fa
=>0x0c1680002d70: fa fa fa fa fd fd fd fd fd fd fd[fd]fd fd fd fd
  0x0c1680002d80: fd fd fa fa fa fa fa fa fa fa fd fd fd fd fd fd
  0x0c1680002d90: fd fd fd fd fd fd fd fa fa fa fa fa fa fa fa fa
  0x0c1680002da0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fa fa
  0x0c1680002db0: fa fa fa fa fa fa fd fd fd fd fd fd fd fd fd fd
  0x0c1680002dc0: fd fd fd fd fa fa fa fa fa fa fa fa fd fd fd fd
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
Sequence.cc:305:12: runtime error: member access within address 0x60b000056ba0 which does not point to an object of type 'esiSequence'
0x60b000056ba0: note: object has invalid vptr
 00 00 00 00  1f 00 00 77 00 00 00 00  30 2b 21 00 20 60 00 00  30 2b 21 00 20 60 00 00  38 2b 21 00
              ^~~~~~~~~~~~~~~~~~~~~~~
              invalid vptr
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior Sequence.cc:305:12 in 
=================================================================
==1704956==ERROR: AddressSanitizer: heap-use-after-free on address 0x60b000056bdc at pc 0x00000228d1d9 bp 0x7fffffff70d0 sp 0x7fffffff70c8
READ of size 4 at 0x60b000056bdc thread T0
    #0 0x228d1d8 in esiSequence::process(int) src/esi/Sequence.cc:305:12
    #1 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #2 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #3 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #4 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #5 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #6 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #7 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #8 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #9 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #10 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #11 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #12 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #13 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #14 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #15 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #16 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #17 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #18 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #19 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #20 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #21 0x93837b in EventLoop::runOnce() src/EventLoop.cc:109:23
    #22 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #23 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #24 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #25 0x10f4819 in main src/main.cc:1391:12
    #26 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #27 0x71dbad in _start (src/squid+0x71dbad)

0x60b000056bdc is located 60 bytes inside of 112-byte region [0x60b000056ba0,0x60b000056c10)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3084 in free_const compat/xalloc.cc:175:5
    #2 0x232e897 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232e270 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2317e45 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x23123d6 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222ef6f in esiWhen::operator delete(void*) src/esi/Esi.cc:194:5
    #7 0x222a4c4 in esiWhen::~esiWhen() src/esi/Esi.cc:2202:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d1fb in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220217 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1996:13
    #11 0x2282bff in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x228c675 in esiSequence::process(int) src/esi/Sequence.cc:290:21
    #13 0x2287702 in esiSequence::processOne(int, unsigned long) src/esi/Sequence.cc:204:30
    #14 0x2285e4c in esiSequence::processStep(int) src/esi/Sequence.cc:191:37
    #15 0x228ad0d in esiSequence::process(int) src/esi/Sequence.cc:266:9
    #16 0x21d143f in ESIContext::process() src/esi/Esi.cc:1338:24
    #17 0x21cca33 in ESIContext::kick() src/esi/Esi.cc:315:17
    #18 0x21e39a1 in esiProcessStream(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Esi.cc:787:22
    #19 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #20 0xe7ad40 in clientReplyContext::pushStreamData(StoreIOBuffer const&, char*) src/client_side_reply.cc:1858:5
    #21 0xe24cb5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2144:9
    #22 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #23 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #24 0x12812b8 in store_client::doCopy(StoreEntry*) src/store_client.cc:373:9
    #25 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #26 0x129a71c in storeClientCopyEvent(void*) src/store_client.cc:194:5
    #27 0xf3d05c in EventDialer::dial(AsyncCall&) src/event.cc:41:30
    #28 0xf3cb9e in AsyncCallT<EventDialer>::fire() src/../src/base/AsyncCall.h:145:34
    #29 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c2d5f in xmalloc compat/xalloc.cc:114:15
    #2 0x232dd2e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2317c06 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2311f06 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222eef5 in esiWhen::operator new(unsigned long) src/esi/Esi.cc:194:5
    #6 0x21f3645 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1061:19
    #7 0x22b061e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/Sequence.cc:305:12 in esiSequence::process(int)
Shadow bytes around the buggy address:
  0x0c1680002d20: fd fd fd fd fd fd fd fd fd fd fa fa fa fa fa fa
  0x0c1680002d30: fa fa fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c1680002d40: fa fa fa fa fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c1680002d50: fd fd fd fd fd fd fa fa fa fa fa fa fa fa fd fd
  0x0c1680002d60: fd fd fd fd fd fd fd fd fd fd fd fd fa fa fa fa
=>0x0c1680002d70: fa fa fa fa fd fd fd fd fd fd fd[fd]fd fd fd fd
  0x0c1680002d80: fd fd fa fa fa fa fa fa fa fa fd fd fd fd fd fd
  0x0c1680002d90: fd fd fd fd fd fd fd fa fa fa fa fa fa fa fa fa
  0x0c1680002da0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fa fa
  0x0c1680002db0: fa fa fa fa fa fa fd fd fd fd fd fd fd fd fd fd
  0x0c1680002dc0: fd fd fd fd fa fa fa fa fa fa fa fa fd fd fd fd
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
  ```
