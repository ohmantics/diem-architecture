---
title: "Digital Emblems Architecture"
abbrev: "DIEM Architecture"
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
  CHARTER:
    target: https://datatracker.ietf.org/doc/charter-ietf-diem/01/
    title: Digital Emblems
    date: 2025-05-27
  ISPM15:
    target: https://www.ippc.int/static/media/files/publication/en/2019/02/ISPM_15_2018_En_WoodPackaging_Post-CPM13_Rev_Annex1and2_Fixed_2019-02-01.pdf
    title: "International Standards for Phytosanitary Measures No. 15: Regulation of Wood Packaging Material in International Trade"
    author:
      org: International Plant Protection Convention
    date: 2019
  DIGINOTAR:
    target: https://zoek.officielebekendmakingen.nl/blg-191225.pdf
    title: "Black Tulip: Report of the investigation into the DigiNotar Certificate Authority breach"
    author:
      org: Fox-IT
    date: 2012-08-13
  EMUDHRA:
    target: https://bugzilla.mozilla.org/show_bug.cgi?id=2083449
    title: "eMudhra: Private key compromise of Intermediate Certificates"
    author:
      org: Mozilla
    seriesinfo:
      Mozilla Bugzilla: Bug 2083449
    date: 2026-10-18
  KAZAKHSTAN:
    target: https://blog.mozilla.org/security/2019/08/21/protecting-our-users-in-kazakhstan/
    title: "Protecting our Users in Kazakhstan"
    author:
      org: Mozilla
    date: 2019-08-21
  REVOCATION:
    title: "An End-to-End Measurement of Certificate Revocation in the Web's PKI"
    author:
      - name: Y. Liu
      - name: W. Tome
      - name: L. Zhang
      - name: D. Choffnes
      - name: D. Levin
      - name: B. Maggs
      - name: A. Mislove
      - name: A. Schulman
      - name: C. Wilson
    seriesinfo:
      "Proceedings of the Internet Measurement Conference": "2015"
      DOI: 10.1145/2815675.2815685
    date: 2015-10

--- abstract

This document describes an architecture for digital emblems: authenticated, typed statements about assets, made by the holder of a domain name and published in the DNS.
It defines the roles involved, the components that connect them, and how trust is anchored, and it explains how individual use cases are accommodated by profiles.
It leaves wire formats and protocol details to other documents.

--- middle

# Introduction

Physical emblems and markings tell people how to treat what they mark: a red cross on a hospital roof, a seal on a diplomatic bag, a label on a shipment of dangerous goods.
Their digital counterparts must do the same for assets that people and software encounter over a network, or identify by a name or code: a service, a device, a container, a person on duty.
{{DIEM-REQS}} sets out the use cases and the requirements they share.

A digital emblem, as this architecture describes it, is a statement about an asset made by the holder of a domain name.
Where a use case requires it, a separate authorizing entity confirms that the issuer is entitled to make the statement.
Profiles determine the semantics and contents of these statements, and a profile will generally cover one or more use cases.
Holders publish emblems in the DNS either directly via dedicated resource records or indirectly via a record that points to an emblem service.
Thus, validators can discover emblems with an ordinary DNS query.

This document describes the roles involved, the components that connect them, how trust is anchored, and how profiles accommodate individual use cases.
It is the basis for protocol documents that will specify encodings, record types, and procedures, and for profile documents that will specify the content of emblems for particular use cases.

# Terminology

This document uses the terms asset, digital emblem (or emblem), and validation as {{DIEM-REQS}} defines them, and DNS terms as {{?RFC9499}} defines them.
It uses the term profile for what {{DIEM-REQS}} calls a digital emblem profile, with the narrower scope described in {{profiles}}.
It refines the {{DIEM-REQS}} terms emblem issuer, authorizing entity, and validator by what each produces and consumes, and adds the other terms below.

**Emblem issuer (issuer)**:
: Produces emblems about its assets, at names in a DNS zone it controls.
  Consumes authorizations, where its use case requires them.

**Emblem publisher**:
: Serves the issuer's emblems in the DNS.
  It may be the issuer itself or a provider to which the issuer delegates part of its namespace.

**Authorizing entity**:
: Produces authorizations, in a DNS zone it controls.

**Authorization**:
: A statement by an authorizing entity that an issuer may issue emblems of a stated scope (for example, a profile, the purposes it may claim, and a period).

**Validator**:
: Consumes emblems, authorizations, and associated validation data.

**Bundle**:
: An emblem together with associated validation data.

# Domain Model {#domain}

An emblem is a statement by an issuer about an asset, its subject.
The subject may be a single item or a composite (a consignment, a convoy) whose parts carry emblems of their own.
The subject is the thing the statement is about, for example, a hospital's records service or a diplomatic bag.

In some use cases anyone may issue an emblem about their own asset.
In others the issuer must be authorized: a ministry of health authorizes a hospital to use a protective emblem, or a national plant protection organization authorizes a producer to mark wood packaging {{ISPM15}}.
More than one authorizing entity may authorize the same issuer and a validator decides which it recognizes.

One asset may carry emblems under several profiles at once: a shipment may carry customs data, dangerous goods data, and recycling data, each potentially described by a different profile and read by a different validator.
An emblem may also refer to other emblems (a consignment to the products it contains).
Each referenced emblem remains the statement of its own issuer and is validated on its own; a reference may pin the version it refers to.

Some of what an emblem says may be meant for some validators only: the value of a shipment for customs, the identity of a courier for the receiving state.

## Roles and Interactions {#roles}

{{fig-roles}} shows the roles and what passes between them.

~~~ ascii-art
+--------------------+
|   Emblem issuer    |
+---------+----------+
          | issues
          | emblem
          v
+--------------------+              +--------------------+
|  Emblem publisher  |              | Authorizing entity |
+---------+----------+              +---------+----------+
          |  publishes emblem                 | authorization
          | (queried, or                      | (queried, or
          |  carried in                       |  bundled with
          |  a bundle)                        |  the emblem)
          |        +---------------+          |
          +------->|   Validator   |<---------+
                   +-------+-------+
                           | result: who asserts what
                           v
                   +---------------+
                   | Person or     |
                   | system acting |
                   | on it         |
                   +---------------+
~~~
{: #fig-roles title="Roles and what passes between them"}

The issuer wants assertions about its asset to be found and accepted by relevant validators.
The authorizing entity wants its authorization to be valid only for a particular scope, and to be able to end it.
The validator wants to learn, reliably and without revealing more than necessary about itself, what is asserted about an asset and by whom.
The emblem publisher serves emblems, potentially as a service that is designed in the DNS.
Depending on concrete operation, the publisher or issuer will sign the emblems.

# Components {#components}

Each component below is described by the properties it must provide.
Encodings, record types, and procedures are left to protocol documents.

## Emblem Information Model {#info-model}

An emblem is a dictionary of metadata about its subject: a set of entries, each pairing a name with a typed value.
Profiles define an emblem's semantics and which data may be stored in the emblem ({{profiles}}).
For example, an emblem may cite the law, agreement, or standard on which its assertions rest, and may name a visual mark that a person can be shown alongside the result.
All emblems, though, carry:

* a subject and further identifiers (a seal number, a serial number) that a validator can use for the marked asset,
* a profile,
* a validity period,
* references to related emblems, and
* values that are confidential to some validators ({{confidentiality}}).

Values are typed, and where possible drawn from types and code lists shared across profiles, so that a validator without software specific to a profile can still validate an emblem and present its content in a generic form.

The architecture will encode emblems in an existing message format that facilitates signing and encryption, which lowers the implementation burden.
Many such message formats exist, e.g., JSON Web Tokens (JWTs) {{?RFC7519}} or CBOR Web Tokens {{?RFC8392}}.

{{DIEM-REQS}} lets a profile require validation of all of an emblem's data or only a defined subset, so that frequently changing data need not be re-signed.
The information model therefore distinguishes data that must be validated from data that may be unvalidated.

## Discovery and Distribution {#distribution}

Emblems are published in DNS zones or via emblem services.
In general, emblems come in bundles that include all relevant data to validate them.
Depending on the profile, the emblem or records pointing to an emblem service may be placed at underscore labels {{?RFC8552}}, which enables delegating emblem distribution to particular name servers.

How a validator discovers an asset's name is outside the scope of the architecture, and we assume that validators determine asset names out-of-band.
For example, networked services naturally have names, and other assets, such as a bag, a pallet, or a person, may carry a physical indicator of the respective name.

## Authorization {#authorization}

An authorization states that an issuer may issue emblems within a stated scope.
The scope may be broad (any emblem under a given profile, for given purposes, during a given period) or narrow (emblems about one named asset), and may bind the issuer's keys.
A validator checks that an authorization covers the particular emblem it is validating.

Which authorizing entities a validator recognizes is a matter of policy, but profiles may name a default set.
Several authorizing entities may authorize one issuer, including entities whose claims compete; the validator is given all of them and determines which it recognizes.

## Validation {#validation}

Profiles specify how emblems are validated, as particular validation requirements as well as trust anchor choices depend on use case details.
The architecture allows for two validation mechanisms:

- *Direct validation*, via the emblem message format that supports message signing ({{authorization}}), or
- *transport validation*, via the emblem's transport protocol.
  For example, unsigned emblems may be delivered as DNSSEC-signed records {{?RFC9364}}, or emblem services may use TLS {{?RFC9846}} for transport and DANE {{?RFC7671}} for authenticating the TLS connection.

Emblem bundles facilitate offline and undetectable validation.
A bundle allows validation without further network access or offline after the bundle was cached or obtained via other, currently out-of-scope transport mechanisms.
Furthermore, bundles allow emblem validation without further follow-up queries, which facilitates undetectable validation.

## Confidentiality and Selective Disclosure {#confidentiality}

The generic emblem message format will support encryption and selective disclosure mechanisms for the emblem itself.
For example, {{?RFC9901}} specifies selective disclosure for JWTs and {{?I-D.spice-sd-cwt}} specifies selective disclosure for CWTs.
Selective disclosure can also occur via an emblem service which can authenticate validators prior to delivering emblems.
Profiles choose which values to protect and how, but the mechanisms are common to all profiles.

# Trust Model {#trust}

The architecture supports authenticating emblem issuers and authorizing entities.

For emblem profiles that use transport validation (see {{validation}}), authentication of issuers is straightforward.
For example, when relying on DNSSEC {{?RFC9364}} for validation, emblems are signed with the keys of the zone that holds them, and those keys are authenticated by a chain of signatures from the DNS root.

For emblem profiles that use direct validation (see {{validation}}), authentication of issuers requires domain-specific mechanisms.
The architecture encourages profiles to use message format authentication mechanisms wherever possible, e.g., signed JWTs support authenticating an issuer via X.509 certificates (see {{?RFC7515}}).

The authentication of authorizing entities is profile-specific as both authorizing entity selection and the nature of their authorization depend on the specificities of individual use cases.
Profiles, though, should follow the same rationale when specifying authorizing entity authentication as when specifying issuer authentication (see above).

# Profiles {#profiles}

Because an emblem is a dictionary of metadata ({{info-model}}), a profile is in essence a schema for that dictionary, for one or more use cases, together with the policy for validating emblems that follow it.
{{DIEM-REQS}} also has a profile state how emblems are discovered and which validation and authorization mechanisms apply; in this architecture, those are choices among the options described in {{components}}, not new mechanisms.

A profile has an identifier, and states:

* the entries its emblems use, their value types and code lists, and which entries are required;
* which values are confidential, to whom, and by which approach ({{confidentiality}});
* whether authorization is required, the default set of authorizing entities, and whether authorizations must be carried with the emblem;
* which validation patterns it supports ({{roles}}), with the threat models that {{DIEM-REQS}} requires for undetectable validation and removability;
* how emblem publishers respond to queries ({{DIEM-REQS}}, Query Response);
* signature lifetimes, and what absence of an emblem means; and
* what a validator should do with each validation result.

A profile may define new entries, value types, and code lists, and may publish additional records at an emblem's name.
It does not change how emblems are discovered, their anchoring in DNSSEC, or how their authenticity is established.
Where a use case needs a mechanism that the architecture lacks, the mechanism should be added to the core architecture, so that later profiles share one mechanism rather than converging on the same choices only by convention.

{{tab-uses}} shows what profiles for the use cases in {{DIEM-REQS}} would need to express or choose.

| Use case | What its profile expresses or chooses |
|---|---|
| IHL protective emblems | authorization by a competent authority; undetectable validation; removability |
| Other IHL-related marks | organizations' own marks; civilian infrastructure categories |
| Diplomatic pouches | authorization by the sending state; validation without stopping the pouch; validators' own records of validation |
| OPCW, IAEA, Basel | hazard classes and identifiers; consignor and consignee; staff protected as couriers |
| Dangerous goods | UN numbers and marks |
| Wood packaging | authorization by a national plant protection organization; absence is meaningful |
| Customs | commodity codes, origin, and value confidential to customs |
| Brands | registration of the mark; anti-counterfeiting checks by customs |
| Civil aviation | identity, flight plan, and current position; a partly redacted manifest |
| Peacekeepers | frequently changing position; point of contact; treatment of strays |
| Press | marking of journalists and their equipment |
| Wetlands | location; presence or absence of a designation |
{: #tab-uses title="Use cases and what their profiles need"}

# Future Work {#future}

Protocol documents are needed for:

* the emblem format and its record type;
* authorizations;
* bundles for offline validation; and
* the first profiles.

# Security Considerations

This section follows {{?RFC3552}} in outline; {{trust}} describes the trust model on which it depends.

**Forgery**:
: Forging an emblem requires the signing key of the issuer's zone or of one of its parents.
  Validators depend on those parents not to sign falsely for the issuer's name.

**Replay**:
: A withdrawn emblem can be replayed until its signatures expire.
  A proof of absence can likewise be replayed, making a newly published emblem appear absent until the proof's signatures expire.

**Misleading names**:
: Authentication shows who the issuer is, not that the issuer is who a person expects.
  An issuer can choose a name that resembles another's.
  Validators should present the issuer's name in a form people can check, and profiles that need more require authorization.

**Compromised or coerced authorizing entities**:
: An authorizing entity can authorize an issuer it should not.
  A validator chooses which authorizing entities it recognizes, and competing authorizations are visible to it.

**Downgrade**:
: An attacker can strip signatures or block answers.
  DNSSEC lets a validator detect both, so a validator does not treat a missing or unsigned answer as proof that no emblem exists.

**Misuse as an intrusion detector**:
: If checking for an emblem is visible to the issuer, an emblem can alert its holder to reconnaissance ({{DIEM-REQS}}).
  Undetectable validation ({{roles}}) mitigates this, within the threat model that each profile states ({{DIEM-REQS}}).

**Denial of service**:
: An attacker can make emblem publishers unreachable.
  The result is indeterminate, not invalid, and bundles remain usable.

**Revocation**:
: Withdrawal and signature expiry are how authorizations account for the compromise of key material, as {{DIEM-REQS}} requires; a profile chooses lifetimes short enough for its use case.

# Privacy Considerations

This section follows {{?RFC6973}} in outline.

DNS queries reveal a validator's interest to its resolver, and, for the names they serve, to the issuer's servers and those of authorizing entities.
A validator limits this by querying through a shared resolver, by obtaining authorizations carried with the emblem, and by validating bundles offline ({{roles}}).

Names can reveal assets.
An issuer avoids names that describe what they mark, and keeps its zone from being enumerated, so that a list of protected assets cannot be read from the DNS.

Emblems can reveal more than their holders intend.
Values meant for some validators only are protected as described in {{confidentiality}}.
Anything carried on an asset, such as a bundle in a tag, can be read by anyone with access to the asset.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

To be written.
