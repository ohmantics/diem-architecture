---
title: "Digital Emblems - Architectural Considerations"
abbrev: "DIEM Architectural Considerations"
category: info

docname: draft-rosenberg-diem-architecture-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: "Applications and Real-Time"
workgroup: "Digital Emblems"
keyword:
  - digital emblems
  - architecture
  - DNS
  - DNSSEC
venue:
  group: "Digital Emblems"
  type: "Working Group"
  mail: "diem@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/diem/"
  github: "ohmantics/diem-architecture"
  latest: "https://ohmantics.github.io/diem-architecture/draft-rosenberg-diem-architecture.html"

author:
  -
    fullname: Alex Rosenberg
    organization: Veridigo
    email: alexr@veridigo.com

normative:

informative:
  DIEM-REQS: I-D.ietf-diem-requirements
  I-D.woodcock-faltstrom-external-registry-rrtypes:
  I-D.brand-indicators-for-message-identification:
  I-D.svg-tiny-ps-abrotman:
  CHARTER:
    target: https://datatracker.ietf.org/doc/charter-ietf-diem/01/
    title: Digital Emblems
    date: 2025-05-27
  RFC1876:
  RFC3597:
  RFC4033:
  RFC4034:
  RFC4035:
  RFC4501:
  RFC4592:
  RFC5646:
  RFC6698:
  RFC6895:
  RFC6973:
  RFC7553:
  RFC7858:
  RFC7901:
  RFC8484:
  RFC8610:
  RFC8624:
  RFC8949:
  RFC9052:
  RFC9102:
  RFC9156:
  RFC9180:
  RFC9230:
  RFC9250:
  RFC9364:
  RFC9460:
  RFC9824:
  RFC9276:
  RFC7871:
  I-D.ietf-cose-hpke:
  I-D.ietf-cbor-edn-literals:
  UNECE20:
    target: https://unece.org/trade/uncefact/cl-recommendations
    title: "Recommendation No. 20: Codes for Units of Measure Used in International Trade"
    author:
      org: United Nations Economic Commission for Europe
  ISO3166:
    target: https://www.iso.org/iso-3166-country-codes.html
    title: "ISO 3166 -- Country Codes"
    author:
      org: International Organization for Standardization
  ISO4217:
    target: https://www.iso.org/iso-4217-currency-codes.html
    title: "ISO 4217 -- Currency Codes"
    author:
      org: International Organization for Standardization
  ISPM15:
    target: https://www.ippc.int/en/publications/640/
    title: "ISPM 15: Regulation of wood packaging material in international trade"
    author:
      org: International Plant Protection Convention
  HS:
    target: https://www.wcoomd.org/en/topics/nomenclature/overview.aspx
    title: "Harmonized Commodity Description and Coding System"
    author:
      org: World Customs Organization
  UNMODELREGS:
    target: https://unece.org/transport/dangerous-goods/un-model-regulations-rev-24
    title: "UN Recommendations on the Transport of Dangerous Goods, Model Regulations"
    author:
      org: United Nations
  VCDR:
    target: https://legal.un.org/ilc/texts/instruments/english/conventions/9_1_1961.pdf
    title: "Vienna Convention on Diplomatic Relations"
    date: 1961
  API:
    target: https://ihl-databases.icrc.org/en/ihl-treaties/api-1977
    title: "Protocol Additional to the Geneva Conventions of 12 August 1949, and relating to the Protection of Victims of International Armed Conflicts (Protocol I)"
    date: 1977

--- abstract

This document describes an architecture for digital emblems in which emblems are authenticated, typed metadata about assets, published in the DNS and protected by DNSSEC.
A single architecture serves many use cases.
Use cases differ in the shape of their metadata, which is described by profiles, rather than in their transport or trust machinery.
The document illustrates the architecture with a strawman encoding (an EMBLEM resource record carrying a deterministic CBOR map with registered, typed keys) and with worked examples drawn from International Humanitarian Law, logistics and customs, diplomatic pouches, and peacekeeping.

--- middle

# Introduction

The DIEM working group is developing standards for digital emblems: a means for associating metadata with physical or virtual assets, often to indicate what treatment they should receive under some normative framework {{CHARTER}} {{DIEM-REQS}}.
The use cases range from protections under International Humanitarian Law (IHL) to trade and logistics labeling, environmental designations, and diplomatic privileges.

The central claim of this document is that these use cases differ mainly in *what they say*, not in *how it is said*.
A hospital, a pallet of smartphones, and a diplomatic pouch all need an identifier that a validator can learn, a way to retrieve statements about the identified asset, assurance that the statements come from the party entitled to make them, and in some cases confidentiality, third-party authorization, or offline operation.
These are architectural concerns that can be addressed once.
What differs between use cases is the dictionary of facts being conveyed, and that is the job of a profile.

An architecture document would ordinarily stay clear of protocol detail.
In DIEM's case, several participants (including the author) have found it difficult to reason about the architecture without thinking concretely about the protocol it implies.
This document therefore includes a strawman encoding ({{strawman}}).
The strawman is illustrative: it exists to show that the architecture can be realized with existing DNS machinery and to give the working group something concrete to criticize.
Normative protocol specifications would follow in separate documents.

This revision substantially reworks draft-rosenberg-diem-architecture-00 in response to working group feedback, particularly from DNS practitioners who urged that DIEM lean on DNSSEC rather than duplicate its integrity guarantees at the application layer.

# Conventions and Definitions

This document uses the terms "(digital) emblem", "asset", "emblem issuer", "authorizing entity", "validator", and "validation" as defined in {{DIEM-REQS}}.
It also uses the following terms:

Profile:
: A digital emblem profile as defined in {{DIEM-REQS}}: a specification of an emblem's data elements and their types, which are required or confidential, how validators interpret them, how emblems are discovered, and whether validation or authorization is required.
  In this architecture, a profile defines its data and *selects* among the discovery, validation, authorization, and confidentiality options described here; it does not define its own transport or trust machinery.
  Different profiles therefore share one validator implementation.
  One profile may serve several use cases (for example, a consignment profile used by both customs and recyclers), and one asset may carry emblems from several profiles.
  {{profiles}} lists what a profile specifies.

Emblem name:
: The DNS owner name at which an asset's emblem records are published.

Emblem record:
: A DNS resource record conveying all or part of an emblem.
  In the strawman, the EMBLEM RRType.

Facet:
: A subset of an emblem's data that is retrieved, cached, or signed independently of the rest, and is therefore published as its own RRset (a separate RRType, or a separate owner name).

Key:
: A registered identifier for one property in an emblem's dictionary, with a defined value type.

Code list:
: An externally maintained controlled vocabulary (for example, ISO 3166-1 country codes {{ISO3166}} or UNECE Recommendation 20 unit codes) used as the type of a key's value.

Endorsement:
: A statement by an authorizing entity that an emblem issuer is authorized to issue emblems of a given profile or scope, published as an RRset in the authorizing entity's own zone.

Visual mark:
: An image of the physical emblem, sign, or logo associated with an emblem's profile or issuer, carried as an optional facet ({{marks}}).
  A visual mark depicts; it is not itself an emblem, and confers nothing.

Offline bundle:
: A self-contained serialization of emblem RRsets, their RRSIGs, and the DNSSEC chain needed to validate them, suitable for transfer without network access.

# Design Principles {#principles}

One architecture, many profiles:
: Use cases vary the data, not the plumbing.
  A validator that understands discovery, DNSSEC validation, and the common encoding can retrieve and authenticate any emblem, even one whose profile it cannot interpret.
  This avoids a family of disjoint per-use-case systems, which would multiply implementation cost and fragment the validator base.

Use the DNS for what it already does:
: DNSSEC {{RFC4033}} {{RFC4034}} {{RFC4035}} {{RFC9364}} provides data origin authentication and integrity for RRsets, authenticated denial of existence, and a delegation model that maps naturally onto organizational boundaries.
  DIEM should rely on these rather than re-create them with application-layer signatures.
  Application-layer cryptography is reserved for what DNSSEC does not provide: confidentiality, cross-zone endorsement, and offline transfer.

Typed data:
: Every key has a registered type, and every value can be checked against it.
  A validator must be able to tell malformed data from well-formed data, and to tell one property from another without reading free text.
  Externally maintained code lists are value types bound to keys, not free-standing records.

Query granularity equals record granularity:
: Data that a validator needs to retrieve independently, or that changes on a different schedule, belongs in its own RRset.
  Data that is always used together belongs in one record.
  The DNS offers projection (by owner name and type) but no predicates, so the record layout is the query model.

Encryption, not audience-dependent answers:
: Restricted fields are encrypted inside signed records, so that every validator receives the same cacheable, signed answer and only authorized validators can read the restricted parts.
  Serving different answers to authenticated clients is possible as a deployment option but is not the architectural primitive (see {{confidentiality}}).

Neutral about normative frameworks:
: The architecture conveys statements; it does not adjudicate them.
  Legal meaning (what an emblem entitles an asset to, who may authorize it) lives in profile data and in the frameworks that profiles reference.

# Roles and Trust {#roles}

The architecture has two active roles on the wire:

* An *emblem issuer* publishes emblem records in a DNS zone it controls (directly, or through a contracted DNS operator).
* A *validator* retrieves emblem records and validates them.

An *authorizing entity*, where a profile requires one, participates by publishing endorsements in its own zone, which validators either query or receive stapled to the emblem ({{endorsement}}).
It does not need to be online during validation.

~~~
  authorizing entity       emblem issuer          validator
  (state, treaty body,     (hospital, maker,      (armed forces,
   NPPO, ministry ...)      shipper, MFA ...)      customs, recycler)
         |                        |                      |
         | endorsement            | emblem records       |
         | (own zone; queried     | (DNSSEC-signed,      |
         |  or stapled)           |  maybe delegated)    |
         v                        v                      |
  +--------------------------------------------+         |
  |                    DNS                     |<--------+
  | root -> TLD -> issuer zone -> emblem names |  query and
  +--------------------------------------------+  validate
~~~
{: title="Roles"}

Trust in an emblem's content derives from the DNSSEC chain of trust to the zone that publishes it.
A validated EMBLEM RRset establishes that the holder of the zone's signing keys published that data, and nothing more.
Whether that zone holder is entitled to make the statement is a separate question, answered by endorsements where a profile requires them and otherwise by the validator's own knowledge (a customs officer knows which shippers they deal with).

This model has a single technical root (the DNS root zone), but it does not create a central authority over emblems.
No party needs another's permission to publish emblems in its own zone, and authorizing entities operate independently of one another.
Profiles whose frameworks require it can place their own authorizing entities wherever the framework places them.

Three consequences deserve emphasis:

* DNSSEC authenticates the zone, not the real-world identity behind it.
  A lookalike domain (for example, one using Cyrillic "a" (U+0430) in place of Latin "a" (U+0061)) validates just as well as the genuine one.
  Profiles should carry cross-check identifiers (a GTIN, a container number, a registration number) that a validator can compare against the physical or contextual evidence in front of it, and endorsements help where a third party vouches for the issuer.
* In practice, many signed zones are signed by contracted operators rather than by registrants.
  The architecture treats the zone's signing keys as the issuer's voice, so issuers should choose operators accordingly.
* The architecture depends on DNSSEC deployment.
  An issuer needs a signed chain from the root to its emblem names, so an issuer whose TLD is not signed must publish under one that is.
  Validators must validate themselves or use a validating resolver they trust over a secure channel.

# Naming and Discovery {#discovery}

## Identifying the Asset

Assets are identified by domain names.
How a validator learns an asset's name depends on the asset:

* For digital services, the asset's own name is usually already known (the validator is about to connect to it).
* For physical assets, the name is conveyed out of band: printed, in a QR code, in an NFC or RFID tag, or in shipping documents.
  A URI form for emblem names would let general-purpose scanners recognize them.
  The existing `dns:` URI scheme {{RFC4501}} already names a DNS owner name, class, and type, and may suffice; a payload that also carries an offline bundle ({{offline}}) may warrant something more.
* Some assets are found by other means (a location, a registry listing), and {{DIEM-REQS}} notes asset identifier discovery as a possible future extension.

## Choosing Emblem Names

The architecture does not constrain emblem names.
An issuer may publish emblems at an asset's own name or at any name in a zone it controls, and validators use whatever emblem name they are given; they do not construct names from a fixed pattern.
Where a profile derives an emblem name from an asset's name (for example, for network services whose names validators already know), that profile defines the derivation.

As an operational recommendation, issuers should consider placing emblem names under a dedicated label, `_diem` in this document's examples, because it permits a zone cut:

~~~
example.org.                  NS   ordinary-dns.example.net.
_diem.example.org.            NS   ns1.diem-provider.example.
                              DS   ...
~~~

With a cut at `_diem`, the issuer's ordinary DNS service needs no knowledge of DIEM.
Emblem names are served by an authoritative server specialized for emblems, which may be backed by an asset-management or inventory system rather than a zone file, and which may have different security and signing arrangements.
Resolvers need no DIEM awareness at all; they cache and validate emblem records like any other RRType.

The cut also addresses a practical obstacle.
Many hosted DNS services and provisioning interfaces accept only a fixed list of RRTypes, and some network equipment mishandles types it does not recognize, even though {{RFC3597}} requires transparent handling.
An issuer whose existing provider cannot serve new RRTypes need not change providers for the rest of its zone; it delegates only the emblem names.

Below the cut, the issuer arranges names however suits its inventory.
Hierarchical names express hierarchical assets ({{hierarchy}}), and wildcards {{RFC4592}} can cover classes of assets that share all of their data.
Wildcards have limits: they do not match names that exist (including empty non-terminals), and because every name under a wildcard appears to carry an emblem, they must not be used where the existence of an emblem for a particular item (such as a serial number) is itself meant to be evidence.

## Retrieval

A validator retrieves an emblem by querying for the relevant RRType at the emblem name, and validates the response with DNSSEC.
Where a profile places facets in separate RRsets ({{facets}}), the validator retrieves only the facets it needs.
A validator that receives NXDOMAIN or NODATA with a valid denial-of-existence proof learns authoritatively that no emblem (or no such facet) is published.

DNS transports are interchangeable here.
Large responses use TCP or encrypted transports (DoT {{RFC7858}}, DoH {{RFC8484}}, DoQ {{RFC9250}}); the encrypted transports also provide confidentiality between the validator and its resolver.

# Data Model {#data-model}

An emblem is a dictionary: a set of key/value pairs, in which values may themselves be arrays or dictionaries.
Every emblem identifies the profile that defines its keys.

## Keys and Types

Keys are drawn from registries.
A small set of common keys applies to all profiles (profile identifier, the FQDN of the marked asset, cross-check asset identifiers, validity period, issuer display name, and the treatment requested of validators, {{treatment}}).
The charter requires that emblems explicitly identify the marked asset by FQDN {{CHARTER}}, so the asset key is required even when the emblem is published at the asset's own name.
Each profile defines additional keys.
Every key definition specifies the value's type, preferably in CDDL {{RFC8610}}.
A plausible arrangement is one registry of common keys, a key space scoped to each registered profile (so that profiles allocate keys without coordinating with one another), and a private-use range for experimentation; profiles that share a concept (an ISO 3166 country of origin, say) should reuse a common key rather than define their own.

A validator checks each value against its key's type.
A value that does not match its type is an error, not a curiosity to be displayed.
Unknown keys may be ignored, unless the emblem marks them as critical (in the manner of the COSE "crit" header parameter {{RFC9052}}), in which case a validator that does not understand them must not rely on the emblem.
This lets profiles evolve without breaking old validators, while still allowing an issuer to say "do not act on this emblem unless you understand all of it".

## Value Types

The following value types cover the use cases identified so far:

| Type | Purpose |
|---|---|
| integer, boolean, byte string | as usual |
| decimal | exact decimal fixed-point (base-10 exponent and mantissa), never binary floating point |
| quantity | decimal plus a unit code from UNECE Recommendation 20 {{UNECE20}}, or an integer count plus a package type code from UNECE Recommendation 21 |
| monetary amount | decimal plus a currency code from ISO 4217 {{ISO4217}} |
| timestamp, duration | 64-bit seconds since the epoch; durations in seconds or calendar units |
| text | UTF-8 with a language tag {{RFC5646}} where it is meant for humans |
| enumeration | a value from a list defined by DIEM or by the profile (in an encoding, a small integer) |
| code | a code from an external code list fixed by the key's definition, carried verbatim |
| citation | a reference to a legal or other normative instrument, in an identified citation scheme, with optional text |
| location | a point (as in LOC {{RFC1876}}) or an area (representation to be determined) |
| URI, domain name | references to other resources and other emblems |
| media | an embedded image, such as a visual mark ({{marks}}) |
| encrypted | ciphertext of any of the above, readable by authorized validators ({{confidentiality}}) |
| dictionary, array | nesting |
{: title="Value types"}

Two distinctions from {{I-D.woodcock-faltstrom-external-registry-rrtypes}} are worth adopting: a value that is *not applicable* (the key is absent) differs from a value that *exists but is unknown to the issuer* (the key is present with an explicit "unknown" marker), and an issuer's signed assertion of ignorance is itself meaningful.

## Treatment {#treatment}

Emblems often exist to ask something of whoever encounters the asset: that it not be attacked, opened, detained, or read.
Such requests are expressed as *treatments*, each a modality (prohibit, request, or require) applied to an action from a small enumeration (attack, open, detain, inspect, read, report, and so on).
Because actions are enumerated, software can ask "does anything here prohibit inspection?" across emblems of different profiles without understanding the law behind each.

A treatment says what is asked, not why.
Its legal basis is carried separately, as one or more citations in an identified scheme (for example, a European Legislation Identifier, an Akoma Ntoso identifier, or a URI), with optional language-tagged text, and optionally a jurisdiction (an ISO 3166 code or an indication of international law).
Citations are open-ended, so profiles do not have to enumerate the laws they might invoke; validators compare citations within a scheme and display them to people, but do not need to interpret them.
A profile may still list the bases it expects, where that set is small (as for the instruments of IHL).

A treatment is a request backed, if at all, by the cited framework.
The emblem does not enforce it, and does not confer the protection or privilege it signals: under IHL, for example, the distinctive emblem indicates protection that exists by law, whether or not the emblem is displayed.
Where a profile requires authorization, the endorsement's scope covers the purposes and treatments the issuer may claim.

## Visual Marks {#marks}

People recognize emblems by sight, and many use cases have a physical sign that people already know: the red cross, red crescent, and red crystal, the Blue Shield, the ISPM 15 mark, a brand's logo.
An emblem may refer to a visual mark (an image of that sign), published as its own facet so that validators that display nothing never retrieve it.
The approach follows Brand Indicators for Message Identification (BIMI) {{I-D.brand-indicators-for-message-identification}}, which lets mail clients display a sender's logo, with three differences.

* The visual mark is published in the DNS as a facet, not fetched from a web server, so retrieving it reveals no more than retrieving the emblem, and it can be cached, stapled, and carried in an offline bundle.
* Where BIMI relies on a certificate authority to verify that a logo belongs to a domain, here that role is played by the endorsement: the authorizing entity includes a digest of the visual mark in the endorsement's scope, attesting that this issuer may display it.
* The authorizing entity is whoever the profile's framework names (a ministry for the distinctive emblems, an NPPO for the ISPM 15 mark), not only a trademark registry.

A validator displays a visual mark only if its digest is covered by a validated endorsement, or if it matches a canonical rendering that the validator already holds for an enumerated value in the emblem (for example, the red cross for a protective red cross purpose).
Otherwise the validator does not display it, or labels it clearly as unverified.
A visual mark never overrides the emblem's data: if the two disagree, the data governs and the disagreement is itself a reason for suspicion.

Marks use a restricted image format, such as the SVG Tiny Portable/Secure profile that BIMI defines {{I-D.svg-tiny-ps-abrotman}}, which excludes scripts, external references, and animation.
A simple symbol in this format is on the order of a hundred octets; detailed logos can run to several kilobytes.

## External Code Lists as Value Types {#code-lists}

Much of the data that emblems carry is already standardized elsewhere: country codes, currency codes, units of measure, location codes, commodity classifications, dangerous goods numbers.
DIEM should use these code lists rather than reinvent them, and should not mirror them into IANA registries.
On both points this document agrees with {{I-D.woodcock-faltstrom-external-registry-rrtypes}}, which argues them well.

Where this document differs is in *where the code list attaches*.
In the external-registry RRTypes, a record names a code list (for example, "ISO 3166-1") and carries a code, an optional quantity, and an optional free-text descriptor.
The property that the code describes (whether "QA" is a flag state, a place of incorporation, or a country of origin) is carried only in the descriptor, which receivers must not interpret.
Two records about the same asset that use the same code list are therefore indistinguishable to software.

In this architecture, the code list is the *type* of a key:

~~~ cbor-diag
/ external-registry pattern (illustrative): meaning only in text /
ISO 3166-1 - QA "flag state"
ISO 3166-1 - US "state of incorporation"

/ typed keys: the key carries the meaning, the code list the type /
{ "flag-state": "QA", "incorporated-in": "US" }
~~~
{: title="Where the meaning lives"}

A key definition says "the value is an ISO 3166-1 alpha-2 code", so a validator can reject "Q4" or "qa" (the codes are compared verbatim, as in the external-registry draft), and software can act on "flag-state" directly.
The alternative, recovering the property by parsing descriptor strings such as "flag state", is error-prone: descriptors are unconstrained text, so the same property can be written in many ways, in any language, abbreviated, or misspelled, and different parser implementors may interpret the strings differently.
Where a key admits values from more than one code list, the value carries a small code-list identifier alongside the code.
The registry of code-list identifiers references external lists without copying their contents.

This approach also reaches code lists that do not fit a one-RRType-per-maintainer pattern.
The OPCW use case in {{DIEM-REQS}} needs Chemical Abstracts Service registry numbers, IUPAC names, GHS hazard pictograms, and UN dangerous goods numbers {{UNMODELREGS}}; the logistics example below needs Harmonized System codes {{HS}} and ISPM 15 treatment codes {{ISPM15}}.
Each of these becomes a value type that keys can use, with no new RRType per maintainer.

## Facets: Signed Once, Signed Often {#facets}

Some emblem data never changes for the life of the emblem (a serial number, a hazard classification, a legal designation).
Some changes constantly (the location of a container in transit).
draft-rosenberg-diem-architecture-00 described this as "signed" versus "informational" fields.
In a DNS-native design, the distinction maps onto RRsets:

* Static facets are published in RRsets that can be signed offline and given long TTLs.
* Dynamic facets are published in separate RRsets with short TTLs.
  Because any key in a zone's DNSKEY RRset can sign any RRset in that zone, an online signing key in the same zone could also forge the static data.
  Dynamic data that must be signed online is therefore placed in a separate, online-signed zone (for example, one shared zone for all of an issuer's dynamic facets), so the static data remains under offline keys.

Facets are also the unit of retrieval.
A recycler that only needs the battery chemistry does not need to fetch the declared customs value, and a customs officer does not need recycling instructions.
Each facet is either a distinct RRType at the emblem name (a LOC record for position, a URI record {{RFC7553}} for a link to documentation) or a distinct owner name below it.

This replaces the application-layer hashing of the -00 draft.
DNSSEC already signs each RRset; a validator that retrieves a facet gets integrity for exactly that facet.

# Hierarchy and Composition {#hierarchy}

Physical assets nest: devices in cartons, cartons on pallets, pallets in containers, containers on ships.
Different parties attest to different levels.
The architecture expresses this in two ways:

* Within one issuer's zone, by DNS hierarchy.
  An emblem name for a product class can have child names for individual items, so that class-wide data (model, hazard classification) is published once and item-specific data (serial number) is published per item.
* Across issuers, by reference.
  A shipper's consignment emblem refers to the manufacturer's emblems by domain name.
  Each referenced emblem is validated through its own zone's DNSSEC chain.
  Where the referencing party wants to fix exactly what it saw at the time (for example, a declaration to customs), the reference also carries a digest of the referenced RRset.

A validator walks as far as its task requires.
A customs officer may validate only the shipper's consignment emblem; a recycler may validate a single device's battery facet back to the manufacturer; an audit may walk everything.
Full walks of large consignments involve many queries, which caching, wildcards for homogeneous items, and offline bundles all mitigate.

# Authorization and Endorsement {#endorsement}

Some profiles require that an issuer be authorized by a third party: a state authorizing use of a protective emblem under IHL, a national plant protection organization authorizing a pallet producer to apply the ISPM 15 mark, a treaty secretariat recognizing a designated site.
DNSSEC proves who published an emblem, not who authorized it.
Endorsement fills that gap.

An endorsement is an RRset in the authorizing entity's own DNSSEC-signed zone.
It names the endorsed issuer and the scope of the endorsement: the profile, the permitted purposes or designations, and a validity period.
For example, `hospital.example._endorse.health.gov.example` might carry an ENDORSE record listing the protective purposes the ministry has authorized for `hospital.example`.
The endorsement can also carry a digest of the issuer's DNSKEY RRset or key-signing key, in the manner of a DS record, so that it binds the zone as it was when endorsed: if the issuer's domain lapses and is re-registered, or its keys are replaced without the authorizing entity's knowledge, the endorsement no longer matches.
(Binding to keys means the authorizing entity must re-endorse when the issuer rolls its key-signing key, so a profile chooses whether to bind to the zone name, the keys, or both.)

Validators obtain an endorsement in one of two ways:

Queried:
: The validator looks up the endorsement in the authorizing entity's zone and validates it with DNSSEC.
  Revocation is removal, bounded by TTL and signature lifetime.
  The cost is an extra query, which reveals to the authorizing entity's servers (through resolvers) that someone is interested in the issuer.

Stapled:
: The issuer retrieves its own endorsement, with its RRSIGs and the DNSSEC chain from the root to the authorizing entity's zone, and publishes that serialization inside or alongside its emblem, much as a TLS server staples a certificate status response.
  A validator validates the stapled RRset against the chain it carries, with no query to the authorizing entity.
  A stapled endorsement cannot be recalled once copied, so its lifetime is that of its signatures.
  The authorizing entity controls that lifetime through the signature validity period it chooses for endorsement RRsets, and the issuer must refresh the staple before it expires.
  This suits profiles that require undetectable validation or offline operation.

Either way the endorsement is one object, signed one way, in one place.
No second signature format is needed, and the authorizing entity's operational duty is the familiar one of maintaining a signed zone.
Authorization that passes through several levels (a state authorizing a ministry, which endorses hospitals) is a chain of endorsements, each in its endorser's zone, and a staple can carry the whole chain.

Where the list of endorsed issuers is itself sensitive, the authorizing entity can name endorsements by a digest of the issuer's name and use a denial-of-existence method that resists enumeration ({{confidentiality}}): a validator that knows the issuer can find the endorsement, but nobody can list all of them.

An authorizing entity that also wants its endorsements to be publicly auditable (so that issuing a secret endorsement is detectable) can log them in a transparency log as an addition; a profile can require this without the architecture depending on it.

A profile states whether it requires endorsement, which authorizing entities it recognizes and how validators learn of them, and whether queried endorsements, stapled endorsements, or both are acceptable.
Because each authorizing entity's trust is anchored in a zone it controls, no global registry of authorities is needed.

# Confidentiality and Selective Disclosure {#confidentiality}

Some emblem data is restricted: a consignment's declared value, the identity of a diplomatic courier, an issuer's internal forensic data.
{{DIEM-REQS}} anticipates selective disclosure.

The architectural primitive is encryption of values inside signed records.
An encrypted value is a COSE {{RFC9052}} encryption structure (COSE_Encrypt0 for a single recipient, COSE_Encrypt with recipient structures where several parties, such as origin and destination customs, need access), with keys distributed as the profile specifies (for example, using HPKE {{RFC9180}} {{I-D.ietf-cose-hpke}} to a public key that a customs authority publishes in its own zone).
DNSSEC signs the ciphertext, so integrity covers restricted fields even for validators that cannot read them.
This resolves an open question from the -00 draft (how encrypted subrecords participate in integrity protection): the signature covers what is published, which is the ciphertext.

The alternative of authenticated DNS views, in which the server authenticates the client and returns different data to different audiences, is not recommended as the general mechanism:

* It identifies validators to issuers' servers, which is unacceptable where inspection patterns are sensitive (and fatal to undetectable validation).
* Answers that differ per client cannot be shared through resolver caches, so the restricted tier becomes an application API with DNS framing.
* Standardizing mandatory client identification for DNS lookups creates a precedent that could be misused well beyond DIEM.

Profiles may still use authenticated access where audit of who looked is itself a requirement, but should do so knowingly.

Confidentiality also covers existence.
Where the mere existence of an emblem name is sensitive, issuers use unguessable labels and a denial-of-existence method that does not reveal them.
Plain NSEC allows zone walking and would expose such labels.
Compact denial of existence {{RFC9824}} reveals nothing but requires online signing; offline-signed zones can use NSEC3 {{RFC9276}}, whose hashed names resist enumeration when the labels themselves have high entropy.
NSEC3 still reveals, through its type bitmaps, how many names in the zone hold emblem records, though not which names they are; issuers for whom that count is sensitive should use compact denial or add decoy names.

# Offline and Constrained Validation {#offline}

{{DIEM-REQS}} requires that the architecture not assume Internet access.
DNSSEC data is self-authenticating: an RRset, its RRSIG, and the chain of DNSKEY and DS records up to a trust anchor can be validated anywhere, by anyone holding the root trust anchor and a clock.

An offline bundle serializes exactly that, following the pattern of the TLS DNSSEC chain extension {{RFC9102}} and the CHAIN query {{RFC7901}}.
A full chain from the root is typically a few kilobytes, so a bundle fits in a large QR code, a high-capacity tag, or an accompanying document, and can be validated at a disconnected port or in the field.
Signature algorithm choice matters here: newer algorithms, such as those based on elliptic curves, have much smaller keys and signatures than RSA (see {{RFC8624}} for current guidance), so issuers that expect their emblems to travel in bundles should prefer them.
Bundle sizes will need revisiting as post-quantum algorithms, whose signatures are generally larger, come into use for DNSSEC.
Its validity is bounded by the shortest RRSIG validity period it contains, which also bounds how long a removed emblem can be replayed.
That includes the signatures of the root and TLD zones, which the issuer does not control and which typically last one to three weeks, so bundles have a limited shelf life and must be refreshed for long journeys.
An offline validator also needs a trustworthy clock, since a validator whose clock can be set back will accept expired signatures.

Offline bundles take over the role that application-layer signed objects (such as signed CBOR) played in earlier thinking, without introducing a second signature system.

# Lifecycle, Removal, and Forensics {#lifecycle}

Removal:
: Some profiles require that an emblem be removable ({{DIEM-REQS}}).
  Removing the records stops new validators from finding them, but signed copies remain valid until their RRSIGs expire, and caches may hold them until their TTLs expire.
  Profiles that need prompt removal should use short signature validity periods and TTLs, which in practice means online signing, and which limits the shelf life of offline bundles accordingly.
  An emblem's own validity period is a different thing: it bounds the issuer's statement, while signature lifetimes bound how long a copy can be replayed, and only the latter governs removal.
  Profiles should also be candid in their threat models that the DNS is not forgetful: passive DNS collectors may retain records indefinitely, which matters for the "learn after the fact" aspect of removability.

Archival:
: Removal from the DNS is not deletion.
  Most use cases expect the issuer to archive emblems, with lifespan rules set by the profile.
  Some archived data is public forensic record; some is internal (for example, the identity of the staff member who created a record) and may be published only in encrypted form or not at all.

Proof of presence:
: A validated DNSSEC response is a portable, third-party-verifiable record that the zone's key signed a given emblem with a validity period covering a given time.
  A validator that retains the response (with the DNSKEYs then in effect) holds evidence supporting Level 1 proof of presence in the terms of {{DIEM-REQS}}.
  Level 2 (that a validator accessed the emblem) would need records of access, for example server-side logs where a profile accepts that access is observable, and Level 3 must come from the validator's own signed records; both are outside the DNS data itself.

# Undetectable Validation {#undetectable}

The IHL use case requires that validation be undetectable by the asset's operator ({{DIEM-REQS}}).
Several mechanisms contribute, and none is sufficient alone:

* Resolver mediation: an issuer's authoritative server sees queries from resolvers, not from validators.
  Large shared resolvers and Oblivious DoH {{RFC9230}} enlarge or obscure the population of query sources.
  EDNS Client Subnet {{RFC7871}} works against this; validators should use resolvers that do not send it, or opt out with a source prefix length of zero.
* Limiting other observers: encrypted transports protect the path to the resolver, and QNAME minimisation {{RFC9156}} keeps parent zones from learning the full name.
  Neither hides the query from the issuer's own authoritative server.
* Caching: a cached answer generates no query at all.
* Self-contained answers: if everything needed for validation (including stapled endorsements, {{endorsement}}) arrives in one response, no follow-up query to the issuer or authorizing entity reveals interest.
* Query indistinguishability: if the query that retrieves an emblem is one that ordinary clients make anyway, retrieving the emblem reveals nothing.
  For a network service, the emblem can travel in a record that clients already request when connecting, such as the asset's HTTPS record {{RFC9460}}.
  Techniques of this kind place emblem data in the asset's own zone, so they cannot be combined with delegating emblems to a separate server through a zone cut, and they should not require changes to recursive resolvers, which would delay deployment.

This document recommends a baseline that works with today's DNS: validators query through large shared resolvers that do not send client subnet information, over encrypted transports, and issuers publish self-contained emblems with stapled endorsements and TTLs long enough for caching to be effective.
Against an issuer that watches its own authoritative server, this baseline hides *who* is validating but not *that* someone is, and only when the resolver's population is large enough to blend in.
Query indistinguishability is the only measure that hides the act of validation itself, so a profile whose threat model includes that adversary should require such a technique and accept its deployment constraints.

A profile that requires undetectable validation must state its threat model ({{DIEM-REQS}}) in terms of which of these it relies on.

# Defining a Profile {#profiles}

Because the architecture leaves the use-case-specific choices to profiles, a profile specification has a predictable shape.
A profile specifies:

Data:
: Its keys, each with a type (preferably in CDDL), and which are required, which are critical, and which are confidential; and which common keys and code lists it reuses.

Layout:
: How its data divides into facets, which facets are static and which dynamic, and the expected TTLs and signing arrangements for each ({{facets}}).

Naming and discovery:
: How validators learn emblem names for its assets, and whether names are derived from the asset's name, conveyed out of band, or deliberately unguessable ({{discovery}}).

Hierarchy:
: Whether its emblems refer to emblems of other issuers or profiles, and whether references carry digests ({{hierarchy}}).

Authorization:
: Whether endorsement is required, which authorizing entities it recognizes and how validators learn of them, and whether queried or stapled endorsements are acceptable ({{endorsement}}).

Confidentiality:
: Which values are encrypted, to whom, and how recipients' keys are found ({{confidentiality}}).

Lifecycle:
: Removal expectations, the signature lifetimes and TTLs that follow from them, archival rules, and the level of proof of presence it supports ({{lifecycle}}).

Threat model:
: Its adversaries, including whether validation must be undetectable and by whom ({{undetectable}}), and whether unvalidated emblems may be used at all.

Interpretation:
: The treatments it may request ({{treatment}}), the citation schemes it expects for their legal basis, what a validated emblem means for a validator acting under the profile's normative framework, and what a validator should do when an expected emblem is absent or fails validation.

A profile specification therefore needs no protocol design of its own.
The worked examples in {{examples}} are informal sketches of four such profiles.

# Validation Outline {#validation}

Validation proceeds in the same way for every profile; only the final interpretation is profile-specific.
In outline, a validator:

1. obtains the emblem name (or an offline bundle) for the asset;
2. retrieves the emblem records it needs, or extracts them from the bundle;
3. validates them with DNSSEC, treating bogus data as absent and, unless the profile permits unvalidated emblems, treating unsigned data as absent;
4. checks that the emblem identifies the asset in front of it, by FQDN and by whatever cross-check identifiers the profile defines;
5. identifies the profile, checks each value against its key's type, and rejects the emblem if it contains a critical key the validator does not understand;
6. checks validity periods;
7. where the profile requires authorization, validates the endorsement (queried or stapled) and checks that its scope covers the emblem;
8. follows references to other emblems as far as its task requires, checking digests where present;
9. decrypts any confidential values it is entitled to read; and
10. interprets the result under the profile, optionally retaining the validated responses as evidence ({{lifecycle}}).

Steps 1 through 9 need no profile-specific code beyond the profile's key and type definitions, which a validator can load as data.

# Meeting the Requirements {#reqs-map}

The following table summarizes how the architecture addresses the requirements of {{DIEM-REQS}}.

| Requirement | How it is addressed |
|---|---|
| Digital emblem format | A typed dictionary that identifies the marked asset by FQDN and names its profile, with a small set of common keys ({{data-model}}, {{strawman}}) |
| Emblem semantics | Defined by each profile ({{profiles}}); requested treatments are expressed uniformly, with an open legal basis ({{treatment}}) |
| Discovery | A DNS query at the emblem name; authenticated denial of existence shows that no emblem exists ({{discovery}}) |
| Query response | Authoritative servers answer every query (Assured Response) unless a profile permits Selective Response, at the cost of cacheability. Encrypted fields give every validator the same answer (Consistent Content) while restricting what each can read; audience-dependent answers (Selective Content) are possible but not recommended ({{confidentiality}}) |
| Removable | Record removal, bounded by signature lifetimes and caching ({{lifecycle}}) |
| Undetectable validation | Resolver mediation, caching, self-contained answers, and query indistinguishability ({{undetectable}}) |
| Validation | DNSSEC, in a common procedure ({{validation}}); facets let a profile validate all data or a subset, with dynamic data signed separately ({{facets}}). Profiles may permit unvalidated emblems |
| Authorization | Endorsements published in the authorizing entity's zone, queried or stapled; key compromise is handled by DNSSEC key rollover, optional key binding in endorsements, and short signature lifetimes on stapled endorsements ({{endorsement}}). No Internet access is assumed ({{offline}}) |
| Extensibility | New profiles and keys without protocol changes; critical keys protect validators from misreading ({{data-model}}) |
| Data formats | Value types, with external code lists as types ({{code-lists}}) |
| Asset identifier discovery, implicit discovery | Out-of-band conveyance of emblem names ({{discovery}}) |
| Confidentiality | Encrypted values in signed records; enumeration-resistant names ({{confidentiality}}) |
| Proof of presence | Retained DNSSEC responses support Level 1 ({{lifecycle}}) |
{: title="Requirements coverage"}

# Strawman Encoding {#strawman}

This section is illustrative.
It sketches one way to realize the architecture so that the working group can evaluate it; it is not a specification.

## The EMBLEM RRType

The EMBLEM RRType carries one emblem dictionary (or one facet of one) as RDATA.

* The RDATA is a single CBOR {{RFC8949}} map using core deterministic encoding (Section 4.2.1 of {{RFC8949}}).
  Deterministic encoding fixes the serialization but not application-level equivalence, so the encoding must also fix canonical forms (for example, decimal mantissas without trailing zeros, integer epoch seconds) for equal emblems to have equal wire representations.
* Map keys are small integers from a "DIEM Emblem Keys" registry (common keys) or from the key space of the profile named in the map.
  Text names are used in this document for readability, for enumerated values as well as for keys.
* Key 1, "profile", is required.
* Enumerated values (profile identifiers, treatment actions, purposes, and the like) are small integers from registries; codes from external code lists are carried verbatim as text.
* Decimal values use CBOR tag 4 (decimal fraction); timestamps use tag 1; language-tagged text uses tag 38; URIs use tag 32; encrypted values are COSE_Encrypt0 (tag 16) or COSE_Encrypt (tag 96); a value that exists but is unknown to the issuer is the CBOR simple value undefined.
* RDATA contains no compressible domain names, so the type is compatible with unknown-type handling {{RFC3597}}.
  Domain-name references are carried as text, normalized for comparison and digests (A-labels, lowercase, without a trailing dot).
* The presentation format would be the generic `\# length hex` form of {{RFC3597}}, with CBOR diagnostic notation for human display.

Why a CBOR map rather than SVCB-style key/value parameters {{RFC9460}}?
Both preserve the dictionary model.
CBOR brings typed values, nesting, decimal fractions, language tags, and COSE encryption without new definitions.
The cost is that zone-file tooling would need a CBOR-aware presentation format to be friendly.

Why not one opaque blob per asset?
Because DNSSEC signs per RRset, packing everything into one record forfeits independent retrieval, independent TTLs, and the static/dynamic split of {{facets}}.
Profiles should split emblems into facets along the lines that validators actually query.

Size needs attention at two levels.
Individual records vary: item-level emblems like those below are well under 100 octets and class-level ones a few hundred, while a consignment emblem with transport references and several encrypted fields approaches the 1,232-octet limit commonly used for DNS over UDP, and an emblem with a stapled endorsement chain exceeds it and arrives over TCP or an encrypted transport.
Aggregates matter more: a consignment of 1,400 devices implies 1,400 item emblems, and validators should not have to fetch them all.
Class-level data is published once, consignments refer to items by digest rather than listing them, and validators sample individual items or use an offline bundle prepared for the consignment.

## Companion RRTypes

Existing RRTypes serve some facets directly: LOC {{RFC1876}} for point positions, URI {{RFC7553}} for references to documentation, TLSA {{RFC6698}} where an emblem is bound to a TLS service.
A separate RRType for typed quantities could serve profiles that query quantities independently.

New types would be needed for endorsements (ENDORSE, whose RDATA could reuse the EMBLEM encoding with a key space of its own: the endorsed issuer in place of the asset, the endorsed profile, scope, validity, and optional key and visual mark digests) and possibly for areas (polygons), which LOC does not express.
A stapled endorsement would be carried in an emblem as a byte string holding the serialized RRsets and chain, in the format of {{RFC9102}}.
All of these, like EMBLEM, need only Expert Review for allocation {{RFC6895}}.

# Example Use Cases {#examples}

The following examples are sketches.
Names, keys, and values are illustrative.
Emblem dictionaries are shown in CBOR diagnostic notation with text keys and text enumerated values (both of which would be integers in an encoding, {{strawman}}), using the elision `...` of {{I-D.ietf-cbor-edn-literals}} for long values.
Country codes beginning with "X" are user-assigned placeholders.

## IHL: A Hospital and Its Digital Services {#ex-ihl}

Example General Hospital operates a patient records system at `records.hospital.example`.
Under IHL, a medical unit entitled to protection may display the red cross, red crescent, or red crystal with the authorization of the competent authority, and Additional Protocol I already provides distinctive signals, including radio signals and electronic identification, chiefly for medical transports (Article 18 and Annex I, Chapter III of {{API}}).
A digital emblem extends this to the hospital's network services.

Issuer and authorization:
: The hospital is the issuer.
  The ministry of health of its state, as the competent authority, endorses the hospital for the protective use of the emblem.
  The ministry publishes the endorsement in its own zone, under a label derived from a digest of the hospital's name so that its list of protected facilities cannot be enumerated.
  Because validators must not reveal their interest, the hospital staples the endorsement, with its DNSSEC chain, inside its emblem, so validation requires no query to the ministry.

Placement:
: The asset is a network service, so its name is already known to anyone who would interact with it.
  The hospital may publish at `_diem.records.hospital.example` (delegated to a provider that serves emblems), or use one of the query-indistinguishability techniques of {{undetectable}} at the service's own name, as its profile's threat model dictates.

~~~ cbor-diag
/ EMBLEM at _diem.records.hospital.example /
{
  "profile": "ihl-protective",
  "purpose": ["protective-red-cross"],
  "asset":   "records.hospital.example",
  "treatment": [{"prohibit": "attack"}],
  "basis":   [{"scheme": "uri",
               "id":   32("https://ihl-databases.icrc.org/..."),
               "text": 38(["en", "GC I, Art. 19"])}],
  "valid":   [1(1793491200), 1(1801440000)],
  "issuer-name": 38(["en", "Example General Hospital"]),
  "mark":    "_mark._diem.records.hospital.example",
  "endorsement": {                / stapled, from the ministry /
    "by":     "health.gov.example",
    "staple": h'...'   / ENDORSE RRset, RRSIGs, chain from root /
  }
}

/ ENDORSE at 3q9v...._endorse.health.gov.example (stapled above) /
{
  "profile": "endorsement",
  "issuer":  "hospital.example",
  "endorsed-profile": "ihl-protective",
  "scope":   ["protective-red-cross", {"prohibit": "attack"}],
  "valid":   [1(1793491200), 1(1801440000)],
  "dnskey-digest": h'...',     / binds the hospital's current KSK /
  "mark-digest":   h'...'      / the mark the hospital may display /
}

/ EMBLEM at _mark._diem.records.hospital.example /
/ (visual mark facet: an image of the distinctive emblem) /
{
  "profile":    "mark",
  "asset":      "records.hospital.example",
  "media-type": "image/svg+xml",
  "image":      h'...'   / <SVG of Red Cross emblem> /
}
~~~

Validator:
: A cyber unit planning operations queries through a large shared resolver, receives the emblem with its stapled endorsement in one response, validates the emblem with DNSSEC and the endorsement against the stapled chain, and finds that the asset is marked with protective use authorized by the named ministry.
  (With the chain stapled, the emblem is several kilobytes, so it arrives over TCP or an encrypted transport.)
  Its display shows the red cross, both because the purpose calls for it and because the visual mark's digest matches the ministry's endorsement; a generic validator with no knowledge of IHL could display the endorsed image of the distinctive emblem on the second ground alone.
  What the unit does with that is a matter for IHL, not for the protocol.

Removal:
: If the hospital judges that the emblem increases its risk, it removes the records.
  Its profile specifies short signature lifetimes and TTLs (hours, not weeks), which bounds replay of the removed emblem and implies that the hospital's emblem zone is signed online.
  The stapled endorsement does not extend that window, because the emblem that carries it is itself short-lived; the ministry's own signature lifetimes bound how long the endorsement could be reused after the ministry withdraws it.

## Logistics and Customs: A Pallet of Smartphones {#ex-logistics}

A manufacturer ships 1,400 boxed smartphones on a wooden pallet.
A freight forwarder consolidates the pallet into a consignment.
Customs at the destination, and eventually a recycler, need different subsets of the data.

The manufacturer's zone holds class-level and item-level data:

~~~
x1.models._diem.maker.example.              EMBLEM  (class: model X1)
sn-35791a7c.x1.models._diem.maker.example.  EMBLEM  (one device)
~~~

~~~ cbor-diag
/ EMBLEM at x1.models._diem.maker.example (static, offline-signed) /
{
  "profile": "product-class",
  "asset":   "x1.models._diem.maker.example",
  "gtin":    "00614141999996",
  "model":   "X1",
  "dangerous-goods": {
    "un-number": "3481",  / Li-ion batteries in equipment /
    "class":     "9"
  },
  "battery-chemistry": "li-ion",
  "battery-energy": [4([-1, 185]), "WHR"],   / 18.5 watt-hours /
  "recycling": 32("https://maker.example/recycle/x1"),
  "origin":    "VN",
  "mid-us":    "VNEXAMAK123HAN",     / US manufacturer ID /
  "radio-auth": {"fcc-id": "2AEXAX1"},
  "components": [
    {"part": "battery-cell",
     "attestation": "cell.x1.supply._diem.maker.example",
     "digest": h'5e1b...'},
    {"part": "display",
     "attestation": "disp.x1.supply._diem.maker.example",
     "digest": h'c07a...'}
  ]
}

/ EMBLEM at sn-35791a7c.x1.models._diem.maker.example /
{
  "profile": "product-item",
  "asset":   "sn-35791a7c.x1.models._diem.maker.example",
  "serial":  "35791A7C",
  "made":    1(1790812800)
}
~~~

The hazardous materials, origin, and regulatory data is published once for the model, not 1,400 times.
A validator holding a device's emblem name finds class data by removing the leftmost label.
The manufacturer mirrors the attestations it holds for key components in its own zone, each with a digest of the original, so that a validator can trace components without contacting the component suppliers.

The pallet carries the ISPM 15 mark {{ISPM15}}, authorized by the national plant protection organization (NPPO) of the country where the mark was applied.
The pallet producer publishes the mark's data, and the NPPO endorses the producer (a queried endorsement suffices here, since nobody needs to hide that they checked a pallet):

~~~ cbor-diag
/ EMBLEM at p-20261031-0042._diem.pallets.example /
{
  "profile":   "ispm15",
  "asset":     "p-20261031-0042._diem.pallets.example",
  "country":   "VN",          / ISO 3166-1 alpha-2 /
  "producer":  "000123",      / registration number from the NPPO /
  "measure":   "HT"           / ISPM 15 treatment: heat /
}
/ ENDORSE at pallets.example._endorse.nppo.gov.example /
~~~

The forwarder wraps everything in a consignment emblem at an unguessable name in its own zone (an NSEC3-signed zone, so the name cannot be enumerated), with live position in a shared online-signed zone:

~~~ cbor-diag
/ EMBLEM at c-7f3k9q2m._diem.forwarder.example /
{
  "profile":    "consignment",
  "asset":      "c-7f3k9q2m._diem.forwarder.example",
  "hs-code":    "851713",          / smartphones /
  "hts-us":     "8517130000",
  "origin":     "VN",
  "container":  "EXMU1234569",     / ISO 6346 /
  "seal":       "EX4471029",
  "packages":   [1, "PX"],         / UNECE Rec. 21: pallet /
  "gross-mass": [4([0, 725]), "KGM"],
  "transport": {
    "bill-of-lading":   "EXSC260098812",
    "carrier":          "EXSC",
    "vessel":           "EXAMPLE STAR",
    "voyage":           "042E",
    "port-of-lading":   "VNSGN",   / UN/LOCODE /
    "port-of-unlading": "USLAX"
  },
  "valuation":  16(...),           / encrypted to customs /
  "parties":    16(...),           / encrypted: seller, buyer, /
                                   / importer of record, bond, /
                                   / ship-to, consolidator,    /
                                   / stuffing location         /
  "contents": [
    {"emblem": "p-20261031-0042._diem.pallets.example"},
    {"emblem": "x1.models._diem.maker.example", "count": 1400,
     "items-digest": h'9a0c...'}   / digest of the item name list /
  ]
}

/ LOC at c-7f3k9q2m._live._diem.forwarder.example /
/ (shared online-signed zone, TTL 300)              /
~~~

The valuation is a dictionary of its own, encrypted so that only customs can read it.
Its plaintext uses monetary amounts (a decimal and an ISO 4217 code) and the commercial terms customs needs to determine the customs value:

~~~ cbor-diag
/ plaintext of "valuation" /
{
  "method":     "transaction-value",   / customs valuation method /
  "related-party": false,
  "incoterm":   "FOB",                   / ICC Incoterms rule /
  "invoice":    "INV-2026-10-4471",
  "unit-price": [4([-1, 4125]), "USD"],  / 412.50 /
  "quantity":   1400,
  "total":      [4([2, 5775]), "USD"],   / 577,500 /
  "freight":    [4([2, 32]), "USD"],     / 3,200 /
  "insurance":  [4([0, 1155]), "USD"],   / 1,155 /
  "additions":  []                       / no assists or royalties /
}
~~~

Customs:
: The officer scans the consignment's code, validates the forwarder's emblem, and decrypts the valuation and parties with the customs authority's key.
  The container and seal numbers, gross mass, and transport references tie the emblem to the carrier's manifest and to what is physically in front of the officer.
  National requirements (here a US tariff number, manufacturer identification code, and radio equipment authorization) are keys like any other.
  Because amounts are typed decimals with currency codes, software can check that unit price times quantity matches the total and the declared item count, compare the unit price with what is plausible for HS 8517.13 and model X1, and compute the customs value on the authority's own basis (adding freight and insurance where it values on a CIF basis).
  The officer also checks the dangerous goods classification, optionally validates the pallet's ISPM 15 data and NPPO endorsement, and can follow the class emblem's component attestations where supply-chain tracing is required.
  A retained copy of the validated response records exactly what the forwarder declared, which matters if the valuation is later disputed.

Authenticity:
: Checking whether goods are genuine today means querying each brand through its own system.
  Here, the officer checks that a device's item emblem exists in the brand's own zone and validates with DNSSEC, which confirms that the manufacturer issued that serial number for that model; a serial that does not resolve, or resolves to a different model, is a red flag.
  The officer must still confirm that the zone really is the brand's (lookalike domains are the counterfeiter's obvious move), for example against a list of brand domains the customs authority maintains.
  An emblem alone does not prove that a physical item is the one the serial was issued for, since a counterfeiter can copy a genuine serial.
  Profiles can address cloning by binding a device-held key into the item emblem so that the device can prove possession when scanned, or by letting the manufacturer flag serials reported in implausible places.
  This replaces today's bespoke per-brand lookup arrangements with one validator that works for every brand and every shipment.

Recycler:
: Years later, a recycler scans one device, validates its item emblem, walks up to the class emblem, and reads the battery chemistry and energy.
  It never sees the commercial data, and never needs to.

## Diplomatic Pouches {#ex-pouch}

Under Article 27 of the Vienna Convention on Diplomatic Relations {{VCDR}}, the diplomatic bag may not be opened or detained, and its packages must bear visible external marks of their character.
Those marks are easy to forge and impossible to verify at a distance, and the requirements document describes both the misuse of pouches for smuggling and the pressure to inspect them.

Issuer:
: The sending state's ministry of foreign affairs publishes an emblem for each pouch under an unguessable label in its own zone, using a denial-of-existence method that prevents enumeration ({{confidentiality}}).

~~~ cbor-diag
/ EMBLEM at k7q2-9xwm-4hd3._pouch._diem.mfa.gov.example /
{
  "profile":       "diplomatic-pouch",
  "asset":         "k7q2-9xwm-4hd3._pouch._diem.mfa.gov.example",
  "sending-state": "XA",                     / ISO 3166-1 /
  "consignee":     "embassy.xa.example",     / receiving mission /
  "seal":          "XA-2026-114357",   / matches physical seal /
  "pieces":        3,
  "gross-mass":    [4([-1, 184]), "KGM"],    / 18.4 kg /
  "courier":       16(...),       / to receiving state /
  "treatment":     [{"prohibit": "open"}, {"prohibit": "detain"}],
  "basis":         [{"scheme": "uri",
                     "id":   32("https://legal.un.org/..."),
                     "text": 38(["en", "VCDR, Art. 27(3)"])}],
  "valid":         [1(1793750400), 1(1794355200)]
}
~~~

Validator:
: The receiving state's customs scans the pouch mark, validates the emblem against the sending ministry's zone, compares the seal number, piece count, and mass with what is in front of them, and decrypts the courier details.
  The pouch is never opened.

Proof of presence:
: The validated response is evidence that the emblem was published and valid when checked.
  A customs service that wants Level 3 proof of presence ({{DIEM-REQS}}) records its own signed validation event; that record is outside the DNS.

Removal:
: After delivery, the ministry removes the emblem and retains it in its archive.

## Peacekeepers: People, Vehicles, and Places {#ex-peacekeepers}

A peacekeeping mission marks its facilities, vehicles, and personnel for the duration of a deployment ({{DIEM-REQS}}).
The mission is the issuer, and the organization that deploys it endorses the mission's zone.
This example differs from the others because some of its assets are people, and an emblem on a person is also a potential tracking identifier.

Facilities:
: A camp is marked by area for the period of its use, with a point of contact.
  Its location is public by design, so it is a static facet that can be signed offline.

Vehicles:
: A convoy's position changes constantly and is sensitive.
  It is a dynamic facet in the mission's online-signed zone, encrypted to the mission's operations center and to the host state's liaison, rather than a plaintext LOC record that anyone could follow.

Personnel:
: Each person's emblem is published under an unguessable label that is replaced at every rotation, so that a name scanned once does not identify the person in future deployments.
  The emblem carries the mission, the person's protected status, and a validity period.
  Their identity is not in it, even in encrypted form, because it is not needed to confirm that the bearer is a member of the mission.

~~~ cbor-diag
/ EMBLEM at r4nq-8w2k-c7tz._p._diem.mission.example /
{
  "profile":  "peacekeeping-personnel",
  "asset":    "r4nq-8w2k-c7tz._p._diem.mission.example",
  "mission":  "mission.example",
  "status":   "peacekeeper",
  "valid":    [1(1793491200), 1(1801267200)],
  "treatment": [{"prohibit": "attack"}, {"prohibit": "detain"}]
}

/ EMBLEM at _diem.mission.example (the mission itself) /
{
  "profile":  "peacekeeping-mission",
  "asset":    "mission.example",
  "contact":  32("https://mission.example/report"),
  "if-invalid": [{"request": "report"}, {"request": "hold"}]
}
~~~

Validator:
: At a checkpoint, a soldier scans a badge, validates the emblem and the mission's endorsement, and learns that the bearer is a member of the mission.
  If a badge does not validate, nothing on it can be trusted, including any instructions it carries.
  The mission therefore publishes its point of contact and what it asks in that case ("if-invalid") in its own emblem, which the validator validates separately and can cache or carry in an offline bundle.

Privacy:
: The mission's authoritative servers see, through resolvers, when each personnel emblem is queried, and could build a map of where its people are being checked.
  The profile may treat that as useful (stray reporting) or as a risk to manage, but it should say which ({{privacy}}).
  If an individual's emblem raises their risk, the mission removes it, and short signature lifetimes bound how long it can be replayed.

## Other Use Cases

The same pattern applies to the other use cases in {{DIEM-REQS}}.
A Ramsar-designated wetland would carry a location area and a designation endorsed by the Ramsar Secretariat; a CITES shipment would carry permit numbers and species codes; an OPCW Schedule 1 shipment would carry CAS numbers and GHS classifications with confidential quantities.
A law firm could mark a case file as privileged, requesting that it not be read without review and citing the applicable rule of its jurisdiction, with no authorizing entity at all.
Each is a profile, not a new architecture.

# Security Considerations

DNSSEC provides integrity and origin authentication for emblem data only if validators actually validate.
Validators must perform DNSSEC validation themselves, or use a resolver they trust over a secure channel, and must treat bogus emblem data as absent.
Unsigned emblem data must also be treated as absent unless the profile permits unvalidated emblems, in which case it carries no assurance of origin or integrity.

An emblem is a statement by a zone holder.
Validators must not infer authorization from publication; where a profile requires authorization, the validator must validate the endorsement as well.
Lookalike and typosquatted domains are not addressed by DNSSEC; cross-check identifiers and endorsements mitigate them.

Removal is bounded by signature lifetimes and caching, and passive DNS collection means that removed emblems may remain known.
Profiles with removability requirements must account for both.

Encrypted values protect confidentiality, but metadata remains visible: the emblem name, the profile, the size of the ciphertext, and the timing of queries.
Unguessable labels, with compact denial or NSEC3 rather than plain NSEC, reduce enumeration risk.

Full audits of hierarchical emblems cause many queries; offline bundles and caching mitigate this, and profiles should state what depth of validation they expect.

Validity checks depend on time.
A validator whose clock is wrong, or can be set back by an attacker, will accept expired signatures and replayed emblems, which matters most for offline validators.

Validators that display images supplied by issuers face two risks: the image may mislead a person (a lookalike mark, or a mark that contradicts the emblem's data), and the image format may attack the software that renders it.
Validators should display only visual marks that meet the conditions of {{marks}}, render them with a restricted renderer, and never let a visual mark override the emblem's data.

As with physical emblems, displaying a digital emblem may increase an asset's risk in some contexts.
Profiles should consider this when specifying discovery and removal.

# Privacy Considerations {#privacy}

Emblems are published so that they can be read, and the privacy questions concern who learns what beyond the emblem's intended content {{RFC6973}}.

Validators:
: Each query reveals interest in an asset to the validator's resolver and, through the resolver, to the issuer's authoritative servers and to any authorizing entity whose endorsement is queried.
  For some profiles this is harmless; for others the pattern of inspections is itself sensitive.
  {{undetectable}} describes the available mitigations and their limits.

Issuers:
: An issuer's emblem names can reveal its inventory, its customers, or its operations.
  Enumeration of a zone (by NSEC walking) or of an authorizing entity's endorsements exposes them in bulk, which unguessable labels and enumeration-resistant denial address ({{confidentiality}}).

Assets that are people:
: {{DIEM-REQS}} includes people among the assets an emblem can mark.
  An emblem name that stays fixed is a persistent identifier: anyone who scans it once can recognize the person later, link sightings, and watch whether the emblem is still published.
  Profiles that mark people should use unguessable names that change on a schedule suited to the use case, keep personal data out of emblems or encrypt it, avoid publishing location in plaintext, and avoid cross-references that link one person's emblems together.
  The issuer can observe where and when a person's emblem is validated, and a profile should say whether that is intended.

Encrypted values:
: Encryption hides content but not size, timing, or the set of recipients a profile defines.
  A value encrypted to a recipient today may be read by anyone who obtains that recipient's key later, and emblems are retained by caches, passive DNS collectors, and validators long after removal ({{lifecycle}}).
  Profiles should not encrypt data whose later disclosure would be harmful unless that risk is acceptable.

# IANA Considerations

This document has no IANA actions.
A specification realizing it would request the EMBLEM and ENDORSE RRTypes and create registries for common keys, profile identifiers, and code-list identifiers.

--- back

# Changes from draft-rosenberg-diem-architecture-00
{:numbered="false" removeinrfc="true"}

* Adopted DNSSEC as the integrity layer, replacing application-layer hashing of emblem records.
* Replaced "signed" and "informational" fields with static and dynamic facets in separate RRsets.
* Replaced the Merkle-tree hierarchy with DNS naming plus cross-zone references with optional digests.
* Added typed keys and external code lists as value types.
* Added treatment (requested actions with an open legal basis) as a common key.
* Added visual marks, displayed only when endorsed or canonical.
* Added endorsement (as RRsets in the authorizing entity's zone, queried or stapled), confidentiality, offline bundles, lifecycle, and undetectable validation sections.
* Added what a profile specifies, and an outline of validation common to all profiles.
* Added Privacy Considerations.
* Added a strawman EMBLEM encoding using deterministic CBOR.
* Added worked examples for IHL, logistics and customs, diplomatic pouches, and peacekeeping.

# Acknowledgments
{:numbered="false"}

The author thanks the participants of the DIEM working group for their drafts and comments.
