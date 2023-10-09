# Assertion in Squid "Helper" Process Creator
Squid uses some external programs called "helpers", which are their own processes. These can include things like authorization verifiers. They are used like this in order to not block requests while they do 'work'. You can think of a helper as an "external" thread service.

## The Issue
When Squid spins up a worker, it uses the function `ipcCreate`. This function does not properly handle when Squid has a system-specific "too many" file descriptors open:
```
    t1 = dup(crfd);
    t2 = dup(cwfd);
    t3 = dup(fileno(debug_log));
    assert(t1 > 2 && t2 > 2 && t3 > 2);
```
The `dup()` function will return `-1` if too many file descriptors are already open.

It is trivial to open a large amount of connections, then request a page which will attempt to spin-up an external helper service, like a password-protected page.

"Too many file descriptors open" is an error which is handled gracefully everywhere else in Squid, leaving this the only vulnerable spot.
