---
title: "Privacy Framework for Remote ATtestation procedureS"
abbrev: "RATS Privacy Framework"
category: info

docname: draft-ounsworth-rats-privacy-framework-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: SEC
workgroup: RATS
keyword:
 - remote attestation
 - privacy
# venue:
#   group: RATS
#   type: Working Group
#   mail: WG@example.com
#   arch: https://example.com/WG
#   github: USER/REPO
#   latest: https://example.com/LATEST

author:
 -
    fullname: Mike Ounsworth
    organization: Cryptic Forest Software
    email: mike@ounsworth.ca
 -
    fullname: Hannes Tschofenig
    organization: University of the Bundeswehr Munich
    abbrev: UniBw M.
    email: hannes.tschofenig@gmx.net
 -
    fullname: Guiliano Lehmann
    organization: University of Applied Sciences Bonn-Rhein-Sieg
    abbrev: H-BRS
    email: guiliano.lehmann@smail.inf.h-brs.de
normative:
  COSE-HPKE: I-D.draft-ietf-cose-hpke
  JOSE-HPKE: I-D.draft-ietf-jose-hpke-encrypt
  SD-CWT: I-D.draft-ietf-spice-sd-cwt
  SD-JWT: RFC9901
  JWP: I-D.draft-ietf-jose-json-web-proof
  JPT: I-D.draft-ietf-jose-json-proof-token
  JPA: I-D.draft-ietf-jose-json-proof-algorithms

informative:
  CMW: I-D.draft-ietf-rats-msg-wrap
  IAB-Statement-2023:
    target: https://datatracker.ietf.org/doc/statement-iab-statement-on-the-risks-of-attestation-of-software-and-hardware-on-the-open-internet/
    title: "IAB Statement on the Risks of Attestation of Software and Hardware on the Open Internet"
    author:
      org:
      - "Internet Architecture Board"


...

--- abstract

This document extends the RATS Architecture to consider "coercive uses of RATS" where a malicious Verifier or Relying Party uses RATS protocols to extract sensitive information from an Attester or a victim Verifier that it would not otherwise be inclined to disclose.  This over-disclosure can include revealing sensitive measurements, stable identifiers, device fingerprints, vendor information, or conclusions derived from Evidence.  This document defines a privacy framework for Remote Attestation that identifies this threat surfaces; classifies claims produced by Attesters and Presenters; restricts sensitive Evidence disclosure to authorized Trusted Verifiers using confidentiality protection; and describes privacy-preserving Attestation Results based on data minimization, Selective Disclosure, and Zero-Knowledge Proofs.

--- middle

# Introduction

## Problem Statement

In its conception, remote attestation focuses on measured boot where Evidence typically consists of measurements of hardware, firmware, and software components.
This data is often treated as operational security data rather than privacy-sensitive data.
The RATS Architecture {{!RFC9334}} assumes that the purpose of remote attestation is for the Attester to prove its own trustworthiness to a Verifier or Relying Party, but it considers only in passing that the Verifier or Relying Party could be malicious and coercively using Remote Attestation to extract sensitive data from an Attester.
This document challenges that assumption and fills in the gaps by providing a framework that can be applied consistently across RATS under which sensitive claims and sensitive conclusions derived from those claims can be safely handled by all RATS entities.
While the RATS Architecture {{!RFC9334}} mentions in passing the need for a Verifier to be authenticated, and the Entity Attestation Token (EAT) {{?RFC9711}} includes a mechanism for encrypting Evidence, this document provides a framework for doing this in a consistent manner.

This document uses the privacy terminology from {{?RFC6973}} rather than relying on jurisdiction-specific legal terms.
RFC 6973 defines an Identity as any subset of an individual's attributes, including names, that identifies the individual within a given context.
It defines an Identifier as a data object uniquely referring to an entity in some context.
It also defines a Fingerprint as a set of information elements that identifies a device or application instance.

Remote attestation usually discloses software and hardware attributes of an Attesting Environment rather than attributes of a human individual.
Examples include firmware versions, software manifests, configuration measurements, attestation public keys, device serial numbers, and hardware capabilities.
When an Attester is associated with an individual, household, or organization, those attributes can enable identification, correlation, secondary use, or disclosure concerning that associated party.

This document also defines Vendor Info as a sensitive category of remote attestation Evidence.
The 2023 IAB Statement on the Risks of Attestation of Software and Hardware on the Open Internet {{IAB-Statement-2023}} does not use this term, but discusses the related risk that attestation mechanisms can be used to restrict access to otherwise open services based on client software or hardware properties.

Overly-verbose Attestation Results can further propagate the privacy risks even when the original Evidence is never disclosed to the Relying Party.
The privacy problem is therefore multi-faceted: sensitive Evidence needs to be protected from passive eavesdroppers in transit to the Verifier, the Verifier needs to be authenticated and authorized to view sensitive Evidence, and Attestation Results disclosed to Relying Parties needs to minimize the inclusion of sensitive claims.

In addition to claims being privacy sensitive, they can also be sensitive in a security sense; being able to view the complete configuration state of a device can greatly aid an attacker in compromising that device for example the manifest of firmware and software versions tells an attacker which known vulnerabilities to try, and the configuration state of the device (such as an increase in the boot counter) could give the attacker confirmation that the attack was successful.

## Threat Surfaces

This document distinguishes three privacy threat surfaces.

* **Disclosure to the Verifier.**  The Verifier appraises Evidence and therefore often needs detailed claims, Endorsements, Reference Values, and appraisal policy.
  A framework can reduce unnecessary disclosure to Verifiers by using claim classification, trust anchor stores for Trusted Verifiers, purpose-specific policies, and encrypted Evidence, but it cannot eliminate the need for the Verifier to receive enough information to perform appraisal.
  In this sense, disclosure to the Verifier is a privacy risk that can be constrained, audited, and made explicit, but not generally removed.

* **Disclosure to the Relying Party.**  The Relying Party applies an Appraisal Policy for Attestation Results to decide whether to rely on the Attester.
  In many deployments, this decision can be made from a minimized Attestation Result, from selectively disclosed claims, or from a proof of a policy predicate.
  This is the primary privacy focus of this framework.
  Relying Parties generally do not need raw Evidence or detailed Attestation Results unless those details are necessary for their own policy decision and are authorized by the Attester or device owner.

* **Disclosure to eavesdroppers.**  Evidence, Attestation Results, and related artifacts can be observed on communication interfaces between Attester, Presenter, Relying Party, and Verifier.
  Communication security protection, as provided by TLS, is necessary for these interfaces.
  Object security, such as encrypting Evidence for the Verifier, provides additional protection against incidental logging, misrouting, intermediaries, and Relying Parties that carry Evidence but are not authorized to inspect it.

This threat separation is important because different controls apply to each case.
Encrypting Evidence for a Verifier protects against eavesdroppers and against Relying Parties that forward Evidence in the Background-Check Model, but it does not minimize what the Verifier learns.
TLS protects communication channels, but it does not prevent an authorized recipient from learning or retaining excessive claims.
Selective Disclosure and Zero-Knowledge Proofs can minimize what a Relying Party learns from an Attestation Result, but they do not replace the Verifier's appraisal of Evidence.

## The Framework

This document provides a remote attestation privacy framework in four parts.

First is a conceptual framework for classifying Evidence claims and Attestation Result claims as identity-related information, Attester Identifiers, Fingerprints, Vendor Info, or unclassified.

Second is a threat framework that separates disclosure to Verifiers, disclosure to Relying Parties, and disclosure to eavesdroppers.

Third is a technical framework whereby an attesting client (RATS Attester or Presenter) maintains one or more trust anchor stores for Trusted Verifiers and encrypts sensitive Evidence for authorized Verifiers.

Fourth is a technical framework for minimizing Attestation Results before they are disclosed to Relying Parties.
This includes conventional data minimization, Selective Disclosure, and Zero-Knowledge Proofs.

## RFC 6973 Identity Roles in RATS

RFC 6973 defines a Relying Party as an entity that relies on assertions of identities from Identity Providers in order to provide services.
The RATS Relying Party role is similar in that it relies on assertions produced by another party, namely Attestation Results produced by a RATS Verifier.

The RATS Verifier can act as an Identity Provider vouching for the identity of the Attester in the limited sense that it can include identifier claims in Attestation Results, but RATS Verifier goes beyond the scope of [RFC6973] establishing, maintaining, securing, and vouching for claims about the entire posture of Attester beyond merely its identity.
Therefore,, this analogy is not intended to be read too broadly.
A RATS Verifier typically vouches for attributes of a device, workload, execution environment, or composite Attester, not for the identity of a human individual.
The Attestation Result might contain a device Identifier, a pseudonymous Attester Identifier, a set of attributes, a policy decision, or a proof about such attributes.
Only in deployments where the Attester is bound to an individual, or where Evidence carries user authentication material, does the Verifier's output become an assertion about a human identity.

This distinction is important for privacy analysis.
RATS can create identity-management-like flows even when the subject of the assertion is not a person.
The privacy risks arise because device and workload attributes can still identify, fingerprint, or correlate the Attester, and because the Attester can be associated with an individual, organization, or role.

## RATS Topologies and Privacy

The RATS architecture defines the Passport Model and the Background-Check Model {{?RFC9334}}.
These topologies have different privacy properties and therefore require separate analysis.

In the **Passport Model**, the Attester obtains an Attestation Result from a Verifier and presents it to a Relying Party.
The Relying Party does not need to carry raw Evidence to the Verifier.
This topology is generally preferable when the privacy goal is to keep raw Evidence away from Relying Parties.
If the Attester-to-Verifier and Attester-to-Relying-Party channels are protected with TLS or equivalent channel security, encrypting Evidence as an object provides less incremental protection against network eavesdroppers than it does in the Background-Check Model.
However, object encryption can still be useful for protecting Evidence from intermediaries, logging systems, queueing infrastructure, or Presenters that are not authorized to inspect the Evidence, which is especially relevant in cloud environments where TLS is terminated at the network edge.
In the Passport Model, the main privacy challenge usually moves from Evidence confidentiality to Attestation Result minimization.

In the **Background-Check Model**, the Relying Party obtains Evidence from the Attester and sends it to the Verifier.
This topology creates a stronger privacy risk because the Relying Party is on the Evidence path.
If the Evidence is only protected by TLS on each hop, the Relying Party can observe, log, retain, or correlate the Evidence.
Encrypting the Evidence for the Verifier changes this property: the Relying Party can carry the Evidence to the Verifier without learning its sensitive contents, or being legally responsible for the correct handling of this content.
Therefore, encrypted Evidence provides a direct privacy benefit in the Background-Check Model.
Nevertheless, the Background-Check Model still requires careful minimization of the Attestation Result returned to the Relying Party.
If the result reveals the same sensitive details that were hidden in the encrypted Evidence, the privacy gain is lost.

Combinations of the two models inherit the risks of each path.
Implementations need to analyze which roles can observe Evidence, which roles can observe Attestation Results, and which parties can correlate presentations over time.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Classification of Claims

For the purposes of this document, an attestation client is either an Attester directly, or a Presenter acting as a proxy between the Attester and the Verifier.
A conforming client is responsible for classifying every Evidence claim that it is capable of producing according to the following categories.
Verifiers producing Attestation Results SHOULD also be aware of the classification of claims in a given Evidence format, and have appropriate policy for transforming sensitive claims into Attestation Results.
For example, if a firmware version claim would be Vendor Info in Evidence, then an Attestation Result claim that reveals the same firmware version is also Vendor Info.

## Identity-Related Information

Identity-related information is any Evidence or Attestation Result claim that contains or can reveal an Identity, Identifier, or Attribute of an individual as those terms are used in {{?RFC6973}}.
Examples include account names, user names, email addresses, biometric authentication attributes, or identity claims carried by embedded authentication formats.

Many RATS deployments will not disclose human identities directly.
However, a device, workload, or execution environment can be closely associated with an individual or a small group.
In such deployments, Attester Identifiers and Fingerprints can become identity-related information because they allow a Relying Party or other observer to link protocol interactions to the associated individual or group.

## Attester Identifiers

An Attester Identifier is any data object that uniquely refers to a specific Attester, device, workload, or execution environment in some context.
This category applies the RFC 6973 concept of an Identifier to RATS protocol entities and subjects.
Attester Identifiers include the obvious things such as serial numbers of either the device or of certificates issued to the device, cryptographic public keys, persistent pseudonyms, static addresses, and stable key identifiers.

Sometimes Attester Identifiers are hiding in places that even the implementers might not be aware of, such as DNs, certificate serial numbers, public keys of certificates issued to the hardware at manufacture-time, or stable values in Endorsements. Implementers need to be especially aware of multiple claims being used in aggregate to form a stable fingerprint that can be used as a stand-in for an identifier, as described in the next section.
Attester Identifiers are privacy-sensitive because they enable correlation across protocol interactions, Relying Parties, or administrative domains.

## Fingerprints

A Fingerprint is a set of information elements that identifies a device or application instance, as described in {{?RFC6973}}.
In RATS, this includes hardware manifests such as the model numbers of all connected peripheral devices, manifests of installed software, detailed configuration measurements, or combinations of appraisal outcomes that are sufficiently distinctive.
Implementations SHOULD also consider hashes of firmware, software, or filesystem contents, especially where this is user-customizable, because these values can quickly become unique Fingerprints.

Fingerprints differ from explicit Attester Identifiers because no single claim is necessarily unique.
The combination of claims can nevertheless identify the Attester with high probability.
This makes Fingerprints especially easy to overlook when designing Evidence and Attestation Result formats.

## Vendor Info

Vendor Info is any Evidence claim that can be used, in isolation or in aggregate, to identify the model and manufacturer of a device.
Obvious examples include version information about hardware, firmware, and software, or manufacturer-issued certificates.
Implementers SHOULD also consider fingerprints that are unique to a given device model or manufacturer, which could include hashes of firmware or software.

The main privacy concern is that Vendor Info can enable policy decisions based on implementation provenance rather than protocol compliance, including proprietary vendor lock-in, forced obsolescence, or other exclusion of otherwise compliant and interoperable implementations.
Such policies can be appropriate within an explicit administrative trust boundary, such as an enterprise network, datacenter, or managed service environment.
In open Internet deployments, devices SHOULD NOT disclose Vendor Info unless the device owner has explicitly consented to onboard the device into a service or network that requires this visibility.

## Unclassified

This document defines an unclassified claim as any claim that does not meet the definition of any of the categories above, and thus MAY be freely released from the device without any access control or confidentiality protection.

This document does not preclude the development of further categories of sensitive claims.

# Technical Framework

The technical framework presented here requires conformant implementations to apply privacy controls at two different information disclosure points.

First, Attesters and Presenters control release of Evidence to Verifiers.
They maintain lists of Trusted Verifiers (either directly, or via chaining to a store of trusted roots) and the categories of sensitive claims that those Verifiers are authorized to request.
Verifiers requesting sensitive claims are required to provide an encryption credential compatible with the object security mechanism used by the deployment, such as COSE or JOSE HPKE, so that Evidence can be encrypted for them.

Second, Verifiers control release of Attestation Results to Relying Parties.
They minimize the content of Attestation Results and MAY use Attestation Result formats based on Selective Disclosure or Zero-Knowledge Proof technologies when the Relying Party's policy can be satisfied without revealing detailed claims.

## Trusted Verifiers

Conformant Clients, which could be either an Attester directly, or a Presenter acting as a proxy for the Attester, MUST have a mechanism to maintain trust anchor stores representing Trusted Verifiers. The trust anchors could be self-signed certificates, public keys such as those described in {{?RFC7250}}, {{?RFC5280}}, and {{?RFC5914}}, or JWK keys as per {{?RFC7517}}. The details of maintaining a trust anchor store and performing path validation against the trust anchor store are out of scope of this document.

Conformant Clients SHOULD have a mechanism to restrict released information based on the authorization level of the Verifier. A simple all-or-nothing approach would be refuse to return any Evidence except to authorized Verifiers who can access the entire content. Or, an Attester might have a "public" version of the Evidence with only minimal content. Or, an Attester might have a complex mechanism to authorize a given Verifier for a given category of claims or even for specific claims.


## Evidence Encryption

Conformant Clients MUST NOT release Evidence containing sensitive claims in plaintext.
Instead, Verifiers MUST provide an encryption credential chaining to a trust anchor in the relevant trust anchor store that can be used to encrypt the attestation payload.

Where the Verifier's encryption credential is an X.509 certificate, it MUST contain an encryption-type keyUsage of keyEncipherment, dataEncipherment, or keyAgreement and MUST contain the Extended Key Usage `id-kp-tbd-evidence-encryption`.

If the attestation payload contains at least one claim of a sensitive category, then the sensitive payload MUST be confidentiality protected for the Verifier.
Evidence can be represented in many different formats, including proprietary formats.
For standardized Entity Attestation Token (EAT) Evidence {{!RFC9711}}, the following encoding-specific guidance applies.
For CBOR/COSE-encoded Evidence, deployments SHOULD use COSE encryption structures with COSE HPKE {{COSE-HPKE}}.
For JSON/JOSE-encoded Evidence, deployments SHOULD use JOSE encryption structures with JOSE HPKE {{JOSE-HPKE}}.
When the Conceptual Message Wrapper (CMW) {{CMW}} is used, the choice of encryption envelope SHOULD match the underlying Evidence encoding in order to reduce parser burden.

This specification does not preclude the development of additional or proprietary Evidence formats which are free to specify their own object encryption mechanism, provided that it is compliant with the privacy objectives laid out in this document.

Object encryption does not replace channel security.
Attester-to-Verifier, Attester-to-Relying-Party, Relying-Party-to-Verifier, and Verifier-to-Relying-Party interfaces SHOULD use TLS or equivalent channel protection.
Object encryption provides protection that channel security alone does not provide when Evidence traverses an entity that is allowed to forward but not inspect it, or when messages are logged, queued, retried, or processed by intermediaries.

Evidence encryption MUST be bound to the intended Verifier and to the protocol context.
Deployments SHOULD use protected headers, additional authenticated data, or equivalent mechanisms to bind the ciphertext to the recipient, Evidence type, nonce or freshness value, and any Relying Party challenge that the Verifier is expected to appraise.
Unprotected metadata SHOULD be minimized and MUST NOT contain sensitive claims.
For CBOR Web Tokens (CWTs), RFC 9597 {{?RFC9597}} defines a mechanism for carrying CWT Claims in COSE headers.
This mechanism can be useful in the Background-Check Model when a Relying Party needs to relay encrypted Evidence to the correct Verifier and the type information available in the CMW is not sufficient for routing.
Any CWT Claims placed in COSE headers for this purpose MUST be limited to non-sensitive routing information.
If such claims are carried in unprotected COSE headers, recipients MUST treat them as untrusted until the Verifier has completed cryptographic validation.

## Attestation Result Minimization

Verifiers MUST treat Attestation Results as potentially sensitive.
An Attestation Result can reveal sensitive claims directly, or it can reveal sensitive information indirectly by exposing detailed appraisal outcomes.

A Verifier producing an Attestation Result for a Relying Party SHOULD disclose the least information needed for the Relying Party's Appraisal Policy for Attestation Results.
Where possible, Verifiers SHOULD disclose a coarse-grained status, policy identifier, assurance level, or authorization decision instead of raw Evidence-derived values.
For example, a Relying Party that only needs to know whether the Attester satisfies an enterprise baseline SHOULD receive a result such as "baseline satisfied" and relevant freshness information, rather than a complete software bill of materials or a list of component versions.

Attestation Results intended for different Relying Parties SHOULD be audience-restricted and purpose-specific.
A result created for one Relying Party MUST NOT be usable as a bearer artifact at a different Relying Party unless the privacy and security policy explicitly permits that use.
Attestation Results SHOULD avoid stable identifiers unless the Relying Party is authorized to identify or correlate the Attester.
When an identifier is needed, deployments SHOULD prefer pairwise, audience-specific, or short-lived identifiers over globally stable identifiers.

The Verifier SHOULD maintain policy describing which Relying Parties are authorized to receive which categories of Attestation Result claims.
The policy SHOULD be at least as strict as the policy used by Attesters and Presenters when releasing sensitive Evidence to Verifiers.

## Selective Disclosure in Attestation Results

Selective Disclosure is useful when an Attestation Result contains multiple claims, but a Relying Party is authorized to learn only a subset of them and the Verifier does not want to, or due to the network protocol is not capable of, producing per-Relying Party Attestation Results.
In this model, the Verifier acts as the issuer of a privacy-preserving Attestation Result, the Attester or Presenter acts as the holder that chooses which disclosable claims to present, and the Relying Party acts as the verifier of that selectively disclosed result.
This role mapping is separate from the RATS role named Verifier.

For CBOR and COSE deployments, Attestation Results that are represented as CWTs SHOULD consider Selective Disclosure CWTs {{SD-CWT}}.
SD-CWT defines a data minimization technique for CWTs aligned with COSE and CWT, including a holder presentation flow and COSE header parameters for disclosed claims.
An SD-CWT-based Attestation Result can allow a holder to disclose only claims needed by a given Relying Party, such as a policy identifier, freshness value, or coarse-grained assurance level, while withholding other claims.

For JSON and JOSE deployments, Attestation Results that are represented as JWTs or verifiable credential-like tokens SHOULD consider SD-JWT {{SD-JWT}}.
An SD-JWT-based Attestation Result can support existing JOSE-based Relying Party ecosystems while enabling disclosure of only selected claims.

Selective Disclosure is not sufficient by itself to prevent all correlation.
Undisclosed claims can still be inferred from disclosed claim combinations, credential type identifiers, issuer identifiers, timestamps, key-binding material, or unique metadata.
Deployments using Selective Disclosure for Attestation Results SHOULD minimize metadata, use audience restriction, support nonce-based freshness, and consider decoy digests or batch issuance where supported by the selected format.

## Zero-Knowledge Proofs in Attestation Results

Zero-Knowledge Proofs are useful when a Relying Party does not need a claim value, but only needs assurance that a predicate over one or more claims is true.
Examples include proving that a firmware version is at least a required version without revealing the exact version, proving that a measured component is a member of an approved set without revealing which member, or proving that an Attester has a certified hardware capability without revealing the manufacturer or model.

JOSE work on JSON Web Proof (JWP) {{JWP}}, JSON Proof Token (JPT) and CBOR Proof Token (CPT) {{JPT}}, and JSON Proof Algorithms (JPA) {{JPA}} is applicable to privacy-preserving Attestation Results.
JWP defines a proof-oriented container with a presentation form for selective disclosure and proof computation.
JPT and CPT define compact claim token formats with selective disclosure, and support unlinkability and reusability when used with ZKPs.
JSON Proof Algorithms define algorithm identifiers for use with JWP, JWK, and COSE.

An Attestation Result profile using ZKPs SHOULD specify:

* the Attestation Result claims or predicates that can be proven without disclosure;
* the public inputs available to the Relying Party;
* the trust anchors and issuer keys used to validate the proof;
* the freshness, audience, and replay-protection mechanisms;
* the mapping between RATS Appraisal Policies for Attestation Results and proof predicates; and
* any residual metadata that remains visible outside the proof.

ZKPs provide the strongest privacy benefit when the Relying Party can make its decision from predicates rather than values.
They also introduce implementation complexity and can raise performance questions.
Deployments SHOULD use ZKPs for high-sensitivity claims and MAY use Selective Disclosure as a simpler mechanism where predicate proofs are not required.


## Relationship to Privacy Pass

Privacy Pass {{?RFC9576}} is an architecture for privacy-preserving authorization in which a Client obtains tokens through an issuance protocol and later redeems those tokens to an Origin.

The Origin can authorize the Client based on a valid token without receiving unique per-Client information through the redemption protocol and without linking redemption to the issuance interaction.

The Privacy Pass HTTP authentication scheme {{?RFC9577}} defines the challenge and redemption mechanism, and the Privacy Pass issuance protocols {{?RFC9578}} define token issuance variants based on VOPRF {{?RFC9497}} and Blind RSA {{?RFC9474}} mechanisms.

Privacy Pass is relevant to this framework because it demonstrates a deployment pattern in which detailed attestation information is used before issuance, while the party making the authorization decision receives only a privacy-preserving token.
In RATS terms, this is similar to using a minimized Attestation Result or a proof of a policy predicate instead of disclosing Evidence or detailed appraisal results to the Relying Party.

RFC 9576 maps Privacy Pass Clients to the RATS Attester role and Privacy Pass Attesters to the RATS Verifier role; the Privacy Pass Origin is analogous to a Relying Party that consumes the resulting token for authorization.

This document does not require Privacy Pass, and a Privacy Pass token is not a general-purpose Attestation Result unless a profile defines the corresponding semantics, issuer trust model, freshness, audience restriction, and claim or predicate meaning.
Existing Privacy Pass tokens are primarily authorization tokens proving that issuance occurred under an accepted issuance policy, whereas RATS Attestation Results can carry richer appraised state, policy identifiers, assurance levels, or proofs over specific claims.
Nevertheless, profiles of this framework can use Privacy Pass as a design reference, or potentially as a concrete mechanism, when the Relying Party only needs to know that an Attester satisfied an issuer policy and does not need the underlying Evidence or claim values.

The privacy properties depend on the deployment model.
As in Privacy Pass, separation between the party that observes detailed attestation information and the party that redeems the token can improve unlinkability, while collusion, small anonymity sets, issuer selection, token metadata, timing, network identifiers, or overly specific challenges can weaken it.
Profiles using Privacy Pass-style tokens therefore still need the minimization, audience restriction, freshness, replay protection, and metadata guidance described in this document.


# Security Considerations

This document relies on the security of the underlying RATS architecture {{?RFC9334}}, Evidence formats, Attestation Result formats, object security mechanisms, and transport protocols.

Encrypting Evidence for the Verifier protects confidentiality but does not by itself prove that the Verifier is authorized to receive the requested claims.
Attesters and Presenters MUST authenticate the Verifier's encryption credential and check it against the applicable trust anchor store and claim-category authorization policy before releasing sensitive Evidence.
Furthermore, this provides only implicit authentication of the Verifier under the assumption that only a Verifier in possession of the corresponding private key will be able to decrypt the Evidence instead of an explicit authentication step prior to transmitting the Evidence. For most deployments, this difference will be inconsequential.

Evidence encryption needs integrity and context binding.
If a ciphertext can be replayed, redirected to a different Verifier, or detached from the Relying Party challenge that caused its creation, then an attacker might cause stale or unintended Evidence to be appraised.
Deployments MUST provide freshness and SHOULD bind ciphertexts to protocol context using protected headers, AAD, or equivalent mechanisms.

Relying Parties might attempt to request more detail than needed, coerce holders into revealing more selectively disclosable claims, or correlate Attesters using stable metadata.
Verifier and holder policies SHOULD reject excessive disclosure requests.
Implementations SHOULD provide a way for the Attester, device owner, or administrator to understand and control which Relying Parties receive sensitive Attestation Result claims.

Selective Disclosure and ZKP-based Attestation Results require careful algorithm agility and validation rules.
Relying Parties MUST validate issuer signatures, proof algorithms, key binding, audience, nonce or freshness values, and claim semantics.
Failure to validate any of these can turn a privacy-preserving presentation into a replayable bearer token or an ambiguous proof.

Object encryption and TLS do not prevent an authorized recipient from retaining data after decryption.
Deployments handling sensitive Evidence or Attestation Results SHOULD define retention, logging, and audit requirements for Verifiers and Relying Parties.
Where the content of the claims could be subject to legal data retention or data handling regulations, decryption and processing of the Evidence SHOULD be done within a contained Verifier module.


# Privacy Considerations

The main privacy principle in this document is categorization of claims according to sensitivity level, protection of Evidence against passive eavesdroppers, and recipient-specific data minimization.
Each party SHOULD receive only the Evidence, Attestation Result claims, or proof outputs needed for its role.
Deployments need to evaluate each protocol role separately because a privacy control at one release point can be undone by excessive disclosure at a later release point.

The Background-Check Model has greater Evidence-disclosure risk than the Passport Model because the Relying Party is on the Evidence path.
Encrypting Evidence for the Verifier, and proper authentication of the Verifier's encryption key against a trust store, is therefore especially important in the Background-Check Model.
However, encrypted Evidence does not help if the Attestation Result later reveals the same details to the Relying Party.

Attestation Results can create linkability even when they do not contain obvious identifiers.
Combinations of claim values, credential type, issuer, issuance time, validity window, policy identifiers, and proof metadata can become a fingerprint.
In the case where the Evidence is used to issue a credential such as an X.509 certificate {{?I-D.ietf-lamps-csr-attestation}}, that certificate acts as an Attestation Result, and the credential itself becomes a fingerprint that can be used to link any subsequent use of the credential back to the Attester.
Profiles of this framework SHOULD consider pairwise identifiers, short validity periods, unlinkable presentations, selective disclosure, and ZKP-based predicates where appropriate.
Pairwise identifiers reduce correlation by using different identifiers for different Relying Parties or audiences instead of a globally stable Attester identifier.
Short validity periods, nonces, and audience restrictions reduce the value of an Attestation Result as a replayable or long-lived correlation handle.
Unlinkable presentations, where supported by the selected token or proof format, reduce the ability of Relying Parties to determine that two presentations came from the same Attester unless that linkage is intentionally disclosed.

Selective Disclosure and ZKP-based predicates can reduce the amount of claim information disclosed to a Relying Party, but they do not by themselves remove all linkability.
Deployments still need to minimize stable metadata, issuer identifiers, key-binding material, credential type identifiers, timestamps, and proof parameters that remain visible outside the selectively disclosed claims or proof.


# IANA Considerations

TBD: The EKU OID `id-kp-tbd-evidence-encryption` needs to be registered here

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
