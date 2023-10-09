
# Assertion in ESI Variable Assignment
Squid is able to process Edge Side Includes (ESI) pages when it acts as a reverse proxy. By default, this processing is enabled by default, and can be 'triggered' by any page whose response can be manipulated to return a single response header, and content.
ESI is an xml-based markup language which, in layman terms, can be used to cache *specific parts* of a single webpage, thus allowing for the caching of static content on an otherwise dynamically created page.

## The Issue
The ESI syntax includes a directive called `<esi:assign>`. This directive can be used to *assign* the contents of a custom variable. For example, one could use the following assignment:
```
<esi:assign name="word" value="never gonna" />
```
and then use it throughout a page, such as:
```
<esi:vars name="$(word)"/> give you up, <esi:vars name="$(word)"/> let you down.
```
which would print ``never gonna give you up, never gonna let you down``.

When this sort of syntax is used, Squid parses the `name` component in order to define the variable for future use.
However, the issue is that there are no checks whether this name is well-formed before it undergoes parsing, and may actually be non-existent. For example, the following (invalid) ESI syntax could be used:
```
<html xmlns:esi="http://www.edge-delivery.org/esi/1.0">
<esi:assign>hello</esi:assign>
</html>
```
Without going into too much detail, the issue is that an empty character buffer is passed to the initialization of a `String` type, akin to:
```
String("", 0)
```
where the `0` here means length-0.
(For reference, this is done in`ESIAssign::process`, which uses `varState->addVariable (name.rawBuf(), name.size(), value);` with `name.size() == 0`).

Passing an empty character buffer to the `String` type results in an assertion:
```
assertion failed: String.cc:106: "str"
```

The example I have given is a real example. This is the only case in ESI's whole codebase which passes a non-existent char buffer to `String`.
