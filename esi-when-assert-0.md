# Assertion Due to 0 ESI 'when' Checking 
Squid is able to process Edge Side Includes (ESI) pages when it acts as a reverse proxy. By default, this processing is enabled by default, and can be 'triggered' by any page whose response can be manipulated to return a single response header, and content.
ESI is an xml-based markup language which, in layman terms, can be used to cache *specific parts* of a single webpage, thus allowing for the caching of static content on an otherwise dynamically created page.

## The Issue
The ESI syntax includes a directive called `<esi:when>`. This directive is used to test for boolean conditions. For example, ``<esi:when test="$(HTTP_USER_AGENT{'os'})=='iPhone'">`` could be used to check whether a request's user agent is identifying itself as an iPhone.

The issue is that when the condition `0` is tested, a discrepancy in the ESI code handles the character differently. Squid uses the `getsymbol` function to parse the test condition, and determine what type of test is being conducted, such as a logical AND, NOT, LESS THAN, MORE THAN, etc. For the character `0`, Squid interprets this as an integer to be tested against *something*. However, it is eventually passed to the function `ESIExpression::Evaluate`, which in some cases, expects a string literal (which is converted to a ESI_EXPR_EXPR type) (rather than an integer). In this specific case of a single `0`, an assertion will occur. The code for `ESIExpression::Evaluate` is:
```
int
ESIExpression::Evaluate(char const *s)
{
    stackmember stack[ESI_STACK_DEPTH_LIMIT];
    int stackdepth = 0;
    char const *end;
    
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

    /* if we hit here, we think we have a valid result */
    assert(stackdepth == 1);
    assert(stack[0].valuetype == ESI_EXPR_EXPR);
    return stack[0].value.integral ? 1 : 0;
```
Here, we can see that if `stackdepth == 1` to begin with, which is the case for the simple case of `test="0"`, nearly none of this function will actually execute. However, because `0` is interpreted as a an integer, the assertion `assert(stack[0].valuetype == ESI_EXPR_EXPR);` will fail.

A simple reproducer is a reverse-proxied page with the following response:
```
<l xmlns:esi="http://www.edge-delivery.org/esi/1.0"><esi:when test="0">
```
Perhaps the bigger issue here is that testing for 0 is (and should be) a completely valid test: it just simply means always false. This could be for debugging reasons, or there could be some *other* script which is setting this to 0 upstream.
