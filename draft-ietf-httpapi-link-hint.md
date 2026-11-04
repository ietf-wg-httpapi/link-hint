---
title: HTTP Link Hints
abbrev:
docname: draft-ietf-httpapi-link-hint-latest
date: {DATE}
category: std

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, tocindent, sortrefs, symrefs, strict, compact, comments, inline]

venue:
  group: "Building Blocks for HTTP APIs"
  type: "Working Group"
  mail: "httpapi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/httpapi/"
  github: "ietf-wg-httpapi/link-hint"
  latest: "https://ietf-wg-httpapi.github.io/link-hint/draft-ietf-httpapi-link-hint.html"

author:
 -
    ins: M. Nottingham
    name: Mark Nottingham
    organization:
    email: mnot@mnot.net
    uri: https://www.mnot.net/

normative:
  HTTP: RFC9110
  HTTP-CACHING: RFC9111
  WEB-LINKING: RFC8288

informative:
  PATCH: RFC5789
  QUERY: RFC10008


--- abstract

This memo specifies "HTTP Link Hints", a mechanism for annotating Web links to HTTP(S) resources with information that otherwise might be discovered by interacting with them.


--- middle

# Introduction {#intro}

HTTP {{HTTP}} clients can discover a variety of information about a resource by interacting with it. For example, the methods supported by a resource can be learned by examining the Allow header field in responses from it, and the need for authentication is conveyed with a 401 (Unauthorized) status code.

Often, it can be beneficial to know this information before interacting with the resource; not only can such knowledge save time (through reduced round trips), but it can also influence the choices made by the code or user driving the interaction. For example, a user interface that presents the data from an HTTP-based API might need to know which resources the user has write access to, so that it can present the appropriate interface.

This specification defines a vocabulary of HTTP link hints that allow such metadata about HTTP resources to be attached to Web links {{WEB-LINKING}}, thereby making it available before the link is followed. It also establishes a registry for future hints.

Hints are just that -- they are not a contract, and are to only be taken as advisory. The runtime behaviour of the resource always overrides hinted information. For example, a client might receive a hint that the PUT method is allowed on all "widget" resources. This means that generally, the client can send a PUT to them, but a specific resource might reject a PUT based upon access control or other considerations.

More fine-grained information might also be gathered by interacting with the resource (e.g., via a GET), or by another resource containing it (such as a widgets collection) or describing it (e.g., one linked to it with a "describedby" link relation).

Hints are conveyed with a link by its serialisation. This specification defines how for the Link HTTP header field ({{link-header}}); other serialisations define their own ({{other-serialisations}}).

Finally, these hints are not universally applicable to all links. {{Section 2.2 of WEB-LINKING}} specifies that target attributes can be defined by link relation types and by serialisations. Therefore, to be used in a given link, these hints need to be explicitly specified by either the link relation type or the serialisation in use (e.g., by referencing this specification).

## Notational Conventions

{::boilerplate bcp14-tagged}


# HTTP Link Hints {#link_hints}

An HTTP link hint is a (name, value) tuple that describes the target resource of a Web link {{WEB-LINKING}} to an HTTP or HTTPS resource. The value's content model is either a String or a List of Strings ({{values}}). Recipients MUST ignore hints they do not recognise, and hints that do not conform to their defined content model.

Typically, HTTP link hints are serialised in links as target attributes ({{Section 3.4 of WEB-LINKING}}). {{link-header}} defines how for the Link HTTP header field; other link formats need to map the content model onto the constraints of that format ({{other-serialisations}}).

The information in a hint SHOULD NOT be considered valid for longer than the freshness lifetime ({{Section 4.2 of HTTP-CACHING}}, including both explicit and heuristic freshness) of the representation that the link occurred within, and in some cases, it might be valid for a considerably shorter period.

Likewise, the information in a hint is specific to the link it is attached to. This means that if a representation is specific to a particular user, the hints on links in that representation are also specific to that user.

## Hint Values {#values}

A String is a sequence of zero or more printable ASCII characters (%x20-7E). A List is an ordered sequence of one or more Strings; a hint that would have no members is omitted.

Strings are limited to printable ASCII. A hint that needs other characters has to define how they are encoded within a String.

Unless the definition of a hint says otherwise, a String is compared as the HTTP protocol element it represents is: method names case-sensitively ({{Section 9.1 of HTTP}}), and authentication scheme names ({{Section 11.1 of HTTP}}) and range units ({{Section 14.1 of HTTP}}) case-insensitively.


## Link Header Field Serialisation {#link-header}

In a Link header field ({{Section 3 of WEB-LINKING}}), a link hint is a target attribute: a link-param whose name is the hint name. This section defines how hints are serialised; it does not by itself make them applicable to every link (see {{intro}}).

A String is the value of one link-param. If it is valid as a `token` ({{Section 5.6.2 of HTTP}}), it can be sent as one. Otherwise it MUST be sent as a `quoted-string` ({{Section 5.6.4 of HTTP}}), with each `"` and `\` preceded by a backslash. Recipients parse both forms and treat them identically, as {{Section 3 of WEB-LINKING}} requires; the String is the value after any unquoting.

Hint names are compared case-insensitively, as {{Section 2.2 of WEB-LINKING}} requires; registered names are lowercase. A link-param without a value ({{Section 3 of WEB-LINKING}}), or whose value after unquoting contains anything other than printable ASCII, does not convey a String.

Note that `/` and `:` are not token characters in HTTP, so most media types need a quoted-string even without parameters.

A List is conveyed by repeating the link-param, once for each member, in order. Members of a List that occur in more than one Link header field line are in the order that the lines were received. A hint whose content model is a String does not conform to it if it occurs more than once in a link-value.

Note that some Link header parsers keep only one occurrence of a repeated parameter, and some mishandle quoted-strings containing `;` or `,`. A recipient built on such a parser can instead read the raw field value and parse it as {{Section 3 of WEB-LINKING}} describes.

For example:

~~~ http-message
Link: </widgets/1>; rel="item";
      allow=GET; allow=PATCH; allow=POST;
      accept-patch="application/merge-patch+json";
      accept-post="application/json";
      accept-post="text/plain; charset=utf-8";
      auth-schemes=Bearer;
      auth-realms="Bob's Files"; auth-realms=staff;
      status=deprecated
~~~

A `"` or `\` in a String is escaped once, at the Link layer:

~~~ http-message
Link: </widgets/1>; rel="item";
      accept-post="application/x; profile=\"https://example.net/p\""
~~~


## Other Serialisations {#other-serialisations}

This specification does not define mappings for other link serialisations. One that can repeat an attribute can carry a List as the Link header field does, one attribute per member. One with a structured data model might carry a String as a string and a List as an array of strings; in Linkset JSON ({{?RFC9264}}), where every extension attribute is an array of strings, each member is one array element. One that cannot repeat an attribute (XML attributes, for example) needs another mapping, such as one child element per member.


# Pre-Defined HTTP Link Hints {#hints}

## allow

* Hint Name: allow
* Description: Hints the HTTP methods that can be used to interact with the target resource; similar to the Allow HTTP header field.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is an HTTP method ({{Section 9 of HTTP}}); unrecognised values MUST be ignored by recipients.

## formats

* Hint Name: formats
* Description: Hints the representation type(s) that the target resource can produce and consume, using the GET and PUT (if allowed) methods.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is a media type ({{Section 8.3.1 of HTTP}}); unrecognised values MUST be ignored by recipients.

## accept-post

* Hint Name: accept-post
* Description: Hints the POST request format(s) that the target resource can consume.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is a media type ({{Section 8.3.1 of HTTP}}); unrecognised values MUST be ignored by recipients.

When this hint is present, "POST" SHOULD be listed in the "allow" hint when it is present.

## accept-patch

* Hint Name: accept-patch
* Description: Hints the PATCH {{PATCH}} request format(s) that the target resource can consume; similar to the Accept-Patch HTTP response header.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is a media type ({{Section 8.3.1 of HTTP}}) that identifies an acceptable patch format (see {{PATCH}}); unrecognised values MUST be ignored by recipients.

When this hint is present, "PATCH" SHOULD be listed in the "allow" hint, when it is present.

## accept-query

* Hint Name: accept-query
* Description: Hints the QUERY {{QUERY}} request format(s) that the target resource can consume;
  similar to the Accept-Query HTTP response header.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is a media type ({{Section 8.3.1 of HTTP}}) that identifies an acceptable query format (see {{QUERY}}); unrecognised values MUST be ignored by recipients.

When this hint is present, "QUERY" SHOULD be listed in the "allow" hint, when it is present.

## accept-ranges

* Hint Name: accept-ranges
* Description: Hints the range-unit(s) available for the target resource; similar to the Accept-Ranges HTTP response header {{HTTP}}.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is a range-unit ({{Section 14.1 of HTTP}}); unrecognised values MUST be ignored by recipients.

## accept-prefer

* Hint Name: accept-prefer
* Description: Hints the preference(s) {{!RFC7240}} that the target resource understands (and might act upon) in requests.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is a preference ({{Section 2 of RFC7240}}); unrecognised values MUST be ignored by recipients. Note that, by its nature, a preference can be ignored by the server.

## precondition-req

* Hint Name: precondition-req
* Description: Hints that the target resource requires state-changing requests (e.g., PUT, PATCH) to include a precondition, as per {{Section 13.1 of HTTP}}, to avoid conflicts due to concurrent updates.
* Content Model: List of Strings
* Specification: \[this document]

Possible values for each String are "etag" and "last-modified", indicating type of precondition expected; they are compared case-sensitively. Other values MUST be ignored by recipients.

See also the 428 Precondition Required status code ({{?RFC6585}}).

## auth-schemes

* Hint Name: auth-schemes
* Description: Hints that the target resource requires authentication using the HTTP Authentication framework {{Section 11 of HTTP}}.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is an HTTP authentication scheme ({{Section 11.1 of HTTP}}); unrecognised values MUST be ignored by recipients.

## auth-realms

* Hint Name: auth-realms
* Description: Hints the authentication realm(s) available for those schemes that support them in the HTTP Authentication framework {{Section 11 of HTTP}}.
* Content Model: List of Strings
* Specification: \[this document]

Each String in the List is a realm value ({{Section 11.5 of HTTP}}); together with the origin of the target resource, it identifies a protection space that the resource is a member of. The hint does not associate realms with authentication schemes; a client learns that from the challenges the target sends. A realm that cannot be represented as a String, because it contains characters outside printable ASCII, is not conveyed.

## status

* Hint Name: status
* Description: Hints the status of the target resource.
* Content Model: String
* Specification: \[this document]

Possible values are:

* deprecated - indicates that use of the resource is not recommended, but it is still available.
* gone - indicates that the resource is no longer available; i.e., it will return a 410 (Gone) HTTP status code if accessed.

Unrecognised Strings MUST be ignored by recipients.


# Security Considerations

Link hints are assertions made by the party that publishes a link. That party might not control the
target resource, and there is no mechanism for hint verification other than interacting with the
resource. Hints are therefore defined as advisory (see {{intro}}): the behaviour of the target
resource always takes precedence, and a client ought to be prepared for any hint to be wrong,
whether through staleness, error or malice.

## Authority of Hints

A hint carries the authority of the link's publisher, not of the target. This is most relevant when
a link in one origin's representation points at another origin. A client ought not to take actions
with side effects, or with security consequences, on the strength of a hint alone.

For example, "auth-schemes" and "auth-realms" could be used to coax a client into offering
credentials to a server that has not asked for them. A client ought not to send credentials to a
target on the strength of a hint alone, unless they are for a protection space ({{Section 11.5 of
HTTP}}) that the client already associates with that target. For those, "auth-schemes" can save the
round trip that a challenge would have cost; for any other, the client waits to be challenged by
the target ({{Section 11 of HTTP}}).

Likewise, a "status" of "gone" or "deprecated" might lead a client or user to avoid a resource that
is in fact available, and "allow" or "formats" might lead a user interface to offer operations that
the resource will refuse.

Hints are not access control. A server cannot rely on a client having honoured a hint, and has to
enforce its policy on every request regardless of what was published. Conversely, the absence of a
hint says nothing about what the target allows or requires.

## Hints That Vary By User

As noted in {{link_hints}}, hints in a user-specific representation are themselves user-specific.
Hints such as "allow" and "precondition-req" reveal what that user is able to do. If a
representation containing them can be stored or served to other parties -- for example, by a shared
cache, or in a catalogue of links assembled for many users -- those hints can be exposed to, or
acted on by, someone they were not generated for. Publishers ought to control caching of such
representations ({{Section 5.2 of HTTP-CACHING}}), and avoid placing user-specific hints in
representations that are shared.

Publishers can also disclose through hints what the target would reveal only to authenticated
clients, such as the methods or formats of a resource whose existence is otherwise confidential.
They ought to consider whether everyone who can see the link is entitled to know what its hints say.

## Stale Hints

A hint can be out of date as soon as it is published. Where a hint and the target's actual
behaviour disagree, a client ought to treat the hint as the one that is wrong.


# IANA Considerations

## HTTP Link Hint Registry {#hint_registry}

This specification defines the HTTP Link Hint Registry. See {{link_hints}} for a general description of the function of HTTP link hints.

HTTP link hints are generic; that is, they are potentially applicable to any HTTP resource, not specific to one application of HTTP, nor to one particular format. Generally, they ought to be information that would otherwise be discoverable by interacting with the resource.

Hint names MUST be composed of the lowercase letters (a-z), digits (0-9), underscores ("_") and hyphens ("-"), and MUST begin with a lowercase letter.

Hint content MUST be a String or a List of Strings ({{values}}); the registration states which.

Hint semantics SHOULD be described in terms of the framework defined in {{WEB-LINKING}}.

New hints are registered using the IETF Review process described in {{?RFC8126}}. Registration of new HTTP link hints is to use the following template:

* Hint Name: \[hint name]
* Description: \[a short description of the hint's semantics]
* Content Model: \[String or List of Strings]
* Specification: \[reference to specification document]

Initial registrations are enumerated in {{hints}}. Additionally, the "anchor", "rel", "rev", "hreflang", "media", "title", and "type" hint names are reserved, so as to avoid potential clashes with link serialisations.


--- back


# Acknowledgements

Thanks to Jan Algermissen, Mike Amundsen, Bill Burke, Graham Klyne, Leif Hedstrom, Jeni Tennison, Erik Wilde and Jorge Williams for their suggestions and feedback.


