---
title: The CDN-Control HTTP Response Header Field
abbrev: CDN-Control
docname: draft-cdn-control-header-latest
date: {NOW}
category: info

ipr: trust200902
area: General
workgroup: HTTP
keyword: Internet-Draft

stand_alone: yes
pi: [toc, tocindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
 -
    ins: S. Ludin
    name: Stephen Ludin
    organization: Akamai
 -
    ins: M. Nottingham
    name: Mark Nottingham
    organization: Fastly
    street: made in
    city: Prahran
    region: VIC
    country: Australia
    email: mnot@mnot.net
    uri: https://www.mnot.net/
 -
    ins: Y. Wu
    name: Yuchen Wu
    organization: Cloudflare

normative:
  RFC2119:

informative:


--- abstract

This specification defines an HTTP header field that conveys HTTP cache directives to gateway caches.

--- note_Note_to_Readers

*RFC EDITOR: please remove this section before publication*

The issues list for this draft can be found at <https://github.com/cdn-specs/control-header/issues>.

The most recent (often, unpublished) draft is at <https://cdn-specs.github.io/control-header/>.

Recent changes are listed at <https://github.com/cdn-specs/control-header/commits/main>.

See also the draft's current status in the IETF datatracker, at
<https://datatracker.ietf.org/doc/draft-cdn-control-header/>.

--- middle

# Introduction

Many HTTP origin servers use gateway caches to speed up distributing their content, either locally (sometimes called a "reverse proxy cache") or in a distributed fashion ("Content Delivery Networks").

While HTTP defines Cache-Control as a means of controlling cache behaviour for both private caches and gateway caches, it is often desirable to give gateway caches separate instructions. To meet this need, this specification defines a separate header field that conveys HTTP cache directives to gateway caches only.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT",
"RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{!RFC2119}} {{!RFC8174}} when, and only when, they appear in all capitals, as
shown here.



# The CDN-Control Response Header Field

The CDN-Control response header field allows origin servers to control the behaviour of gateway caches interposed between them and clients, separately from other caches that might handle the response.

It is a Dictionary Structured Header {{!I-D.ietf-httpbis-header-structure}}, whose members are cache-directives defined in {{!RFC7234}}. In practice, they can be any directive registered in the HTTP Cache Directive Registry <https://www.iana.org/assignments/http-cache-directives/http-cache-directives.xhtml>.

For gateway caches, when a valid CDN-Control is present in a response, it MUST take precedence over Cache-Control and Expires headers. In other words, CDN-Control disables all cache directives in other header fields, and is a wholly separate way to control the cache. Note that this is on a response-by-response basis; if CDN-Control is not present, caches MAY fall back to other control mechanisms as required by HTTP {{!I-D.ietf-httpbis-cache}}.

The semantics and precedence of cache directives in CDN-Control are the same as those in Cache-Control. In particular, no-store and no-cache make max-age inoperative.

Caches that use CDN-Control MUST implement the semantics of the following directives:

* max-age
* must-revalidate
* no-store
* no-cache
* private

Gateway caches that used CDN-Control MAY forward this header so that downstream gateway caches can use it as well. However, doing so exposes its value to all downstream clients, which might be undesirable. As a result, gateways that process this header field MAY remove it (for example, when configured to do so because it is known not to be used downstream).

A proxy that does not implement caching MUST pass the CDN-Control header through.

A gatway cache that does not use CDN-Control MUST pass the CDN-Control header through.

Private caches SHOULD ignore the CDN-Control header field.

## Examples

For example, the following header fields would instruct a gateway cache to consider the response fresh for 600 seconds, other shared caches for 120 seconds and any remaining caches for 60 seconds:

~~~ example
Cache-Control: max-age=60, s-maxage=120
CDN-Control: 600
~~~

These header fields would instruct a gateway cache to consider the response fresh for 600 seconds, while all other caches would be prevented from storing it:

~~~ example
Cache-Control: no-store
CDN-Control: max-age=600
~~~

Because CDN-Control is not present, this header field would prevent all caches from storing the response:

~~~ example
Cache-Control: no-store
~~~

Whereas these would prevent all caches except for gateway caches from storing the response:

~~~ example
Cache-Control: no-store
CDN-Control: none
~~~

(note that 'none' is not a registered cache directive; it is here to avoid sending a header field with an empty value, because such a header might not be preserved in all cases)


## Parsing

CDN-Control is specified as a Structured Field {{!I-D.ietf-httpbis-header-structure}}, and implementations are encouraged to use a parser for that format in the interests of robustness, interoperability and security.

When an implementation parses CDN-Control as a Structured Field, each directive will be assigned a value. For example, max-age has an integer value; no-store’s value is boolean true, and no-cache’s value can either be boolean true or a list of field names. Implementations SHOULD NOT accept other values (e.g. coerce a max-age with a decimal value into an integer). Likewise, implementations SHOULD ignore parameters on directives, unless otherwise specified.

However, some implementers MAY initially reuse a Cache-Control parser for simplicity. If they do so, they SHOULD observe the following points, to aid in a smooth transition to a full Structured Field parser and prevent interoperability issues:

* If a directive is repeated in the field value (e.g., "max-age=30, max-age=60"), the last value 'wins' (60, in this case)
* Members of the directives can have parameters (e.g., "max-age=30;a=b;c=d"), which should be ignored unless specified.


# Security Considerations

The security considerations of HTTP caching {{!I-D.ietf-httpbis-cache}} apply.


--- back

# Frequently Asked Questions

## Why not Surrogate-Control?

The Surrogate-Control header field is used by a variety of cache implementations, but their interpretation of it is not consistent; some only support 'no-store', others support a few directives, and still more support a larger variety of implementation-specific directives. These implementations also differ in how they relate Surrogate-Control to Cache-Control and other mechanisms.

Rather than attempting to align all of these different but well established behaviours (which would likely fail, because many existing deployments depend upon them) or defining a very small subset, a new header field seems more likely to provide clear interoperability without compromising functionality.

## Why not mix with Cache-Control?

An alternative design would be to have gateway caches combine the directives found in Cache-Control and CDN-Control, considering their union as the directives that must be followed.

While this would be slightly less verbose in some cases, it would make interoperability considerably more complex to achieve. Consider the case when there are syntax errors in the argument of a directive; e.g., 'max-age=n60'. Should that directive be ignored, or does it invalidate the entire header field value? If the directive is ignored in CDN-Control, should the cache fall back to a value in Cache-Control? And so on.

Also, this approach would make it difficult to direct the gateway cache to store something while directing other caches to avoid storing it (because no-store overrides max-age).

## Is this just for CDNs?

No; any gateway cache can use it. The name is chosen for convenience and brevity.
