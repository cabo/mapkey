---
v: 3

title: >
  Generating Numeric Map Labels from Readable CBOR EDN
abbrev: EDN for Numeric Map Labels
docname: draft-bormann-cbor-edn-mapkey-latest
# date: 2025-06-28
keyword:
  - CBOR EDN
  - References to CDDL numbers
cat: info
stream: IETF

pi: [toc, sortrefs, symrefs, compact, comments]

venue:
  mail: cbor@ietf.org
  github: cbor-wg/edn-e-ref
  latest: https://cbor-wg.github.io/edn-e-ref/draft-ietf-cbor-edn-e-ref.html

author:
  - name: Carsten Bormann
    org: Universität Bremen TZI
    street: Postfach 330440
    city: Bremen
    code: D-28359
    country: Germany
    phone: +49-421-218-63921
    email: cabo@tzi.org
  - name: Your | name here

normative:
  RFC8949: cbor
  RFC8610: cddl
  I-D.ietf-cbor-edn-literals: edn
informative:
  I-D.ietf-cbor-edn-e-ref: eref
  RFC9290: cpd
  RFC9528: edhoc
  RFC8747: pop
  RFC8392: cwt
  RFC9781: uccs
  STD96: cose
  I-D.bormann-cbor-cddl-2-draft: cddl2


--- abstract

The Concise Binary Object Representation (CBOR, RFC 8949) is a data
format whose design goals include the possibility of extremely small
code size, fairly small message size, and extensibility without the
need for version negotiation.

CBOR diagnostic notation (EDN) is widely used to represent CBOR data
items in a way that is accessible to humans, for instance for examples
in a specification.
Complex examples often use nested maps, the map keys (labels) of each
of which are often sourced from different specifications.
While the `e''` application extension provides a way to import data items,
particularly constant values, from a CDDL model, it does not help with
automatically selecting the right kind of map depending on its
position in the nested maps.

[^status]: The present document is intended to capture ideas initially
    discussed at the CBOR WG interim 2025-06-25 and demonstrate some
    design alternatives.  It is not ready for adoption in any way.

[^status]

--- middle

Introduction        {#intro}
============

(Please see abstract.)
{{-cbor}} {{-edn}} {{-eref}}

# The `mapkey''` application extension: importing map labels from CDDL

## Problem
{:unnumbered}

In diagnostic notation examples, authors often would prefer to use
text names in place of the integer map keys that are used in the
protocol message for efficiency.
While the `e''` application extension provides a way to associate
integer data items with names, the protocol designer must be very
careful to use the right name for the kind of map that uses the
integer key: Different specifications use different integer numbers
for a key of the same name, and even in a single specification there
may be homonyms that resolve to different integer values in different
kinds of maps.

For example, Figure 4 in {{Section 3.5.2 of -edhoc}} contains this
example that employs nested maps:

~~~ cbor-diag
{                                              /CCS/
  2 : "42-50-31-FF-EF-37-32-39",               /sub/
  8 : {                                        /cnf/
    1 : {                                      /COSE_Key/
      1 : 1,                                   /kty/
      2 : h'00',                               /kid/
     -1 : 4,                                   /crv/
     -2 : h'b1a3e89460e88d3a8d54211dc95f0b90   /x/
            3ff205eb71912d6db8f4af980d2db83a'
    }
  }
}
~~~

To check this example, a human reviewer has to look up the integer
labels in the specifications for the different maps employed and
translate them to the names of the map keys defined for each type of
map.
The outer map is a CWT Claims Set (CCS), for which the labels 2 (sub)
and 8 (cnf) are defined in {{-cwt}} and {{-pop}}, respectively.
Within cnf, the label for COSE_Key is also defined by {{-pop}}, while
the labels within that map are defined in {{Section 7 of RFC9052@-cose}}.
Map keys also often an extension point and therefore also may require
consulting an IANA registry.

The objective of the present proposal is that a specification writer
could employ an EDN app-extension that allows the example to read a
bit like:

~~~ cbor-diag
mapkey<<"Claims-Set",
  {
    "sub": "42-50-31-FF-EF-37-32-39"
    "cnf": {
      "COSE_Key": {
        "kty": "OKP"
        "kid": h'00'
        "crv": "X25519"
        "x": h'b1a3e89460e88d3a8d54211dc95f0b90
              3ff205eb71912d6db8f4af980d2db83a'
      }
    }
  }
>>
~~~

Note that this example not only uses names for map keys, but also uses
names for map values 1 ("OKP") and 4 ("X25519").

[^Discussion]: Discussion:

[^Discussion] For use in EDN, the names need to be provided in some CBOR form; in
the example above, this is done in text strings; for clarity, it could
be done in a different way, e.g., as byte strings or even an
`e'...'`-like construct.

## Solution
{:unnumbered}

In many cases, the constants needed to clean up this example are
already available in a CDDL model, or could be easily made available
in this way.

For the example used in this section, {{-uccs}} provides CDDL for {{-cwt}}
and {{-pop}}, and {{-cose}} provides CDDL for its data types.
Note that, to remain useful with extension points where new map keys
are defined regularly, there needs to be a way to reference
IANA registries for the name-to-integer translation; this is a
separate problem for which a potential solution is presented in
{{Appendix A.2.1 of -cddl2}}.

This section needs to define where in the CDDL the names to be
resolved are looked for; CDDL rule names are one obvious candidate, as
are member names for group choices that are often
employed for documentation (Figure 2 in {{Section 2 of -cpd}} shows
member names such as "title", "detail", etc.):

~~~ cddl
   problem-details = non-empty<{
     ? &(title: -1) => oltext
     ? &(detail: -2) => oltext
     ? &(instance: -3) => ~uri
     ? &(response-code: -4) => uint .size 1
     ? &(base-uri: -5) => ~uri
     ? &(base-lang: -6) => tag38-ltag
     ? &(base-rtl: -7) => tag38-direction
~~~

<!-- {: #fig-cddl title="CDDL model defining the constants for e''"} -->

This specification defines an app-extension `mapkey<<>>` with two parameters:

* The name of the CDDL rule for the top level (here: `Claims-Set`),
  and
* an EDN depiction of the data structure where names can be used in
  place of numeric map keys and values.

Note that the app-extension does not itself define
where the CDDL definitions it uses come from.
This information needs to come from the context of the example, and
there is probably value in establishing a convention.

IANA Considerations
==================

IANA is requested to make the following two assignments in the CBOR
Diagnostic Notation Application-extension Identifiers
registry \[IANA.cbor-diagnostic-notation]:

| Application-extension Identifier | Description                          |
|----------------------------------+--------------------------------------|
| mapkey                           | import map labels from external CDDL |
{: #tab-iana title="Additions to Application-extension Identifier Registry"}

All entries the Change Controller "IETF" and the Reference "RFC-XXXX".

[^to-be-removed]

[^to-be-removed]: RFC Editor: please replace RFC-XXXX with the RFC
    number of this RFC, \[IANA.cbor-diagnostic-notation] with a
    reference to the new registry group, and remove this note.


Security considerations
=======================

The security considerations of {{-cddl}}, {{-cbor}}, {{-edn}} apply.

TBD.

--- back

Acknowledgements
================
{: numbered="no"}

TBD.
