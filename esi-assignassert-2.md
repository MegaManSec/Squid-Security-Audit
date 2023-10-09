# Assertion in ESI Variable Assignment (String)
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

When this sort of syntax is used, Squid parses the `name` component in order to define the variable for future use. If applicable, it also parses this component**s** (if there are multiple), converting it from a list, to a single character buffer using ``ESISegment::listToChar``:
```
char *
ESISegment::listToChar() const
{
    size_t length = listLength();
    char *rv = (char *)xmalloc (length + 1);
    assert (rv);
    rv [length] = '\0';

    ESISegment::Pointer temp = this;
    size_t pos = 0;

    while (temp.getRaw()) {
        memcpy(&rv[pos], temp->buf, temp->len);
        pos += temp->len;
        temp = temp->next;
    }

    return rv;
}
```
There is no limit to how long this list can be.

While there are *initial* checks for how long a `name` type is, this check is only made at the start of the processing of the ESI document, and is made against the literal strings inside the `name` component. Due to the way XML is parsed by the processor, this `name` component may be expanded after these checks by the XML processor, which makes them very large. Explanation and theory about this process goes beyond the scope of this document (sorry). But perhaps the following debug information is useful, from the function esiSequence::process:
```
[…]
[Checks the length of `name` and rejects if it is too long]
[…]
286             ESISegment::Pointer temp(new ESISegment);
287             render (temp); <---- 'renders' the XML inside the `name` component, expanding it much greater than originally
289             if (temp->next.getRaw() || temp->len) {
(gdb) p strlen(temp->listToChar()) 
$13 = 69071 <---- Our `name` is now larger than original checks would have rejected.
```
Anyways.

This means that, for example in the ``ESIAssign::provideData`` function, the function can return a character buffer of over 65535-bytes long. If this large buffer is then converted to a `String` type, an assertion will happen because a `String` can only hold 65535-bytes:
```
void
ESIAssign::provideData (ESISegment::Pointer data, ESIElement * source)
{
    assert (source == variable.getRaw());
    char *result = data->listToChar();
    unevaluatedVariable = result;
    safe_free (result);
}
```
Here, `unevaluatedVariable` is of type `String`, and an assertion will happen if `listToChar` returns a very large buffer.
An example is given:
```
<html xmlns:esi="http://www.edge-delivery.org/esi/1.0">
<esi:assign>
00<h000 xmlns:esi="0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000">
000000<esi:kk000000000>
0000<esi:w000 t000="00"/>
00000000000<esi:w000 t000="00000000000000000"/>
00000000000<esi:w00/>
00000000000<esi:w000 t000="000000000000000000000000"/>
0000000<esi:i000000000>
0000<esi:w000 t000="00"/>
00000000000<esi:w000 t000="00000000000000000000000000000000000000000000000"/>
0000000000000000000<esi:w000 t000="00"/>
00000000000<esi:w000 t000="000000000000000000000000000000000000000000000000000"/>
00000000000<esi:w000 t000="00000000000000000000000"/>
00000000000<esi:w000 t000="00000000000000000000000000000000000000000000000000000000000"/>
00000000000<esi:w000 t000="0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"/>
0000000<esi:i000000000>
0000<esi:w000 t000="00"/>
00000000000<esi:w000 t000="0000000000000000000000000000000000000000000000000000000"/>
0000000
</esi:i000000000>
</esi:i000000000>
</esi:kk000000000>
</h000>
</esi:assign>
</html>
```
