# Buffer Underflow in ESI 
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
However, the issue is that there are no checks whether this name is well-formed before it undergoes parsing, and may, within the context of decimal (twos compliment) encoding. For example, consider the following assignment:
```
<esi:assign name="К" value="gene"/>
```
Note: that `К` is the Cyrillic К, not the Latin letter K.
`К` in hex-encoded UTF-8 is 0xD0 0x9A -- aka. a multi-byte character. We can convert this to decimal (twos complement) "-48 -102". Squid's ESI handling does not correctly handle multi-byte characters nor characters outside the standard ASCII range.
In order to define a variable, the `TrieNode::add` function is used:
```TrieNode::add (this=0x1853510, aString=0x18522c2 "К",theLength=2, privatedata=0x1854650, transform=0x0)```
We can see the definition of this function:
```
TrieNode::add(char const *aString, size_t theLength, void *privatedata, TrieCharTransform *transform)
{
    /* We trust that privatedata and existent keys have already been checked */

    if (theLength) {
        int index = transform ? (*transform)(*aString): *aString;

        if (!internal[index])
            internal[index] = new TrieNode;

        return internal[index]->add(aString + 1, theLength - 1, privatedata, transform);
```
The issue here is that the integer `index` becomes whatever the pointer character `*aString` points to. In the case of standard ASCII, that would be always positive. However,in the case of our Cryllic `К`, the becomes negative: `-48`. If this does not crash (or worse) already, this function will be called once again and `index` will be set to `-102`.  Effectively, memory will be allocated at `internal[-48]` and `internal[-102]` respectively. Oops!
