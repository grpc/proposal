A106: xDS Unified Matcher and CEL Integration
======

* Author(s): Sergii Tkachenko (@sergiitk)
* Approver: Mark Roth (@markdroth)
* Status: In Review
* Last updated: 2025-11-20
* Discussion at: TODO(sergiitk): insert google group thread

## Abstract

We will add support for the xDS [Unified Matcher API] and
[Common Expression Language] (CEL) within gRPC. This integration will enable
advanced, flexible matching capabilities for various xDS-managed features, such
as server-side rate limiting (RLQS, [A77]), Role Based Access Control (RBAC,
[A41]), and Composite Filter ([A103]).

## Background

[Unified Matcher API] is an adaptable framework that can be used in any xDS
component that needs matching features. Historically, xDS filters implemented
its custom mechanisms for performing assertion against response/request
metadata. The Unified Matcher API was introduced to standardize and unify these
matching capabilities across various xDS components.

[Common Expression Language](CEL) is an open-source, non-Turing complete
expression language designed for evaluating expressions quickly and safely. It
is commonly used in authorization, policy enforcement, and data validation
scenarios. CEL expressions are evaluated against a set of input variables and
can perform operations such as comparisons, logical operations, string
manipulations, and map/list indexing.

CEL is particularly well-suited for xDS because it allows the control plane to
push user-defined matching logic to gRPC clients, while CEL's non-Turing
complete nature ensures safe and predictable execution.

The Unified Matcher API is designed to be extensible, allowing for different
types of inputs and matching logic to be plugged in. CEL integration is achieved
through this extension mechanism, where CEL expressions can be used as a
powerful, flexible and safe custom matcher. This allows for complex, dynamic
request matching based on a wide range of request attributes.

### Related Proposals

*   [gRFC A41: xDS RBAC Support][A41]
*   [gRFC A77: xDS Server-Side Rate Limiting][A77] (WIP)
*   [gRFC A103: xDS Composite Filter][A103] (WIP)

[A41]: A41-xds-rbac.md
[A77]: https://github.com/grpc/proposal/pull/414
[A103]: https://github.com/grpc/proposal/pull/511

[Unified Matcher API]: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/matching/matching_api.html
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

[Common Expression Language]: https://cel.dev
[`cel.expr.CheckedExpr`]: https://github.com/google/cel-spec/blob/master/proto/cel/expr/checked.proto

[CEL Integration]: #cel-integration
[CEL: `CelExpression`]: #cel-celexpression
[CEL: Runtime Restrictions]: #cel-runtime-restrictions
[CEL: Supported Functions]: #cel-supported-functions
[CEL: Supported Variables]: #cel-supported-variables

[`StringValue`]: https://protobuf.dev/reference/protobuf/google.protobuf/#string-value
[`TypedExtensionConfig`]: https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/core/v3/extension.proto#L14
[RE2_wiki]: https://github.com/google/re2/wiki/Syntax

## Proposal

### Unified Matcher

#### Unified Matcher: Core Concepts

The Unified Matcher API revolves around a few key concepts:

1.  **Matcher**: A matcher is a rule that evaluates to true or false based on
   some properties of the input data. Matchers can be simple (e.g., checking if
   a header has a specific value) or complex (e.g., a boolean combination of
   other matchers, nested matchers, etc).
2.  **Matcher Action**: If a matcher evaluates to true, an associated action is
   taken. This action could be anything from selecting a route to applying a
   filter.
3.  **Matcher Input**: This extracts the data from the Matcher Context and
   provides it to the matchers evaluate. For example, it may get a specific
   header from the request provided in the matcher context.
4.  **Matcher Context**: This holds the input data and any other contextual
   information needed during the matching process. It may include information
   about the request being processed, the response, connection info, a
   combination of thereof, additional parameters to mathers that support it,
   etc.

#### Unified Matcher: Filter Integration

When implementing Unified Matcher API, a filter must define the following for
each [Unified Matcher: `Matcher`] field in its config:

1.  Supported protocol-specific actions.
2.  Supported [Unified Matcher: Input Extensions].
3.  Filter-specific behavior for unsuccessful matches.

Note that the selection of input extensions defines the
[Unified Matcher: Matching Extensions] that can be used with the filter.

##### Protocol-Specific Actions

Protocol-specific actions are used in
[`OnMatch.action`][Unified Matcher: `OnMatch`] and may be any protocol-specific
message packed into [`TypedExtensionConfig`].

The filter implementing the Unified Matcher API must define the set of
protocol-specific actions it supports. If an action is not supported, gRPC will
NACK the xDS resource.

Upon a successful match, the matched action will be returned as the result of
the matcher tree evaluation.

##### Behavior for Unsuccessful Matches

The match is considered unsuccessful:

1.  If no match found after evaluating the [Unified Matcher: `Matcher`] AND
2.  `on_no_match` field is unset OR its evaluation is unsuccessful.

The filter may define any behavior for an unsuccessful match, f.e. NACK the xDS
resource, failing open/closed, etc.

#### Unified Matcher: `Matcher`

While the Unified Matcher API allows for matcher trees of arbitrary depth, gRPC
will reject any matcher definition with a tree depth greater than `16`, NACKing
the xDS resource.

Envoy provides two syntactically equivalent Unified Matcher definitions:
[`envoy.config.common.matcher.v3.Matcher`](https://github.com/envoyproxy/envoy/blob/426cd861187368163b42fce910ab5828f7f0b392/api/envoy/config/common/matcher/v3/matcher.proto)
and
[`xds.type.matcher.v3.Matcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto),
which is the preferred version for all new APIs using Unified Matcher. We will
produce the same form for either one.

We will support the following `Matcher` fields:

-   `matcher_type`: One of the following must be present:
    -   [`matcher_list`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L134)
        ([Unified Matcher: `MatcherList`]).
    -   [`matcher_tree`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L137)
        ([Unified Matcher: `MatcherTree`]).
-   [`on_no_match`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L143):
    ([Unified Matcher: `OnMatch`]): Specifies the action executed if no match is
    found in the `matcher_list` or `matcher_tree`. If not set, refer to filter's
    [unsuccessful match behavior][Unified Matcher: Filter Integration].

#### Unified Matcher: `OnMatch`

We will support the following fields in the
[`xds.type.matcher.v3.Matcher.OnMatch`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L24)
message:

-   `on_match`: One of the following must be present:
    -   [`matcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L33)
        ([Unified Matcher: `Matcher`]): A nested matcher that allows for
        building more complex, tree-like matching logic.
    -   [`action`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L36)
        ([`TypedExtensionConfig`]): If set, must contain one of the
        protocol-specific actions
        [supported by the filter][Unified Matcher: Filter Integration].
-   [`keep_matching`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L45)
    (bool): If this field is set in a context in which it's not supported, the
    xDS resource will be NACKed.

#### Unified Matcher: `MatcherList`

[`Matcher.MatcherList.Predicate`]: https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L53
[`Matcher.MatcherList.Predicate.PredicateList`]: https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L73

We will support the following fields in the
[`xds.type.matcher.v3.Matcher.MatcherList`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L51)
message:

-   [`matchers`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L104)
    (repeated
    [`Matcher.MatcherList.FieldMatcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L95)):
    Must contain at least 1 item.
    -   [`predicate`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L97)
        ([`Matcher.MatcherList.Predicate`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L53)):
        Must be present.
        -   `match_type`: One of the following must be present:
            -   [`single_predicate`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L81)
                ([`Matcher.MatcherList.Predicate.SinglePredicate`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L55)):
                If set, the return type of the `input` must match the input type
                of the `matcher`.
                -   [`input`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L58)
                    ([`TypedExtensionConfig`]): Must be present and contain one
                    of the input extensions
                    [supported by the filter][Unified Matcher: Filter Integration].
                    Must have return type compatible with the `matcher`.
                -   `matcher`: One of the following must be present:
                    -   [`value_match`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L64)
                        ([Unified Matcher: `StringMatcher`]): Only compatible
                        with `input` that returns a string.
                    -   [`custom_match`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L68)
                        ([`TypedExtensionConfig`]): Must contain one of the
                        matching extensions
                        [supported by the filter][Unified Matcher: Filter Integration].
                        Must be compatible with the return type of the `input`.
            -   [`or_matcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L84)
                ([`Matcher.MatcherList.Predicate.PredicateList`]):
                -   [`predicate`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L74)
                    (repeated [`Matcher.MatcherList.Predicate`]): Must contain
                    at least 2 items. Returns true if any of them are true.
            -   [`and_matcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L87)
                ([`Matcher.MatcherList.Predicate.PredicateList`]):
                -   [`predicate`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L74)
                    (repeated [`Matcher.MatcherList.Predicate`]): Must contain
                    at least 2 items. Returns true if all of them are true.
            -   [`not_matcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L90)
                ([`Matcher.MatcherList.Predicate`]): Returns the inverted result
                of predicate evaluation.
    -   [`on_match`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L100)
        ([Unified Matcher: `OnMatch`]): Must be present.

#### Unified Matcher: `MatcherTree`

[`Matcher.MatcherTree.MatchMap`]: https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L109

We will support the following fields in the
[`xds.type.matcher.v3.Matcher.MatcherTree`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L107)
message:

-   [`input`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L114)
    ([`TypedExtensionConfig`]): Must be present and contain one of the input
    extensions
    [supported by the filter][Unified Matcher: Filter Integration]. Must have
    return type compatible with each matcher specified in the tree.
-   `tree_type`: One of the following must be present:
    -   [`exact_match_map`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L122)
        ([`Matcher.MatcherTree.MatchMap`]): Only compatible with `input` that
        returns a `string`.
        -   [`map`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L110)
            A map from a `string` to [Unified Matcher: `OnMatch`]. Must contain
            at least 1 pair.
    -   [`prefix_match_map`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L125)
        ([`Matcher.MatcherTree.MatchMap`]): Only compatible with `input` that
        returns a `string`.
        -   [`map`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/matcher.proto#L110)
            A map from a `string` to [Unified Matcher: `OnMatch`]. Must contain
            at least 1 pair.

The following are not supported by gRPC in the initial implementation and will
result in xDS resource NACK:

-   `custom_match`

#### Unified Matcher: Input Extensions

In this iteration, the following Unified Mather extensions will be supported:

1.  [Unified Matcher: `HttpRequestHeaderMatchInput`]
2.  [Unified Matcher: `HttpAttributesCelMatchInput`]

##### Unified Matcher: `HttpRequestHeaderMatchInput`

Returns a `string` containing the value of the header with name specified in
`header_name`.

We will support the following fields in the
[`envoy.type.matcher.v3.HttpRequestHeaderMatchInput`](https://github.com/envoyproxy/envoy/blob/426cd861187368163b42fce910ab5828f7f0b392/api/envoy/type/matcher/v3/http_inputs.proto#L22)
message:

-   [`header_name`](https://github.com/envoyproxy/envoy/blob/426cd861187368163b42fce910ab5828f7f0b392/api/envoy/type/matcher/v3/http_inputs.proto#L24):
    Must be present. Value length must be in the range `[1, 16384)`. Must be a
    valid HTTP/2 header name.

##### Unified Matcher: `HttpAttributesCelMatchInput`

Returns a language-specific interface that allows to access request RPC metadata
as defined in [CEL: Supported Variables].

We will support the following fields in the
[`xds.type.matcher.v3.HttpAttributesCelMatchInput`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/http_inputs.proto#L22)
message:

-   no fields.

#### Unified Matcher: Matching Extensions

In this iteration, the following Unified Mather extensions will be supported:

1.  [Unified Matcher: `StringMatcher`](standard matcher)
2.  [Unified Matcher: `CelMatcher`]

##### Unified Matcher: `StringMatcher`

Compatible with [Unified Matcher: Input Extensions] that return a `string`.

We will support the following fields in the
[`xds.type.matcher.v3.StringMatcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/string.proto#L19)
message:

-   `match_pattern`: One of the following must be present:
    -   [`exact`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/string.proto#L28):
        The input string must match exactly. An empty string is a valid value.
    -   [`prefix`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/string.proto#L36):
        The input string must have this prefix. Must be non-empty.
    -   [`suffix`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/string.proto#L44):
        The input string must have this suffix. Must be non-empty.
    -   [`safe_regex`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/string.proto#L47):
        ([`RegexMatcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/regex.proto#L15))
        The input string must match the regular expression.
        -   [`google_re2`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/regex.proto#L40):
            Effectively ignored, [Google's RE2][RE2_wiki] is the only supported
            implementation.
        -   [`regex`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/regex.proto#L45)
            Must be non-empty.
    -   [`contains`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/string.proto#L55):
        The input string must contain this substring. Must be non-empty.
-   [`ignore_case`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/string.proto#L65):
    If `true`, the matching is case-insensitive. Does not apply to the
    `safe_regex` match type.

The following are not supported by gRPC and will result in xDS resource NACK:

-   `custom`

The following fields are ignored:

-   `RegexMatcher.google_re2`

##### Unified Matcher: `CelMatcher`

Compatible with [Unified Matcher: `HttpAttributesCelMatchInput`].

Performs a match by evaluating a [Common Expression Language] (CEL) expression.
See [CEL Integration] for details.

We will support the following fields in the
[`xds.type.matcher.v3.CelMatcher`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/cel.proto#L30)
message:

-   [`expr_match`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/cel.proto#L32)
    ([`xds.type.v3.CelExpression`][CEL: `CelExpression`]): Must be present.
    This message will be converted into a native CEL Abstract Syntax Tree (AST)
    using the language-specific CEL library. The AST's output (return) type must
    be boolean. The resulting CEL program must also be validated to conform to
    [CEL: Runtime Restrictions] and only contain [CEL: Supported Variables]. If
    the conversion or the validation step fail, gRPC will NACK the xDS resource.
-   [`description`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/matcher/v3/cel.proto#L36):
    An optional string. May be ignored or used for testing/debugging.

#### Unified Matcher: Evaluation Flow

**Goal:** To produce a list of matching `Action`s.\
**Matcher Context:** The input to the Matcher evaluation tree. Provided at
runtime by the filter, contains the input data. May contain any other contextual
information relevant to the filter. Implementation note: when setting an input,
consider memory footprint. For example, instead of resolving all headers in
advance, provide them in a lazy-loading wrapper.
**Matching Process:** Starting with a top-level `Matcher`, there will be one
of the following matchers types:

-   **[List Matcher][Unified Matcher: `MatcherList`]:** This is like a series of
    "if-elif-elif-else" statements. It goes through a list of rules:
    -   Each rule has a **Condition** (`Predicate`) and a **Result** (`OnMatch`).
    -   **Condition Checking:** To check a typical condition:
        1.  Using the `input` extension, extract a specific piece of data from
            the **Matcher Context** (e.g., the value of requests's `:host`
            header).
        2.  Using the `matcher` extension, compare the extracted data against
            the matcher's criteria (e.g., "is it equal to `'example.com'`?",
            "does it start with `'api.'`?").
    -   Conditions can be combined using AND, OR, NOT, or nested via `OnMatch`.
    -   **First Match Wins:** The *first* rule whose **Condition** is true has
        its **Result** executed.

-   **[Exact Map Matcher][Unified Matcher: `MatcherTree`]:** This is like a
    switch statement or dictionary lookup.
    -   Using the `input` extension, it extracts a specific string value from
        the **Matcher Context**.
    -   It looks this the key for this exact string the a predefined map.
    -   If found, it executes the corresponding **Result**.

-   **[Prefix Map Matcher][Unified Matcher: `MatcherTree`]:** Similar to the Map
    Matcher, but uses prefix matching (a Trie data structure).
    -   Using the `input` extension, it extracts a specific string value from
        the **Matcher Context**.
    -   It finds all entries in the map whose keys are prefixes of the input
        string.
    -   The **Result** is chosen based the the key with the *longest* matching
        prefix.
    -   Undefined behavior: If there are multiple prefixes of the same greatest
        length, their **Result**s are all processed in order of Trie traversal.
        There's no other tie-breaking rule like alphabetical order among the
        tied keys.

**[Result][Unified Matcher: `OnMatch`]:** When a match occurs, the `OnMatch`
dictates the outcome:

-   It can contain an `Action` to be added to the results.
-   It can contain a nested `Matcher`, triggering a further round of matching.
    -   The tree is validated to not contain matchers with the tree depth
         greater than `16`. If this three depth is reached at runtime,
        the tree evaluation is terminated, and considered an
        [unsuccessful match][Unified Matcher: Filter Integration].
-   `keep_matching` determines whether a successful match within an `OnMatch`
    should be considered "terminal" for the current `Matcher` being evaluated.
    It essentially answers the question: "After processing this `OnMatch`,
    should the current matcher stop looking for more matches, or continue?"
    -   If `keep_matching` is `false` (the usual case), finding this `OnMatch`
       is terminal. The `Action` is added (or the nested `Matcher` is
       evaluated), and the current matcher stops searching.
    -   If `keep_matching` is `true`, the `Action` is added (or nested
       `Matcher` evaluated), but the current matcher *continues* to look for
       more matches. The overall process is not considered complete until an
       `OnMatch` with `keep_matching` set to `false` is encountered.

**Default/No Match:** Any `Matcher` can have a default `OnMatch` to use if none
of its primary conditions or map lookups succeed. See details in
[Unified Matcher: `Matcher`] and [Unified Matcher: Filter Integration].\
**The Output:** A list of `Action`s accumulated from all triggered `OnMatch`
results. Generally, only a single `Action` will be returned, unless
`keep_matching` is enabled and multiple matches found.

#### Unified Matcher: Evaluation Examples

For simplicity:

-   `TypedExtensionConfig` fields: The type is captured in a comment, and the
    value directly contains the unpacked message content.
-   `on_match` action: Represented by a string, for example,
    `"on_match": { "action": "route_to_cluster_A" }`.
-   The first example will include `input` to demonstrate the data flow.
-   Other examples will skip the input and simply indicates the result of
    evaluation in `custom_match` field, for example, `{ "custom_match": true }`.

For even more examples, refer to
[Envoy's Unified Matcher API Documentation][Unified Matcher API] (note that
these examples might be Envoy-specific).

##### Example 1: Simple Linear Match

This example shows a basic matcher list. It routes requests based on the value
of a single header, the first matching predicate wins.

**Configuration:**

```json5
{
  "matcher_list": {
    "matchers": [
      {
        "predicate": {
          "single_predicate": {
            // envoy.type.matcher.v3.HttpRequestHeaderMatchInput
            "input": { "header_name": "x-user-segment" },
            "value_match": { "exact": "premium" }
          }
        },
        "on_match": { "action": "route_to_premium_cluster" }
      },
      {
        "predicate": {
          "single_predicate": {
            // envoy.type.matcher.v3.HttpRequestHeaderMatchInput
            "input": { "header_name": "x-user-segment" },
            "value_match": { "prefix": "standard-" }
          }
        },
        "on_match": { "action": "route_to_standard_cluster" }
      }
    ]
  },
  "on_no_match": { "action": "route_to_default_cluster" }
}
```

**Request Input 1:**

-   Headers: `{ "x-user-segment": "standard-user-1" }`

**Evaluation (detailed):**

1.  The `matcher_list` evaluates its matchers in order.
2.  The first matcher is evaluated.
    -   The `input` executes `HttpRequestHeaderMatchInput` extension:
        -   The extension logic extracts the value of the `x-user-segment`
            header from the Matcher Context.
        -   The `input` returns `standard-user-1`.
    -   The `StringMatcher` is evaluated (standard matcher):
        -   The input is a string `standard-user-1`, which is the correct input
            type for this matcher.
        -   The `StringMatcher` checks if the value `standard-user-1` has
            the exact value `premium`.
        -   The result of matcher evaluation is `false`
    -   The predicate is `false`, matching continues.
3.  The second matcher is evaluated:
    -   The `input` executes `HttpRequestHeaderMatchInput` extension:
        -   The extension logic extracts the value of the `x-user-segment`
            header from the Matcher Context.
        -   The `input` returns `standard-user-1`.
    -   The `StringMatcher` is evaluated (standard matcher):
        -   The input is a string `standard-user-1`, which is the correct input
            type for this matcher.
        -   The `StringMatcher` checks if the value `standard-user-1` has the
            prefix `standard-`.
        -   The result of matcher evaluation is `true`
    -   The predicate is `true`, its `on_match` is evaluated.
        -   The action `route_to_standard_cluster` is chosen.
        -   The `matcher_list` stops processing further matchers because
            `keep_matching` is not set.

**Result 1:** `["route_to_standard_cluster"]`.

**Request Input 2:**

-   Headers: `{ "x-user-segment": "guest" }`

**Evaluation (simplified):**

1.  The `matcher_list` evaluates its matchers in order.
2.  The first matcher for `premium` is `false`.
3.  The second matcher for `standard-` prefix is `false`.
4.  No matchers in the list evaluated to `true`, therefore the `on_no_match` is
    evaluated:
    -   The action `route_to_default_cluster` is chosen.

**Result 2:** ["route_to_default_cluster"]

##### Example 2: Keep Matching

This example demonstrates the effect of `keep_matching: true`. Actions are
accumulated until a matcher with `keep_matching: false` (the default) is found.

**Configuration:**

```json5
{
  "matcher_list": {
    "matchers": [
      // Matcher 1
      {
        "predicate": { "single_predicate": { "custom_match": true } },
        "on_match": { "action": "action_1", "keep_matching": true }
      },
      // Matcher 2
      {
        "predicate": { "single_predicate": { "custom_match": false } },
        "on_match": { "action": "action_2" }
      },
      // Matcher 3
      {
        "predicate": { "single_predicate": { "custom_match": true } },
        "on_match": { "action": "action_3" }
      },
      // Matcher 4
      {
        "predicate": { "single_predicate": { "custom_match": false } },
        "on_match": { "action": "action_4" }
      }
    ]
  }
}
```

**Evaluation:**

1.  Matcher 1 evaluates to `true`.
    -   `action_1` is added to the result list.
    -   Matching continues because `keep_matching: true`.
2.  Matcher 2 evaluates to `false`.
3.  Matcher 3 evaluates to `true`.
    -   `action_3` is added to the result list.
    -   Matching stops because `keep_matching` is false by default.
4.  Matcher 4 is not evaluated.

**Result:** `["action_1", "action_3"]`

##### Example 3: Nested Matcher

This example shows a matcher whose action is another matcher.

**Configuration:**

```json5
{
  "matcher_list": {
    "matchers": [
      {
        "predicate": { "single_predicate": { "custom_match": true } },
        "on_match": {
          // Nested matcher with matcher_list.
          "matcher": {
            "matcher_list": {
              "matchers": [
                {
                  "predicate": { "single_predicate": { "custom_match": false } },
                  "on_match": { "action": "inner_matcher_1" }
                },
                {
                  "predicate": { "single_predicate": { "custom_match": true } },
                  "on_match": { "action": "inner_matcher_2" }
                }
              ]
            }
          }
          // Nested matcher end.
        }
      }
    ]
  }
}
```

**Evaluation:**

1.  The outer `matcher_list` evaluates its first (and only) matcher:
    -   The predicate of the outer matcher evaluates to `true`.
    -   The `on_match` of the outer matcher contains a nested matcher.
    -   The three depth is not greater than 16, the nested matcher is evaluated:
        1.  The inner `matcher_list` evaluates first matcher to `false`.
        2.  The inner `matcher_list` evaluates its second matcher to `true`:
            -   The predicate is `true`, its `on_match` is evaluated.
            -   The action `inner_match_2` is added to the result list.
            -   Evaluation stops because `keep_matching` is not set.

**Result 1:** `["inner_matcher_2"]`

##### Example 4: Prefix Map Matcher

This shows a prefix map, where the longest prefix match wins.

**Configuration:**

```json5
{
  "matcher_prefix_map": {
    // envoy.type.matcher.v3.HttpRequestHeaderMatchInput
    "input": { "header_name": "x-user-segment" },
    "map": {
      "grpc": { "action": "shorter_prefix" },
      "grpc.channelz": { "action": "longer_prefix" }
    }
  }
}
```

**Request Input:**

-   Path: `grpc.channelz.v1.Channelz/GetTopChannels`

**Evaluation:**

1.  The input path `grpc.channelz.v1.Channelz/GetTopChannels` is checked against
    the map keys.
2.  It matches both `grpc` and `grpc.channelz`.
3.  The longest matching prefix is `grpc.channelz`.
    -   The action `longer_prefix` is chosen.

**Result 1:** `["longer_prefix"]`

---

### CEL Integration

We will support request metadata matching via CEL expressions.

CEL evaluation environment is a set of available variables and extension
functions in a CEL program. We will match
[Envoy CEL environment](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes)
and CEL interpreter configuration.

#### CEL: `CelExpression`

[`xds.type.v3.CelExpression`]: https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/v3/cel.proto#L26

CEL expressions will be provided by the xDS Control Plane in the
[`xds.type.v3.CelExpression`] message, which allows to specify CEL Abstract
Syntax Tree (AST) in different forms (e.g., `googleapis` or canonical, and each
may be either parsed or checked). We will only support one form: type-checked
Canonical CEL, specifically the [`cel.expr.CheckedExpr`] message.

We will support the following fields in the [`xds.type.v3.CelExpression`]
message:

-   [`cel_expr_checked`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/v3/cel.proto#L49)
    ([`cel.expr.CheckedExpr`]): Must be present.

The following fields will be ignored by gRPC:

-   `parsed_expr` - deprecated, only Canonical CEL is supported.
-   `checked_expr` - deprecated, only Canonical CEL is supported.
-   `cel_expr_parsed` - only Checked CEL expressions are supported.

The following are not supported by gRPC in the initial implementation and will
result in xDS resource NACK:

-   `cel_expr_string`

#### CEL: `CelExtractString`

`CelExtractString` is a small tool that allows to extract a string from
[CEL: Supported Variables] using a CEL expression. The expression must evaluate
to a `string`.

We will support the following fields in the
[`xds.type.v3.CelExtractString`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/v3/cel.proto#L69)
message:

-   [`expr_extract`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/v3/cel.proto#L72)
    ([`xds.type.v3.CelExpression`][CEL: `CelExpression`]): Must be present.
    This message will be converted into a native CEL Abstract Syntax Tree (AST)
    using the language-specific CEL library. The AST's output (return) type must
    be a `string`. It may only contain [CEL: Supported Functions] and
    [CEL: Supported Variables]. The resulting CEL program must also be validated
    to conform to [CEL: Runtime Restrictions]. If the conversion or the
    validation step fail, gRPC will NACK the xDS resource.
-   [`default_value`](https://github.com/cncf/xds/blob/2ac532fd44436293585084f8d94c6bdb17835af0/xds/type/v3/cel.proto#L76):
    ([`StringValue`]) Optional. If set, and the CEL expression evaluates to an
    error or a non-string type, this default value will be returned instead.

#### CEL: Runtime Restrictions

Certain CEL features can lead to superlinear time complexity or memory
exhaustion. To ensure consistent behavior with Envoy and maintain security, gRPC
will configure the CEL runtime
[similar to Envoy](https://github.com/envoyproxy/envoy/blob/c57801c2afbe26dd6fad7f5ce764f267d07fbd04/source/extensions/filters/common/expr/evaluator.cc#L17-L23):

```cpp
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

#### CEL: Supported Functions

Similar to Envoy, we will support
[standard CEL functions](https://github.com/google/cel-spec/blob/c629b2be086ed6b4c44ef4975e56945f66560677/doc/langdef.md#standard-definitions)
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

#### CEL: Supported Variables

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

**<sup>1</sup> `request.method`** \
Hard-coded to `"POST"` if unavailable and a code audit confirms the server
denies requests for all other method types.

**<sup>2</sup> `request.headers`** \
As defined in [A41], "header" field.

##### CEL: Variable Implementation Details

For performance reasons, CEL variables should be resolved on demand. CEL Runtime
provides the different variable resolving approaches based on the language:

-   CPP:
    [`BaseActivation::FindValue()`](https://github.com/google/cel-cpp/blob/9310c4910e598362695930f0e11b7f278f714755/eval/public/base_activation.h#L35)
-   Go:
    [`Activation.ResolveName(string)`](https://github.com/google/cel-go/blob/3f12ecad39e2eb662bcd82b6391cfd0cb4cb1c5e/interpreter/activation.go#L30)
-   Java:
    [`CelVariableResolver`](https://javadoc.io/doc/dev.cel/runtime/0.6.0/dev/cel/runtime/CelVariableResolver.html)

#### CEL: Unified Matcher Example

CEL will be integrated into the Unified Matcher API like so:

```textproto
matcher_list {
  matchers {
    predicate {
      single_predicate {
        input {
          typed_config: {
            [type.googleapis.com/xds.type.matcher.v3.HttpAttributesCelMatchInput] {}
          }
        }
        custom_match: {
          typed_config: {
            [type.googleapis.com/xds.type.matcher.v3.CelMatcher] {
              expr_match: {
                cel_expr_checked: {
                  # Checked CEL AST here.
                }
              }
            }
          }
        }
      }
    }
    on_match: {
      # Action on successful match.
    }
  }
}
```

### Temporary Environment Variable Protection

The Unified Matcher API feature will not be guarded by a dedicated environment
variable. The environment variable protection will be handled by the features
that depend on it (e.g., `GRPC_EXPERIMENTAL_XDS_ENABLE_RLQS` for RLQS).

## Rationale

The chosen approach, integrating the xDS Unified Matcher API with CEL, was
selected over several alternatives due to its advantages in consistency,
maintainability, safety, and alignment with the broader xDS ecosystem.

### Considered Alternatives

The main alternative is for each xDS feature (e.g., RLQS, RBAC) to define its
own custom matching logic. This is how older xDS features were designed, but it
leads to significant drawbacks:

-   Duplication and Inconsistency: It forces repeated implementation of
    common matching primitives (header, path, etc.) across different filters and
    gRPC language implementations, leading to code bloat and subtle behavioral
    differences.
-   High Maintenance Cost: Bug fixes and new features must be implemented in
    multiple places.

The Unified Matcher API provides a single, consistent, and reusable framework
that is implemented once and shared by all features. This improves
maintainability, provides a uniform configuration experience for users, and
aligns gRPC with Envoy, creating a more cohesive xDS ecosystem.

### Disadvantages and Trade-offs

-   **Increased Complexity**: The system is powerful but also complex. The
    matcher API involves nested structures, different evaluation flows
    (`MatcherList` vs. `MatcherTree`), and nuanced behaviors like
    `keep_matching`. Debugging this can be more challenging than simpler
    matching schemes.
-   **Restricted CEL Functionality**: To ensure safety and performance, the
    implementation explicitly disables certain CEL features, such as
    comprehensions (`exists()`, `all()`). This is a direct trade-off of power
    for safety, meaning not all standard CEL capabilities are available.
-   **Performance Overhead**: Evaluating CEL expressions for every request
    introduces computational overhead. While designed to be fast, it will be
    slower than simple, hard-coded logic or basic string comparisons.

## Implementation

Will be implemented in C-core, Java, Go, and Node as part of either RLQS
([A77]) or Composite Filter ([A103]), whichever happens to be implemented first
in any given language. Role Based Access Control (RBAC, [A41]) currently does
not support the Unified Matcher API in gRPC, though it is supported by Envoy,
but it may be added in the future.
