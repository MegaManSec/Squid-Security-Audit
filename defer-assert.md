# Assertion Crash in Deferred Requests
Squid sometimes does what it calls 'deferring' of requests, which means it puts off a request for a later time. There are multiple reasons this can happen, and it goes beyond the scope of this document.

## The Issue

When a request is marked as deferred, it is never unmarked as un-deferred. This causes multiple issues, as in some cases, Squid will not know what to do with the request. 
As a Squid developer states:


>If the "deferred" state is not cleared when the request is made alive
>again, then I see how a "late" (using writeControlMsgAndCall()
>terminology) 1xx control message can trigger this assertion, just as
>depicted in the bug 5117 stack trace:
>> #2  0x0000555555e2305d in xassert ()
> #3  0x0000555555db403e in ClientSocketContextPushDeferredIfNeeded(RefCount<Http::Stream>, ConnStateData*) ()
> #4  0x0000555555ddb887 in ConnStateData::doneWithControlMsg() ()
> #5  0x0000555555ddafcf in ConnStateData::sendControlMsg(HttpControlMsg) ()
> #8  0x0000555555ea86df in AsyncCallT<UnaryMemFunT<ConnStateData, HttpControlMsg, HttpControlMsg> >::fire() ()

No further information is given about this crash because it is extremely difficult (but not impossible) to reproduce in a reliable manner.
