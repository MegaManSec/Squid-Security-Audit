
# Assertion Using ESI's When Directive 
Squid is able to process Edge Side Includes (ESI) pages when it acts as a reverse proxy. By default, this processing is enabled by default, and can be 'triggered' by any page whose response can be manipulated to return a single response header, and content.
ESI is an xml-based markup language which, in layman terms, can be used to cache *specific parts* of a single webpage, thus allowing for the caching of static content on an otherwise dynamically created page.

## The Issue
The ESI syntax includes a directive called `<esi:when>`. This directive is used to test for boolean conditions. For example, ``<esi:when test="$(HTTP_USER_AGENT{'os'})=='iPhone'">`` could be used to check whether a request's user agent is identifying itself as an iPhone.

The issue is that when the condition `0!0&lt;0` is tested, Squid will crash with an assertion. Note: after this condition is parsed by the XML parser, it becomes `0!0<0`. 
The exact reason is complicated and goes beyond the scope of this document, however the affected (highly edited) code is as follows:
```
Evaluate(char const *s)
{
    stackmember stack[ESI_STACK_DEPTH_LIMIT];
    int stackdepth = 0;
    char const *end;
    while (*s) {
        stackmember candidate = getsymbol(s, &end);
            if (!addmember(stack, &stackdepth, &candidate)) {
                return 0;
            }
    }

    if (stackdepth > 1) {
        stackmember rv;
        rv.valuetype = ESI_EXPR_INVALID;
        rv.precedence = 0;

        if (stack[stackdepth - 2].
                eval(stack, &stackdepth, stackdepth - 2, &rv)) {
            /* special case - leading operator failed */
            debugs(86, DBG_IMPORTANT, "invalid expression");
            PROF_stop(esiExpressionEval);
            return 0;
        }
    }

    if (stackdepth == 0) {
        /* Empty expression - evaluate to false */
        PROF_stop(esiExpressionEval);
        return 0;
    }
    assert(stackdepth == 1);
} 

```
Of note, we can see see that if `stackdepth` is greater than 1, the function `stack[stackdepth - 2].eval(stack, &stackdepth, stackdepth - 2, &rv` is called. However, if this returns false (AKA the expression is **valid**), the value of `stackdepth` is not updated, and thus, we simply move towards the final line which will cause an assertion.

As mentioned, the exact reason for this code is difficult to understand. But the point is, it assumes that an expression can only hold a maximum of 2 evaluations, rather than more than 0.

An example which will cause this assertion is as follows:
```
<l xmlns:esi="http://www.edge-delivery.org/esi/1.0"><esi:when test="0!0&lt;0">
```
