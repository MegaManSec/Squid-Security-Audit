# Memory Leak in CacheManager URI Parsing
Squid's internal Cache Manager is accessible by system administrators, usually using some sort of authentication. They require pages, such as, `cache_object://website.com/info`, to gain information about the system, debugging information, and statistics.

If any error or unexpected issue arises, most code is caught by exception handling in CacheManager.

## The Issue
The CacheManager is required to parse the page requested by the client, which is done in the function `CacheManager::ParseUrl`. This function uses `sscanf` to parse the absolute URI for the host, page, and the query parameters:
```
t = sscanf(url, "cache_object://%[^/]/%[^?]%n?%s", host, request, &pos, params);
```

The `params` variable is then passed to ``Mgr::QueryParams::Parse(params, cmd->params.queryParams)`` which does the following:
```
for (size_t i = n; i < len; ++i) {
   if (aParamsStr[i] == '&') {
      if (!ParseParam(aParamsStr.substr(n, i), param))
      …
   }
   …
}
```
If the function `substr` is passed something unexpected, like `n = i = 0`, it will cause an exception, which will be caught somewhere in the request handling code.

It is trivial to cause this exception with a request to the page `cache_object://0/io?&`
In this case, the `params` variable will contain a single character: `&`. 
As such, once the code inside `Mgr::QueryParams::Parse` is calls, it will set `n=0` and `i=0`, which will cause an exception. More explicitly:
```
2021/04/18 17:07:18.926| 16,3| cache_manager.cc(205) ParseUrl: MGR request: t=3, host='0', request='io', pos=19, password='', params='&'
2021/04/18 17:07:18.927| 0,3| String.cc(227) substr: check failed: to > 0 && to <= size()
    exception location: String.cc(227) substr
2021/04/18 17:07:18.927| 33,3| ../../src/base/AsyncJobCalls.h(178) dial: Server::doClientRead threw exception: check failed: to > 0 && to <= size()
    exception location: String.cc(227) substr
2021/04/18 17:07:18.927| 93,2| AsyncJob.cc(129) callException: check failed: to > 0 && to <= size()
    exception location: String.cc(227) substr
```

The (security) issue here is that memory has previously been allocated, which, when the exception is caught, will be leaked.
Specifically, the memory which is allocated for access control checking:
```
ACLFilledChecklist *
clientAclChecklistCreate(const acl_access * acl, ClientHttpRequest * http)
{
    const auto checklist = new ACLFilledChecklist(acl, nullptr, nullptr);
    clientAclChecklistFill(*checklist, http);
    return checklist;
}
```
is lost.

Luckily, given the CacheManager is a 'privileged area', it is unlikely this issue can be exploited too easily without convincing an (already authenticated) admin to visit a page to trigger it.

This bug was assigned [CVE-2021-28652](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-28652)

