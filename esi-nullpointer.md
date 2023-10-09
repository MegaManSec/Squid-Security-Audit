# Null Pointer Dereference In ESI's esi:include and esi:when 
Squid is able to process Edge Side Includes (ESI) pages when it acts as a reverse proxy. By default, this processing is enabled by default, and can be 'triggered' by any page whose response can be manipulated to return a single response header, and content.
ESI is an xml-based markup language which, in layman terms, can be used to cache *specific parts* of a single webpage, thus allowing for the caching of static content on an otherwise dynamically created page.

## The Issue
The ESI syntax includes a directive called `<esi:when>`. This directive is used to test for boolean conditions. For example, ``<esi:when test="$(HTTP_USER_AGENT{'os'})=='iPhone'">`` could be used to check whether a request's user agent is identifying itself as an iPhone.
Simiarly, ESI has a directive called `<esi:include>` which can be used to include other files either locally or remotely.

In both of these cases, the function `ESIVarState::extractChar` is used in order to extract the contents of the tags, such as in the case of `<esi:when>` given before, it is used to extract `test=<this part>`. 

The bug is not very interesting so much of the technical details are left out of this document, however the point is, in both `<esi:when>` and `<esi:include>` cases, if nothing else is placed inside the tag they use (such as `when`'s "test" tag), `ESIVarState::extractChar` will cause a null pointer dereference:
```
char *
ESIVarState::extractChar ()
{
    if (!input.getRaw())
        fatal ("Attempt to extract variable state with no data fed in \n");

    doIt();
    char *rv = output->listToChar();

    ESISegmentFreeList (output);
    debugs(86, 6, "ESIVarStateExtractList: Extracted char");
    return rv;
}

```
Specifically, the function `doIt()` allocates memory to `output`:
```
void
ESIVariableProcessor::doIt()
{
    assert (output == NULL);

    while (pos < len) {
        /* skipping pre-variables */
        if (string[pos] != '$') {
            ++pos;
        } else {
            if (pos - done_pos)
                /* extract known plain text */
                ESISegment::ListAppend (output, string + done_pos, pos - done_pos);
            done_pos = pos;
            ++pos;
            identifyFunction();
            doFunction();
        }
    }
    /* pos-done_pos chars are ready to copy */
    if (pos-done_pos)
        ESISegment::ListAppend (output, string+done_pos, pos - done_pos);
    safe_free (found_default);
    safe_free (found_subref);
}
```
However, if both `len` and `pos` here are zero, this function will do nothing, and no memory will be allocated to `output`. Thus, the line
```
    char *rv = output->listToChar();
```
will cause a null pointer dereference.

A simple reproducer for the `<esi:when>` case:
```
<l xmlns:esi="http://www.edge-delivery.org/esi/1.0"><esi:when test="">
```
