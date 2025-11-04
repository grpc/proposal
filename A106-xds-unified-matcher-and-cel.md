A106: xDS Unified Matcher and CEL Integration
======

* Author(s): Sergii Tkachenko (@sergiitk)
* Approver: Mark Roth (@markdroth)
* Status: In Review
* Implemented in:
* Last updated: 2025-11-03
* Discussion at:
  - [ ] TODO(sergiitk): insert google group thread

## Abstract

We will add support for the xDS [Unified Matcher API] and
[Common Expression Language](cel.dev) (CEL) within gRPC. This integration will
enable advanced, flexible matching capabilities for various xDS-managed
features, such as server-side rate limiting (RLQS, [A77]), external
authorization (ExtAuthz, [A92]), and external processing (ExtProc, [A93]).

TODO(sergiitk): q: is this really needed for ExtAuthz, ExtProc?

## Background

[Unified Matcher API] is an adaptable framework that can be used in any xDS
component that needs matching features. Historically, xDS filters implemented
its custom mechanisms for performing assertion against response/request
metadata. The Unified Matcher API was introduced to standardize and unify these
matching capabilities across various xDS components.

TODO(sergiitk): finish

### Related Proposals

* [gRFC A41: xDS RBAC Support][A41]
* [gRFC A77: xDS Server-Side Rate Limiting][A77] (WIP)
* [gRFC A92: xDS ExtAuthz Support][A92] (WIP)
* [gRFC A93: xDS ExtProc Support][A93] (WIP)

[A41]: A41-xds-rbac.md
[A77]: https://github.com/grpc/proposal/pull/414
[A92]: https://github.com/grpc/proposal/pull/481
[A93]: https://github.com/grpc/proposal/pull/484

[Unified Matcher API]: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/matching/matching_api.html
[Unified Matcher API Support]: #unified-matcher-api-support
[Unified Matcher: Filter Integration]: #unified-matcher-filter-integration
[Unified Matcher: Input Extensions]: #unified-matcher-input-extensions
[Unified Matcher: Matching Extensions]: #unified-matcher-matching-extensions
[Unified Matcher: `Matcher`]: #unified-matcher-matcher
[Unified Matcher: `OnMatch`]: #unified-matcher-onmatch
[Unified Matcher: `MatcherList`]: #unified-matcher-matcherlist
[Unified Matcher: `MatcherTree`]: #unified-matcher-matchertree
[Unified Matcher: `HttpRequestHeaderMatchInput`]: #unified-matcher-httprequestheadermatchinput
[Unified Matcher: `HttpAttributesCelMatchInput`]: #unified-matcher-httpattributescelmatchinput
[Unified Matcher: `StringMatcher`]: #unified-matcher-stringmatcher
[Unified Matcher: `CelMatcher`]: #unified-matcher-celmatcher

[`cel.expr.CheckedExpr`]: https://github.com/google/cel-spec/blob/master/proto/cel/expr/checked.proto
[CEL Integration]: #cel-integration
[CEL Runtime Restrictions]: #cel-runtime-restrictions
[Supported CEL Variables]: #supported-cel-variables

## Proposal

### Unified Matcher API Support

Envoy provides two syntactically equivalent Unified Matcher definitions:
[`envoy.config.common.matcher.v3.Matcher`](https://github.com/envoyproxy/envoy/blob/e3da7ebb16ad01c2ac7662758a75dba5cdc024ce/api/envoy/config/common/matcher/v3/matcher.proto)
and
[`xds.type.matcher.v3.Matcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto),
which is the preferred version for all new APIs using Unified Matcher. If
`envoy.config.common.matcher.v3.Matcher` is provided, we will interpret it as is
`xds.type.matcher.v3.Matcher`.

In this iteration the following Unified Mather extensions will be supported:

1.  Inputs:
    1.  [Unified Matcher: `HttpRequestHeaderMatchInput`]
    2.  [Unified Matcher: `HttpAttributesCelMatchInput`]
2.  Matchers:
    1.  [Unified Matcher: `StringMatcher`] (standard matcher)
    1.  [Unified Matcher: `CelMatcher`]

#### Unified Matcher: Filter Integration

When implementing Unified Matcher API, a filter must define the following:

-   Supported protocol-specific actions (see [Unified Matcher: `OnMatch`]).
-   Supported [Unified Matcher: Input Extensions].
-   Supported [Unified Matcher: Matching Extensions], including any additional
    limitations on their inputs.
-   Filter-specific default no-match behavior (f.e. xDS resource NACK).

#### Unified Matcher: `Matcher`

While the Unified Matcher API allows for matcher trees of arbitrary depth, gRPC
will reject any matcher definition with a tree depth greater than 100, NACKing
the xDS resource.

We will support the following fields in the
[`xds.type.matcher.v3.Matcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L22)
message:

-   `matcher_type`: One of the following must be present and valid:
    -   [`matcher_list`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L126):
        A valid [Unified Matcher: `MatcherList`] message.
    -   [`matcher_tree`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L129):
        A valid [Unified Matcher: `MatcherTree`] message.
-   [`on_no_match`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L135):
    Specifies the action executed if no match is found in the `matcher_list` or
    `matcher_tree`. If set, must be a valid [Unified Matcher: `OnMatch`].
    If not set, refer to filter's default no-match behavior.

#### Unified Matcher: `OnMatch`

We will support the following fields in the
[`xds.type.matcher.v3.Matcher.OnMatch`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L24)
message:

-   `on_match`: One of the following must be present and valid:
    -   [`matcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L33):
        A nested [Unified Matcher: `Matcher`] for more complex, tree-like
        matching logic.
    -   [`action`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L36):
        A [`TypedExtensionConfig`] containing a protocol-specific action to
        take.

The following fields will be ignored by gRPC:

-   `keep_matching`: Not supported in the initial implementation, may be added
    later.

#### Unified Matcher: `MatcherList`

We will support the following fields in the
[`xds.type.matcher.v3.Matcher.MatcherList`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L43)
message:

-   [`matchers`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L96):
    A list of
    [`Matcher.MatcherList.FieldMatcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L87)
    messages. Must contain at least 1 item.
    -   [`predicate`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L89):
        Must be set and contain a valid
        [`Matcher.MatcherList.Predicate`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L45)
        message.
        -   `match_type`: One of the following must be present and valid:
            -   [`single_predicate`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L73):
                A
                [`Matcher.MatcherList.Predicate.SinglePredicate`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L47)
                message. The return type of the input must match the input type
                of the matcher.
                -   [`input`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L50):
                    A valid [`TypedExtensionConfig`]. Must be present and
                    contain one of the input extensions
                    [supported by the filter][Unified Matcher: Filter Integration].
                    Must have return type compatible with the `matcher`.
                -   `matcher`: One of the following must be present and valid:
                    -   [`value_match`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L56):
                        A valid [Unified Matcher: `StringMatcher`]. Only
                        compatible with `input` that returns a string.
                    -   [`custom_match`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L60):
                        A valid [`TypedExtensionConfig`] containing one of the
                        matching extensions
                        [supported by the filter][Unified Matcher: Filter Integration].
                        Must have input type compatible with the `input`. Must
                        return a boolean indicating the status of the match.
            -   [`or_matcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L76):
                A
                [`Matcher.MatcherList.Predicate.PredicateList`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L65)
                message.
                -   [`predicate`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L66):
                    A list of `Matcher.MatcherList.Predicate` messages. Must
                    contain at least 2 items. Returns true if any of them are
                    true.
            -   [`and_matcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L79):
                A
                [`Matcher.MatcherList.Predicate.PredicateList`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L65)
                message.
                -   [`predicate`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L66):
                    A list of `Matcher.MatcherList.Predicate` messages. Must
                    contain at least 2 items. Returns true if all of them are
                    true.
            -   [`not_matcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L82):
                A nested `Matcher.MatcherList.Predicate` message. Returns the
                inverted result of predicate evaluation.
    -   [`on_match`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L92):
        Must be set and contain a valid [Unified Matcher: `OnMatch`] message.

#### Unified Matcher: `MatcherTree`

We will support the following fields in the
[`xds.type.matcher.v3.Matcher.MatcherTree`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L99)
message:

-   [`input`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L106):
    A valid [`TypedExtensionConfig`]. Must be present and contain one of the
    input extensions
    [supported by the filter][Unified Matcher: Filter Integration]. Must have
    return type compatible with the `matcher`.
-   `tree_type`: One of the following must be present and valid:
    -   [`exact_match_map`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L114):
        A
        [`Matcher.MatcherTree.MatchMap`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L101)
        message. Only compatible with `input` that returns a string.
        -   [`map`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L102):
            A map from a string to a valid [Unified Matcher: `OnMatch`] message.
            Must contain at least 1 pair.
    -   [`prefix_match_map`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L117):
        A
        [`Matcher.MatcherTree.MatchMap`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L101)
        message. Only compatible with `input` that returns a string.
        -   [`map`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L102):
            A map from a string to a valid [Unified Matcher: `OnMatch`] message.
            Must contain at least 1 pair.
    -   [`custom_match`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L120):
        A valid [`TypedExtensionConfig`] containing one of the matching
        extensions
        [supported by the filter][Unified Matcher: Filter Integration]. Must
        have input type compatible with the `input`. Must return a boolean
        indicating the status of the match.

#### Unified Matcher: Input Extensions

##### Unified Matcher: `HttpRequestHeaderMatchInput`

Returns a `string` containing the value of the header with name specified in
`header_name`.

We will support the following fields in the
[`envoy.type.matcher.v3.HttpRequestHeaderMatchInput`](https://github.com/envoyproxy/envoy/blob/7ebdf6da0a49240778fd6fed42670157fde371db/api/envoy/type/matcher/v3/http_inputs.proto#L22)
message:

-   [`header_name`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/matcher.proto#L106):
    Must be present. Value length must be in the range `[1, 16384)`. Must be a
    valid HTTP/2 header name.

##### Unified Matcher: `HttpAttributesCelMatchInput`

Returns a language-specific interface that allows to access request RPC metadata
as defined in [Supported CEL Variables].

We will support the following fields in the
[`xds.type.matcher.v3.HttpAttributesCelMatchInput`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/http_inputs.proto#L22)
message:

-   no fields.

#### Unified Matcher: Matching Extensions

##### Unified Matcher: `StringMatcher`

Compatible with [Unified Matcher: Input Extensions] that return a `string`.

We will support the following fields in the
[`xds.type.matcher.v3.StringMatcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/string.proto#L19)
message:

-   `match_pattern`: One of the following must be present and valid:
    -   [`exact`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/string.proto#L28):
        The input string must match exactly. An empty string is a valid value.
    -   [`prefix`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/string.proto#L36):
        The input string must have this prefix. Must be non-empty.
    -   [`suffix`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/string.proto#L44):
        The input string must have this suffix. Must be non-empty.
    -   [`contains`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/string.proto#L55):
        The input string must contain this substring. Must be non-empty.
-   [`ignore_case`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/string.proto#L65):
    If `true`, the matching is case-insensitive.

The following are not supported by gRPC in the initial implementation and will
result in xDS resource NACK:

-   `safe_regex`
-   `custom`

##### Unified Matcher: `CelMatcher`

Compatible with [Unified Matcher: `HttpAttributesCelMatchInput`].

Performs a match by evaluating a Common Expression Language (CEL) expression.
See [CEL Integration] for details.

We will support the following fields in the
[`xds.type.matcher.v3.CelMatcher`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/cel.proto#L30)
message:

-   [`expr_match`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/cel.proto#L32):
    Must be present and contain a valid
    [`xds.type.v3.CelExpression`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/v3/cel.proto#L26)
    message.
    -   [`cel_expr_checked`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/v3/cel.proto#L49):
        Must be present and contain a valid [`cel.expr.CheckedExpr`] message.
        This message will be converted into a native CEL Abstract Syntax Tree
        (AST) using the language-specific CEL library. The AST's output (return)
        type must be boolean. The resulting CEL program must also be validated
        to conform to [CEL Runtime Restrictions]. If any of these conversion or
        validation steps fail, gRPC will NACK the xDS resource.
-   [`description`](https://github.com/cncf/xds/blob/b4127c9b8d78b77423fd25169f05b7476b6ea932/xds/type/matcher/v3/cel.proto#L36):
    An optional string. May be ignored or used for testing/debugging.

The following fields will be ignored by gRPC:

-   `CelExpression.parsed_expr` - deprecated, only Canonical CEL is supported.
-   `CelExpression.checked_expr` - deprecated, only Canonical CEL is supported.
-   `CelExpression.cel_expr_parsed` - only Checked CEL expressions are
    supported.

### CEL Integration

We will support request metadata matching via CEL expressions. Only Canonical
CEL and only checked expressions will be supported [`cel.expr.CheckedExpr`].

CEL evaluation environment is a set of available variables and extension
functions in a CEL program. We will match
[Envoy CEL environment](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes)
and CEL interpreter configuration.

#### CEL Runtime Restrictions

Certain CEL features can lead to superlinear time complexity or memory
exhaustion. To ensure consistent behavior with Envoy and maintain security,
gRPC will configure the CEL runtime
[similar to Envoy](https://github.com/envoyproxy/envoy/blob/c57801c2afbe26dd6fad7f5ce764f267d07fbd04/source/extensions/filters/common/expr/evaluator.cc#L17-L23):

```c
// Disables comprehension expressions, e.g. exists(), all().
options.enable_comprehension = false;

// Limits the maximum program size for RE2 regex to 100.
options.regex_max_program_size = 100;

// Disables string() overloads.
options.enable_string_conversion = false;

// Disables string concatenation overload.
options.enable_string_concat = false;

// Disables list concatenation overload.
options.enable_list_concat = false;
```

#### Supported CEL Functions

Similar to Envoy, we will
support [standard CEL functions](https://github.com/google/cel-spec/blob/c629b2be086ed6b4c44ef4975e56945f66560677/doc/langdef.md#standard-definitions)
except comprehension-style macros.

| CEL Method                                         | Description                                                                                   |
|----------------------------------------------------|-----------------------------------------------------------------------------------------------|
| `size(x)`                                          | Returns the length of a container x (string, bytes, list, map).                               |
| `x.matches(y)`                                     | Returns true if the string x is partially matched by the specified [RE2][RE2_wiki] pattern y. |
| `x.contains(y)`                                    | Returns true if the string x contains the substring y.                                        |
| `x.startsWith(y)`                                  | Returns true if the string x begins with the substring y.                                     |
| `x.endsWith(y)`                                    | Returns true if the string x ends with the substring y.                                       |
| `timestamp(x)`, `timestamp.get*(x)`, `duration`    | Date/time functions.                                                                          |
| `in`, `[]`                                         | Map/list indexing.                                                                            |
| `has(m.x)`                                         | (macro) Returns true if the map `m` has the string `"x"` as a key.                            |
| `int`, `uint`, `double`, `string`, `bytes`, `bool` | Conversions and identities.                                                                   |
| `==`, `!=`, `>`, `<`, `<=`, `>=`                   | Comparisons.                                                                                  |
| `or`, `&&`, `+`, `-`, `/`, `*`, `%`, `!`           | Basic functions.                                                                              |

[RE2_wiki]: https://en.wikipedia.org/wiki/RE2_(software)

#### Supported CEL Variables

In the initial implementation only the `request` variable is supported in CEL
expressions. We will adapt
[Envoy's Request Attributes](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes#request-attributes)
for gRPC.

| Attribute           | Type                  | gRPC source                  | Envoy Description                                           |
|---------------------|-----------------------|------------------------------|-------------------------------------------------------------|
| `request.path`      | `string`              | Full method name             | The path portion of the URL.                                |
| `request.url_path`  | `string`              | Same as `request.path`       | The path portion of the URL without the query string.       |
| `request.host`      | `string`              | Authority                    | The host portion of the URL.                                |
| `request.scheme`    | `string`              | Not set                      | The scheme portion of the URL.                              |
| `request.method`    | `string`              | `POST`<sup>1</sup>           | Request method.                                             |
| `request.headers`   | `map<string, string>` | `metadata`<sup>2</sup>       | All request headers indexed by the lower-cased header name. |
| `request.referer`   | `string`              | `metadata["referer"]`        | Referer request header.                                     |
| `request.useragent` | `string`              | `metadata["user-agent"]`     | User agent request header.                                  |
| `request.time`      | `timestamp`           | Not set                      | Time of the first byte received.                            |
| `request.id`        | `string`              | `metadata["x-request-id"]`   | Request ID corresponding to `x-request-id` header value     |
| `request.protocol`  | `string`              | Not set                      | Request protocol.                                           |
| `request.query`     | `string`              | `""`                         | The query portion of the URL.                               |

##### Footnotes

**<sup>1</sup> `request.method`**\
Hard-coded to `"POST"` if unavailable and a code audit confirms the server
denies requests for all other method types.

**<sup>2</sup> `request.headers`**\
As defined in [A41], "header" field.

##### CEL Variable Implementation Details

For performance reasons, CEL variables should be resolved on demand. CEL Runtime
provides the different variable resolving approaches based on the language:

* CPP: [`BaseActivation::FindValue()`](https://github.com/google/cel-cpp/blob/9310c4910e598362695930f0e11b7f278f714755/eval/public/base_activation.h#L35)
* Go: [`Activation.ResolveName(string)`](https://github.com/google/cel-go/blob/3f12ecad39e2eb662bcd82b6391cfd0cb4cb1c5e/interpreter/activation.go#L30)
* Java: [`CelVariableResolver`](https://javadoc.io/doc/dev.cel/runtime/0.6.0/dev/cel/runtime/CelVariableResolver.html)

### Temporary Environment Variable Protection

During initial development, this feature will be enabled via
the `GRPC_EXPERIMENTAL_XDS_ENABLE_RLQS` environment variable. This environment
variable protection will be removed once the feature has proven stable.

## Rationale

TODO(sergiitk): rationale

## Implementation

Will be implemented in C-core, Java, Go, and Node as part of either RLQS
([A77]), ExtAuthz ([A92]), or ExtProc ([A93]), whichever happens to be
implemented first in any given language.
