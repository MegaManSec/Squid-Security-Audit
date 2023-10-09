
# Use-After-Free in ESI 'Try' (and 'Choose') Processing 
Squid is able to process Edge Side Includes (ESI) pages when it acts as a reverse proxy. By default, this processing is enabled by default, and can be 'triggered' by any page whose response can be manipulated to return a single response header, and content.
ESI is an xml-based markup language which, in layman terms, can be used to cache *specific parts* of a single webpage, thus allowing for the caching of static content on an otherwise dynamically created page.

Note: This bug is very similar to that described in `Use-After-Free in ESI Expression Evaluation`.

## The Issue
The ESI syntax includes a directive called `<esi:try>`. This directive is used when an *attempt* to do something is required. For example, a document could *attempt* to include an external page; if it succeeds, something may happen, and if it fails, something else may happen, depending on how the ESI syntax is scripted. It can thought of like exception handling. An example is:
```
<esi:try> 
    <esi:attempt>
        <esi:comment text="Include an ad"/> 
        <esi:include src="http://www.example.com/ad1.html"/> 
    </esi:attempt>
    <esi:except> 
        <esi:comment text="Just write some HTML instead"/> 
        <a href=www.akamai.com>www.example.com</a>
    </esi:except> 
</esi:try>
```
In this example, a request would be made to example.com. If it succeeds, it would be included on the page. If it fails, the HTML `<a href=www.akamai.com>www.example.com</a>` is printed on the page. The attempt *response* (i.e. whether it succeeded or failed) is handled by the `esiTry::notifyParent` function:
```
void
esiTry::notifyParent()
{
    if (flags.attemptfailed) {
        if (flags.exceptok) {
            parent->provideData (exceptbuffer, this);
            exceptbuffer = NULL;
        } else if (flags.exceptfailed || except.getRaw() == NULL) {
            parent->fail (this, "esi:try - except claused failed, or no except clause found");
        }
    }
    /* nothing to do when except fails and attempt hasn't */
}
```

The issue here is that `provideData` frees `this`, and thus multiple use-after-frees (both write and reads) occur throughout the rest of the code, including the function which is calling `notifyParent()`.

A valid proof of concept is given:
```
<html xmlns:esi="http://www.edge-delivery.org/esi/1.0">
<esi:assign name="k" value="gene"/>
<esi:try>
<esi:attempt>
<esi:comment text="Include an ad"/>
<esi:include src="http://www.example.com/ad1.html"/>
</esi:attempt>
<esi:except>
<esi:comment text="Just write some HTML instead"/>
</esi:except>
</esi:try>
</html>
```
which will cause Squid to exhibit multiple use-after-frees, as well as fatally crash due to a large amount of memory being misallocated. An extremely similar bug is also present in `<esi:choose>` usage:
```
<html xmlns:esi="http://www.edge-delivery.org/esi/1.0">
<esi:choose> 
    <esi:when test="$(HTTP_COOKIE{group})=='Advanced'"> 
        <esi:include src="http://www.example.com/advanced.html"/> 
    </esi:when> 
    <esi:when test="$(HTTP_COOKIE{group})=='Basic User'">
        <esi:include src="http://www.example.com/basic.html"/>
    </esi:when> 
    <esi:otherwise> 
        <esi:include src="http://www.example.com/basic.html"/>
    </esi:otherwise>
    <esi:otherwise> 
        <esi:include src="http://www.example.com/basic.html"/>
    </esi:otherwise>
</esi:choose>
</html>
```


A crash and use-after-frees will occur in both cases:
```
Esi.cc:1803:13: runtime error: member access within address 0x6060001578e0 which does not point to an object of type 'esiTry'
0x6060001578e0: note: object has invalid vptr
 00 00 00 00  27 00 80 64 00 00 00 00  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  06 00 00 00
              ^~~~~~~~~~~~~~~~~~~~~~~
              invalid vptr
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior Esi.cc:1803:13 in 
=================================================================
==1859085==ERROR: AddressSanitizer: heap-use-after-free on address 0x606000157908 at pc 0x000002236708 bp 0x7fffffff4c70 sp 0x7fffffff4c68
READ of size 8 at 0x606000157908 thread T0
    #0 0x2236707 in RefCount<ESISegment>::dereference(ESISegment const*) src/esi/../../src/base/RefCount.h:96:28
    #1 0x222cf5b in RefCount<ESISegment>::operator=(RefCount<ESISegment>&&) src/esi/../../src/base/RefCount.h:63:13
    #2 0x2212d39 in esiTry::notifyParent() src/esi/Esi.cc:1803:26
    #3 0x2214106 in esiTry::fail(ESIElement*, char const*) src/esi/Esi.cc:1825:5
    #4 0x228e82f in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:321:13
    #5 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #6 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #7 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #8 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #9 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #10 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #11 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #12 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #13 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #14 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #15 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #16 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #17 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #18 0x1283542 in store_client::scheduleMemRead() src/store_client.cc:474:5
    #19 0x1282c53 in store_client::scheduleRead() src/store_client.cc:440:9
    #20 0x1281c66 in store_client::doCopy(StoreEntry*) src/store_client.cc:401:5
    #21 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #22 0x12982a1 in StoreEntry::invokeHandlers() src/store_client.cc:793:9
    #23 0x125eef1 in StoreEntry::flush() src/store.cc:1651:9
    #24 0x1240f87 in StoreEntry::startWriting() src/store.cc:1797:5
    #25 0x15ef9ef in Client::setFinalReply(HttpReply*) src/clients/Client.cc:153:12
    #26 0x161a8b1 in Client::adaptOrFinalizeReply() src/clients/Client.cc:978:5
    #27 0x1049c6b in HttpStateData::processReply() src/http.cc:1330:9
    #28 0x105d9b4 in HttpStateData::readReply(CommIoCbParams const&) src/http.cc:1302:5
    #29 0x10b19b6 in CommCbMemFunT<HttpStateData, CommIoCbParams>::doDial() src/../src/CommCalls.h:205:29
    #30 0x10ab5ca in JobDialer<HttpStateData>::dial(AsyncCall&) src/../src/base/AsyncJobCalls.h:175:9
    #31 0x10b25c7 in AsyncCallT<CommCbMemFunT<HttpStateData, CommIoCbParams> >::fire() src/../src/base/AsyncCall.h:145:34
    #32 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #33 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #34 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #35 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #36 0x93866d in EventLoop::runOnce() src/EventLoop.cc:121:19
    #37 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #38 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #39 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #40 0x10f4819 in main src/main.cc:1391:12
    #41 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #42 0x71dbad in _start (src/squid+0x71dbad)

0x606000157908 is located 40 bytes inside of 64-byte region [0x6060001578e0,0x606000157920)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3ce4 in free_const compat/xalloc.cc:175:5
    #2 0x232f4f7 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232eed0 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2318aa5 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x2313036 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222e36f in esiTry::operator delete(void*) src/esi/Esi.cc:125:5
    #7 0x220aa94 in esiTry::~esiTry() src/esi/Esi.cc:1643:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d4ab in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220837 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1998:13
    #11 0x2282eef in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x2212c31 in esiTry::notifyParent() src/esi/Esi.cc:1802:21
    #13 0x2214106 in esiTry::fail(ESIElement*, char const*) src/esi/Esi.cc:1825:5
    #14 0x228e82f in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:321:13
    #15 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #16 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #17 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #18 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #19 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #20 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #21 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #22 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #23 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #24 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #25 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #26 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #27 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #28 0x1283542 in store_client::scheduleMemRead() src/store_client.cc:474:5
    #29 0x1282c53 in store_client::scheduleRead() src/store_client.cc:440:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c39bf in xmalloc compat/xalloc.cc:114:15
    #2 0x232e98e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2318866 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2312b66 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222e2f5 in esiTry::operator new(unsigned long) src/esi/Esi.cc:125:5
    #6 0x21f2501 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1036:19
    #7 0x22b095e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/../../src/base/RefCount.h:96:28 in RefCount<ESISegment>::dereference(ESISegment const*)
Shadow bytes around the buggy address:
  0x0c0c80022ed0: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80022ee0: 00 00 00 00 00 00 00 00 fa fa fa fa fd fd fd fd
  0x0c0c80022ef0: fd fd fd fd fa fa fa fa fd fd fd fd fd fd fd fa
  0x0c0c80022f00: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80022f10: 00 00 00 00 00 00 00 00 fa fa fa fa fd fd fd fd
=>0x0c0c80022f20: fd[fd]fd fd fa fa fa fa fd fd fd fd fd fd fd fa
  0x0c0c80022f30: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80022f40: fd fd fd fd fd fd fd fa fa fa fa fa fd fd fd fd
  0x0c0c80022f50: fd fd fd fd fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c0c80022f60: fa fa fa fa fd fd fd fd fd fd fd fa fa fa fa fa
  0x0c0c80022f70: fd fd fd fd fd fd fd fd fa fa fa fa 00 00 00 00
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
==1859085==ERROR: AddressSanitizer: heap-use-after-free on address 0x606000157908 at pc 0x000002236735 bp 0x7fffffff4c70 sp 0x7fffffff4c68
WRITE of size 8 at 0x606000157908 thread T0
    #0 0x2236734 in RefCount<ESISegment>::dereference(ESISegment const*) src/esi/../../src/base/RefCount.h:97:12
    #1 0x222cf5b in RefCount<ESISegment>::operator=(RefCount<ESISegment>&&) src/esi/../../src/base/RefCount.h:63:13
    #2 0x2212d39 in esiTry::notifyParent() src/esi/Esi.cc:1803:26
    #3 0x2214106 in esiTry::fail(ESIElement*, char const*) src/esi/Esi.cc:1825:5
    #4 0x228e82f in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:321:13
    #5 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #6 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #7 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #8 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #9 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #10 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #11 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #12 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #13 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #14 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #15 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #16 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #17 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #18 0x1283542 in store_client::scheduleMemRead() src/store_client.cc:474:5
    #19 0x1282c53 in store_client::scheduleRead() src/store_client.cc:440:9
    #20 0x1281c66 in store_client::doCopy(StoreEntry*) src/store_client.cc:401:5
    #21 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #22 0x12982a1 in StoreEntry::invokeHandlers() src/store_client.cc:793:9
    #23 0x125eef1 in StoreEntry::flush() src/store.cc:1651:9
    #24 0x1240f87 in StoreEntry::startWriting() src/store.cc:1797:5
    #25 0x15ef9ef in Client::setFinalReply(HttpReply*) src/clients/Client.cc:153:12
    #26 0x161a8b1 in Client::adaptOrFinalizeReply() src/clients/Client.cc:978:5
    #27 0x1049c6b in HttpStateData::processReply() src/http.cc:1330:9
    #28 0x105d9b4 in HttpStateData::readReply(CommIoCbParams const&) src/http.cc:1302:5
    #29 0x10b19b6 in CommCbMemFunT<HttpStateData, CommIoCbParams>::doDial() src/../src/CommCalls.h:205:29
    #30 0x10ab5ca in JobDialer<HttpStateData>::dial(AsyncCall&) src/../src/base/AsyncJobCalls.h:175:9
    #31 0x10b25c7 in AsyncCallT<CommCbMemFunT<HttpStateData, CommIoCbParams> >::fire() src/../src/base/AsyncCall.h:145:34
    #32 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #33 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #34 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #35 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #36 0x93866d in EventLoop::runOnce() src/EventLoop.cc:121:19
    #37 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #38 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #39 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #40 0x10f4819 in main src/main.cc:1391:12
    #41 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #42 0x71dbad in _start (src/squid+0x71dbad)

0x606000157908 is located 40 bytes inside of 64-byte region [0x6060001578e0,0x606000157920)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3ce4 in free_const compat/xalloc.cc:175:5
    #2 0x232f4f7 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232eed0 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2318aa5 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x2313036 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222e36f in esiTry::operator delete(void*) src/esi/Esi.cc:125:5
    #7 0x220aa94 in esiTry::~esiTry() src/esi/Esi.cc:1643:1
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d4ab in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2220837 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1998:13
    #11 0x2282eef in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #12 0x2212c31 in esiTry::notifyParent() src/esi/Esi.cc:1802:21
    #13 0x2214106 in esiTry::fail(ESIElement*, char const*) src/esi/Esi.cc:1825:5
    #14 0x228e82f in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:321:13
    #15 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #16 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #17 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #18 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #19 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #20 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #21 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #22 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #23 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #24 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #25 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #26 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #27 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #28 0x1283542 in store_client::scheduleMemRead() src/store_client.cc:474:5
    #29 0x1282c53 in store_client::scheduleRead() src/store_client.cc:440:9

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c39bf in xmalloc compat/xalloc.cc:114:15
    #2 0x232e98e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2318866 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2312b66 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222e2f5 in esiTry::operator new(unsigned long) src/esi/Esi.cc:125:5
    #6 0x21f2501 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1036:19
    #7 0x22b095e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/../../src/base/RefCount.h:97:12 in RefCount<ESISegment>::dereference(ESISegment const*)
Shadow bytes around the buggy address:
  0x0c0c80022ed0: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80022ee0: 00 00 00 00 00 00 00 00 fa fa fa fa fd fd fd fd
  0x0c0c80022ef0: fd fd fd fd fa fa fa fa fd fd fd fd fd fd fd fa
  0x0c0c80022f00: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80022f10: 00 00 00 00 00 00 00 00 fa fa fa fa fd fd fd fd
=>0x0c0c80022f20: fd[fd]fd fd fa fa fa fa fd fd fd fd fd fd fd fa
  0x0c0c80022f30: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80022f40: fd fd fd fd fd fd fd fa fa fa fa fa fd fd fd fd
  0x0c0c80022f50: fd fd fd fd fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c0c80022f60: fa fa fa fa fd fd fd fd fd fd fd fa fa fa fa fa
  0x0c0c80022f70: fd fd fd fd fd fd fd fd fa fa fa fa 00 00 00 00
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
Sequence.cc:322:23: runtime error: member access within address 0x60800008be20 which does not point to an object of type 'esiSequence'
0x60800008be20: note: object has invalid vptr
 00 00 00 00  26 00 00 3a 00 00 00 00  e0 7e 15 00 60 60 00 00  08 7f 15 00 60 60 00 00  20 7f 15 00
              ^~~~~~~~~~~~~~~~~~~~~~~
              invalid vptr
SUMMARY: UndefinedBehaviorSanitizer: undefined-behavior Sequence.cc:322:23 in 
=================================================================
==1859085==ERROR: AddressSanitizer: heap-use-after-free on address 0x60800008be28 at pc 0x00000223b569 bp 0x7fffffff4f30 sp 0x7fffffff4f28
READ of size 8 at 0x60800008be28 thread T0
    #0 0x223b568 in __gnu_cxx::__normal_iterator<RefCount<ESIElement>*, std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > > >::__normal_iterator(RefCount<ESIElement>* const&) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_iterator.h:954:20
    #1 0x22315e1 in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::begin() /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:812:16
    #2 0x221adbe in FinishAllElements(std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >&) src/esi/Esi.cc:2005:24
    #3 0x228e904 in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:322:5
    #4 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #5 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #6 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #7 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #8 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #9 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #10 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #11 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #12 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #13 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #14 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #15 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #16 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #17 0x1283542 in store_client::scheduleMemRead() src/store_client.cc:474:5
    #18 0x1282c53 in store_client::scheduleRead() src/store_client.cc:440:9
    #19 0x1281c66 in store_client::doCopy(StoreEntry*) src/store_client.cc:401:5
    #20 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #21 0x12982a1 in StoreEntry::invokeHandlers() src/store_client.cc:793:9
    #22 0x125eef1 in StoreEntry::flush() src/store.cc:1651:9
    #23 0x1240f87 in StoreEntry::startWriting() src/store.cc:1797:5
    #24 0x15ef9ef in Client::setFinalReply(HttpReply*) src/clients/Client.cc:153:12
    #25 0x161a8b1 in Client::adaptOrFinalizeReply() src/clients/Client.cc:978:5
    #26 0x1049c6b in HttpStateData::processReply() src/http.cc:1330:9
    #27 0x105d9b4 in HttpStateData::readReply(CommIoCbParams const&) src/http.cc:1302:5
    #28 0x10b19b6 in CommCbMemFunT<HttpStateData, CommIoCbParams>::doDial() src/../src/CommCalls.h:205:29
    #29 0x10ab5ca in JobDialer<HttpStateData>::dial(AsyncCall&) src/../src/base/AsyncJobCalls.h:175:9
    #30 0x10b25c7 in AsyncCallT<CommCbMemFunT<HttpStateData, CommIoCbParams> >::fire() src/../src/base/AsyncCall.h:145:34
    #31 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #32 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #33 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #34 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #35 0x93866d in EventLoop::runOnce() src/EventLoop.cc:121:19
    #36 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #37 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #38 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #39 0x10f4819 in main src/main.cc:1391:12
    #40 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #41 0x71dbad in _start (src/squid+0x71dbad)

0x60800008be28 is located 8 bytes inside of 88-byte region [0x60800008be20,0x60800008be78)
freed by thread T0 here:
    #0 0x798ba2 in free (src/squid+0x798ba2)
    #1 0x23c3ce4 in free_const compat/xalloc.cc:175:5
    #2 0x232f4f7 in xfree(void const*) src/mem/../../compat/xalloc.h:60:50
    #3 0x232eed0 in MemPoolMalloc::deallocate(void*, bool) src/mem/PoolMalloc.cc:49:9
    #4 0x2318aa5 in MemImplementingAllocator::freeOne(void*) src/mem/Pool.cc:210:5
    #5 0x2313036 in Mem::AllocatorProxy::freeOne(void*) src/mem/AllocatorProxy.cc:22:21
    #6 0x222e7ef in esiSequence::operator delete(void*) src/esi/../../src/esi/Sequence.h:21:5
    #7 0x2234ff4 in esiAttempt::~esiAttempt() src/esi/../../src/esi/Attempt.h:17:8
    #8 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #9 0x222d4ab in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #10 0x2219c69 in esiTry::finish() src/esi/Esi.cc:1896:13
    #11 0x22200c0 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1995:18
    #12 0x2282eef in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #13 0x2212c31 in esiTry::notifyParent() src/esi/Esi.cc:1802:21
    #14 0x2214106 in esiTry::fail(ESIElement*, char const*) src/esi/Esi.cc:1825:5
    #15 0x228e82f in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:321:13
    #16 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #17 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #18 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #19 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #20 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #21 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #22 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #23 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #24 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #25 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #26 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #27 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #28 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #29 0x1283542 in store_client::scheduleMemRead() src/store_client.cc:474:5

previously allocated by thread T0 here:
    #0 0x798e0d in malloc (src/squid+0x798e0d)
    #1 0x23c39bf in xmalloc compat/xalloc.cc:114:15
    #2 0x232e98e in MemPoolMalloc::allocate() src/mem/PoolMalloc.cc:37:19
    #3 0x2318866 in MemImplementingAllocator::alloc() src/mem/Pool.cc:202:12
    #4 0x2312b66 in Mem::AllocatorProxy::alloc() src/mem/AllocatorProxy.cc:16:28
    #5 0x222e425 in esiSequence::operator new(unsigned long) src/esi/../../src/esi/Sequence.h:21:5
    #6 0x21f2885 in ESIContext::start(char const*, char const**, unsigned long) src/esi/Esi.cc:1041:19
    #7 0x22b095e in ESIExpatParser::Start(void*, char const*, char const**) src/esi/ExpatParser.cc:41:20
    #8 0x7ffff7d2c6b8  (/lib/x86_64-linux-gnu/libexpat.so.1+0xb6b8)

SUMMARY: AddressSanitizer: heap-use-after-free /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_iterator.h:954:20 in __gnu_cxx::__normal_iterator<RefCount<ESIElement>*, std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > > >::__normal_iterator(RefCount<ESIElement>* const&)
Shadow bytes around the buggy address:
  0x0c1080009770: fa fa fa fa 00 00 00 00 00 00 00 00 00 00 00 fa
  0x0c1080009780: fa fa fa fa fd fd fd fd fd fd fd fd fd fd fd fa
  0x0c1080009790: fa fa fa fa fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c10800097a0: fa fa fa fa fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c10800097b0: fa fa fa fa 00 00 00 00 00 00 00 00 00 00 00 fa
=>0x0c10800097c0: fa fa fa fa fd[fd]fd fd fd fd fd fd fd fd fd fa
  0x0c10800097d0: fa fa fa fa fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c10800097e0: fa fa fa fa fd fd fd fd fd fd fd fd fd fd fd fa
  0x0c10800097f0: fa fa fa fa fd fd fd fd fd fd fd fd fd fd fd fa
  0x0c1080009800: fa fa fa fa fd fd fd fd fd fd fd fd fd fd fd fa
  0x0c1080009810: fa fa fa fa fd fd fd fd fd fd fd fd fd fd fd fa
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
==1859085==ERROR: AddressSanitizer: heap-use-after-free on address 0x606000157ee0 at pc 0x000002231401 bp 0x7fffffff4e10 sp 0x7fffffff4e08
READ of size 8 at 0x606000157ee0 thread T0
    #0 0x2231400 in RefCount<ESIElement>::operator bool() const src/esi/../../src/base/RefCount.h:69:45
    #1 0x221ff0c in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1994:9
    #2 0x221af50 in FinishAllElements(std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >&) src/esi/Esi.cc:2006:9
    #3 0x228e904 in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:322:5
    #4 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #5 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #6 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #7 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #8 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #9 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #10 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #11 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #12 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #13 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #14 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #15 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #16 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #17 0x1283542 in store_client::scheduleMemRead() src/store_client.cc:474:5
    #18 0x1282c53 in store_client::scheduleRead() src/store_client.cc:440:9
    #19 0x1281c66 in store_client::doCopy(StoreEntry*) src/store_client.cc:401:5
    #20 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #21 0x12982a1 in StoreEntry::invokeHandlers() src/store_client.cc:793:9
    #22 0x125eef1 in StoreEntry::flush() src/store.cc:1651:9
    #23 0x1240f87 in StoreEntry::startWriting() src/store.cc:1797:5
    #24 0x15ef9ef in Client::setFinalReply(HttpReply*) src/clients/Client.cc:153:12
    #25 0x161a8b1 in Client::adaptOrFinalizeReply() src/clients/Client.cc:978:5
    #26 0x1049c6b in HttpStateData::processReply() src/http.cc:1330:9
    #27 0x105d9b4 in HttpStateData::readReply(CommIoCbParams const&) src/http.cc:1302:5
    #28 0x10b19b6 in CommCbMemFunT<HttpStateData, CommIoCbParams>::doDial() src/../src/CommCalls.h:205:29
    #29 0x10ab5ca in JobDialer<HttpStateData>::dial(AsyncCall&) src/../src/base/AsyncJobCalls.h:175:9
    #30 0x10b25c7 in AsyncCallT<CommCbMemFunT<HttpStateData, CommIoCbParams> >::fire() src/../src/base/AsyncCall.h:145:34
    #31 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #32 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #33 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #34 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #35 0x93866d in EventLoop::runOnce() src/EventLoop.cc:121:19
    #36 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #37 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #38 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #39 0x10f4819 in main src/main.cc:1391:12
    #40 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #41 0x71dbad in _start (src/squid+0x71dbad)

0x606000157ee0 is located 0 bytes inside of 64-byte region [0x606000157ee0,0x606000157f20)
freed by thread T0 here:
    #0 0x7ca72d in operator delete(void*) (src/squid+0x7ca72d)
    #1 0x2238f91 in __gnu_cxx::new_allocator<RefCount<ESIElement> >::deallocate(RefCount<ESIElement>*, unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/ext/new_allocator.h:133:2
    #2 0x2238f41 in std::allocator_traits<std::allocator<RefCount<ESIElement> > >::deallocate(std::allocator<RefCount<ESIElement> >&, RefCount<ESIElement>*, unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/alloc_traits.h:492:13
    #3 0x2238e80 in std::_Vector_base<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::_M_deallocate(RefCount<ESIElement>*, unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:354:4
    #4 0x2238c06 in std::_Vector_base<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::~_Vector_base() /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:335:2
    #5 0x223062a in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::~vector() /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:683:7
    #6 0x227dd47 in esiSequence::~esiSequence() src/esi/Sequence.cc:31:1
    #7 0x223515f in esiAttempt::~esiAttempt() src/esi/../../src/esi/Attempt.h:17:8
    #8 0x2234f85 in esiAttempt::~esiAttempt() src/esi/../../src/esi/Attempt.h:17:8
    #9 0x2234feb in esiAttempt::~esiAttempt() src/esi/../../src/esi/Attempt.h:17:8
    #10 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #11 0x222d4ab in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #12 0x2219c69 in esiTry::finish() src/esi/Esi.cc:1896:13
    #13 0x22200c0 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1995:18
    #14 0x2282eef in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #15 0x2212c31 in esiTry::notifyParent() src/esi/Esi.cc:1802:21
    #16 0x2214106 in esiTry::fail(ESIElement*, char const*) src/esi/Esi.cc:1825:5
    #17 0x228e82f in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:321:13
    #18 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #19 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #20 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #21 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #22 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #23 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #24 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #25 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #26 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #27 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #28 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #29 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14

previously allocated by thread T0 here:
    #0 0x7c9ecd in operator new(unsigned long) (src/squid+0x7c9ecd)
    #1 0x223b109 in __gnu_cxx::new_allocator<RefCount<ESIElement> >::allocate(unsigned long, void const*) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/ext/new_allocator.h:115:27
    #2 0x223b06d in std::allocator_traits<std::allocator<RefCount<ESIElement> > >::allocate(std::allocator<RefCount<ESIElement> >&, unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/alloc_traits.h:460:20
    #3 0x223a9d8 in std::_Vector_base<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::_M_allocate(unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:346:20
    #4 0x223958e in void std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::_M_realloc_insert<RefCount<ESIElement> const&>(__gnu_cxx::__normal_iterator<RefCount<ESIElement>*, std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > > >, RefCount<ESIElement> const&) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/vector.tcc:440:33
    #5 0x22311e6 in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::push_back(RefCount<ESIElement> const&) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:1198:4
    #6 0x22856f5 in esiSequence::addElement(RefCount<ESIElement>) src/esi/Sequence.cc:171:14
    #7 0x21f56a7 in ESIContext::addLiteral(char const*, int) src/esi/Esi.cc:1199:29
    #8 0x21f8ab4 in ESIContext::parserDefault(char const*, int) src/esi/Esi.cc:1141:5
    #9 0x22b0f72 in ESIExpatParser::Default(void*, char const*, int) src/esi/ExpatParser.cc:57:20
    #10 0x7ffff7d25695  (/lib/x86_64-linux-gnu/libexpat.so.1+0x4695)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/../../src/base/RefCount.h:69:45 in RefCount<ESIElement>::operator bool() const
Shadow bytes around the buggy address:
  0x0c0c80022f80: 00 00 00 00 fa fa fa fa fd fd fd fd fd fd fd fa
  0x0c0c80022f90: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80022fa0: 00 00 00 00 00 00 00 00 fa fa fa fa 00 00 00 00
  0x0c0c80022fb0: 00 00 00 00 fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c0c80022fc0: fa fa fa fa fd fd fd fd fd fd fd fa fa fa fa fa
=>0x0c0c80022fd0: fd fd fd fd fd fd fd fd fa fa fa fa[fd]fd fd fd
  0x0c0c80022fe0: fd fd fd fd fa fa fa fa fd fd fd fd fd fd fd fa
  0x0c0c80022ff0: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80023000: fd fd fd fd fd fd fd fa fa fa fa fa fd fd fd fd
  0x0c0c80023010: fd fd fd fd fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c0c80023020: fa fa fa fa fd fd fd fd fd fd fd fa fa fa fa fa
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
==1859085==ERROR: AddressSanitizer: heap-use-after-free on address 0x606000157ee0 at pc 0x00000222d051 bp 0x7fffffff4e10 sp 0x7fffffff4e08
READ of size 8 at 0x606000157ee0 thread T0
    #0 0x222d050 in RefCount<ESIElement>::operator->() const src/esi/../../src/base/RefCount.h:73:53
    #1 0x221ff65 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1995:9
    #2 0x221af50 in FinishAllElements(std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >&) src/esi/Esi.cc:2006:9
    #3 0x228e904 in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:322:5
    #4 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #5 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #6 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #7 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #8 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #9 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #10 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #11 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #12 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #13 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #14 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #15 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14
    #16 0x127b77e in store_client::callback(long, bool) src/store_client.cc:178:9
    #17 0x1283542 in store_client::scheduleMemRead() src/store_client.cc:474:5
    #18 0x1282c53 in store_client::scheduleRead() src/store_client.cc:440:9
    #19 0x1281c66 in store_client::doCopy(StoreEntry*) src/store_client.cc:401:5
    #20 0x127f4d0 in storeClientCopy2(StoreEntry*, store_client*) src/store_client.cc:355:9
    #21 0x12982a1 in StoreEntry::invokeHandlers() src/store_client.cc:793:9
    #22 0x125eef1 in StoreEntry::flush() src/store.cc:1651:9
    #23 0x1240f87 in StoreEntry::startWriting() src/store.cc:1797:5
    #24 0x15ef9ef in Client::setFinalReply(HttpReply*) src/clients/Client.cc:153:12
    #25 0x161a8b1 in Client::adaptOrFinalizeReply() src/clients/Client.cc:978:5
    #26 0x1049c6b in HttpStateData::processReply() src/http.cc:1330:9
    #27 0x105d9b4 in HttpStateData::readReply(CommIoCbParams const&) src/http.cc:1302:5
    #28 0x10b19b6 in CommCbMemFunT<HttpStateData, CommIoCbParams>::doDial() src/../src/CommCalls.h:205:29
    #29 0x10ab5ca in JobDialer<HttpStateData>::dial(AsyncCall&) src/../src/base/AsyncJobCalls.h:175:9
    #30 0x10b25c7 in AsyncCallT<CommCbMemFunT<HttpStateData, CommIoCbParams> >::fire() src/../src/base/AsyncCall.h:145:34
    #31 0x1884f4e in AsyncCall::make() src/base/AsyncCall.cc:44:9
    #32 0x188bc11 in AsyncCallQueue::fireNext() src/base/AsyncCallQueue.cc:60:11
    #33 0x188a611 in AsyncCallQueue::fire() src/base/AsyncCallQueue.cc:43:9
    #34 0x938da6 in EventLoop::dispatchCalls() src/EventLoop.cc:144:54
    #35 0x93866d in EventLoop::runOnce() src/EventLoop.cc:121:19
    #36 0x937cf2 in EventLoop::run() src/EventLoop.cc:83:13
    #37 0x10f6d51 in SquidMain(int, char**) src/main.cc:1716:14
    #38 0x10f484c in SquidMainSafe(int, char**) src/main.cc:1403:16
    #39 0x10f4819 in main src/main.cc:1391:12
    #40 0x7ffff73cd0b2 in __libc_start_main /build/glibc-eX1tMB/glibc-2.31/csu/../csu/libc-start.c:308:16
    #41 0x71dbad in _start (src/squid+0x71dbad)

0x606000157ee0 is located 0 bytes inside of 64-byte region [0x606000157ee0,0x606000157f20)
freed by thread T0 here:
    #0 0x7ca72d in operator delete(void*) (src/squid+0x7ca72d)
    #1 0x2238f91 in __gnu_cxx::new_allocator<RefCount<ESIElement> >::deallocate(RefCount<ESIElement>*, unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/ext/new_allocator.h:133:2
    #2 0x2238f41 in std::allocator_traits<std::allocator<RefCount<ESIElement> > >::deallocate(std::allocator<RefCount<ESIElement> >&, RefCount<ESIElement>*, unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/alloc_traits.h:492:13
    #3 0x2238e80 in std::_Vector_base<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::_M_deallocate(RefCount<ESIElement>*, unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:354:4
    #4 0x2238c06 in std::_Vector_base<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::~_Vector_base() /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:335:2
    #5 0x223062a in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::~vector() /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:683:7
    #6 0x227dd47 in esiSequence::~esiSequence() src/esi/Sequence.cc:31:1
    #7 0x223515f in esiAttempt::~esiAttempt() src/esi/../../src/esi/Attempt.h:17:8
    #8 0x2234f85 in esiAttempt::~esiAttempt() src/esi/../../src/esi/Attempt.h:17:8
    #9 0x2234feb in esiAttempt::~esiAttempt() src/esi/../../src/esi/Attempt.h:17:8
    #10 0x127435c in RefCount<ESIElement>::dereference(ESIElement const*) src/../src/base/RefCount.h:100:13
    #11 0x222d4ab in RefCount<ESIElement>::operator=(RefCount<ESIElement>&&) src/esi/../../src/base/RefCount.h:63:13
    #12 0x2219c69 in esiTry::finish() src/esi/Esi.cc:1896:13
    #13 0x22200c0 in FinishAnElement(RefCount<ESIElement>&, int) src/esi/Esi.cc:1995:18
    #14 0x2282eef in esiSequence::provideData(RefCount<ESISegment>, ESIElement*) src/esi/Sequence.cc:131:5
    #15 0x2212c31 in esiTry::notifyParent() src/esi/Esi.cc:1802:21
    #16 0x2214106 in esiTry::fail(ESIElement*, char const*) src/esi/Esi.cc:1825:5
    #17 0x228e82f in esiSequence::fail(ESIElement*, char const*) src/esi/Sequence.cc:321:13
    #18 0x227155b in ESIInclude::subRequestDone(RefCount<ESIStreamContext>, bool) src/esi/Include.cc:560:21
    #19 0x226ac17 in ESIInclude::includeFail(RefCount<ESIStreamContext>) src/esi/Include.cc:445:5
    #20 0x225b094 in esiBufferRecipient(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/esi/Include.cc:94:37
    #21 0xd2dde5 in clientStreamCallback(clientStreamNode*, ClientHttpRequest*, HttpReply*, StoreIOBuffer) src/clientStream.cc:159:5
    #22 0xe8424f in clientReplyContext::processReplyAccessResult(Acl::Answer const&) src/client_side_reply.cc:2070:5
    #23 0xe8468c in clientReplyContext::ProcessReplyAccessResult(Acl::Answer, void*) src/client_side_reply.cc:1977:9
    #24 0x15b2f34 in ACLChecklist::checkCallback(Acl::Answer) src/acl/Checklist.cc:169:9
    #25 0x15b39c6 in ACLChecklist::completeNonBlocking() src/acl/Checklist.cc:54:5
    #26 0x15bb2f5 in ACLChecklist::nonBlockingCheck(void (*)(Acl::Answer, void*), void*) src/acl/Checklist.cc:257:13
    #27 0xe7d8aa in clientReplyContext::processReplyAccess() src/client_side_reply.cc:1970:21
    #28 0xe24ea5 in clientReplyContext::sendMoreData(StoreIOBuffer) src/client_side_reply.cc:2156:5
    #29 0xe14e63 in clientReplyContext::SendMoreData(void*, StoreIOBuffer) src/client_side_reply.cc:1804:14

previously allocated by thread T0 here:
    #0 0x7c9ecd in operator new(unsigned long) (src/squid+0x7c9ecd)
    #1 0x223b109 in __gnu_cxx::new_allocator<RefCount<ESIElement> >::allocate(unsigned long, void const*) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/ext/new_allocator.h:115:27
    #2 0x223b06d in std::allocator_traits<std::allocator<RefCount<ESIElement> > >::allocate(std::allocator<RefCount<ESIElement> >&, unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/alloc_traits.h:460:20
    #3 0x223a9d8 in std::_Vector_base<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::_M_allocate(unsigned long) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:346:20
    #4 0x223958e in void std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::_M_realloc_insert<RefCount<ESIElement> const&>(__gnu_cxx::__normal_iterator<RefCount<ESIElement>*, std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > > >, RefCount<ESIElement> const&) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/vector.tcc:440:33
    #5 0x22311e6 in std::vector<RefCount<ESIElement>, std::allocator<RefCount<ESIElement> > >::push_back(RefCount<ESIElement> const&) /usr/bin/../lib/gcc/x86_64-linux-gnu/10/../../../../include/c++/10/bits/stl_vector.h:1198:4
    #6 0x22856f5 in esiSequence::addElement(RefCount<ESIElement>) src/esi/Sequence.cc:171:14
    #7 0x21f56a7 in ESIContext::addLiteral(char const*, int) src/esi/Esi.cc:1199:29
    #8 0x21f8ab4 in ESIContext::parserDefault(char const*, int) src/esi/Esi.cc:1141:5
    #9 0x22b0f72 in ESIExpatParser::Default(void*, char const*, int) src/esi/ExpatParser.cc:57:20
    #10 0x7ffff7d25695  (/lib/x86_64-linux-gnu/libexpat.so.1+0x4695)

SUMMARY: AddressSanitizer: heap-use-after-free src/esi/../../src/base/RefCount.h:73:53 in RefCount<ESIElement>::operator->() const
Shadow bytes around the buggy address:
  0x0c0c80022f80: 00 00 00 00 fa fa fa fa fd fd fd fd fd fd fd fa
  0x0c0c80022f90: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80022fa0: 00 00 00 00 00 00 00 00 fa fa fa fa 00 00 00 00
  0x0c0c80022fb0: 00 00 00 00 fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c0c80022fc0: fa fa fa fa fd fd fd fd fd fd fd fa fa fa fa fa
=>0x0c0c80022fd0: fd fd fd fd fd fd fd fd fa fa fa fa[fd]fd fd fd
  0x0c0c80022fe0: fd fd fd fd fa fa fa fa fd fd fd fd fd fd fd fa
  0x0c0c80022ff0: fa fa fa fa fd fd fd fd fd fd fd fd fa fa fa fa
  0x0c0c80023000: fd fd fd fd fd fd fd fa fa fa fa fa fd fd fd fd
  0x0c0c80023010: fd fd fd fd fa fa fa fa fd fd fd fd fd fd fd fd
  0x0c0c80023020: fa fa fa fa fd fd fd fd fd fd fd fa fa fa fa fa
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
2021/04/20 12:00:21.248| FATAL: Received Segment Violation...dying.
    current master transaction: master54
2021/04/20 12:00:21.248| Closing HTTP(S) port 0.0.0.0:2222
    current master transaction: master54
2021/04/20 12:00:21.395| 5,3| comm.cc(877) _comm_close: start closing FD 10 by Connection.cc:89
2021/04/20 12:00:21.395| 5,3| comm.cc(562) commUnsetFdTimeout: Remove timeout for FD 10
2021/04/20 12:00:21.395| storeDirWriteCleanLogs: Starting...
    current master transaction: master54
2021/04/20 12:00:21.395|   Finished.  Wrote 0 entries.
    current master transaction: master54
2021/04/20 12:00:21.395|   Took 0.00 seconds (  0.00 entries/sec).
    current master transaction: master54
2021/04/20 12:00:21.395| 21,3| tools.cc(494) uniqueHostname:  Config: '
sh: 1: mail: not found
Aborted

```


There are many many many other places this may occur, and a full assessment has not been made.
