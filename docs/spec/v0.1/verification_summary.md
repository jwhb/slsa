---
title: SLSA Verification Summary Attestation (VSA)
description: SLSA v0.1 specification for a verification summary of artifacts by a trusted verifier entity.
layout: standard
---

Verification summary attestations communicate that an artifact has been verified
at a specific SLSA level and details about that verification.

This document defines the following predicate type within the [in-toto
attestation] framework:

```json
"predicateType": "https://slsa.dev/verification_summary/v0.1"
```

> Important: Always use the above string for `predicateType` rather than what is
> in the URL bar. The `predicateType` URI will always resolve to the latest
> minor version of this specification. See [parsing rules](#parsing-rules) for
> more information.

## Purpose

Describe what SLSA level an artifact or set of artifacts was verified at
and other details about the verification process including what SLSA level
the dependencies were verified at.

This allows software consumers to make a decision about the validity of an
artifact without needing to have access to all of the attestations about the
artifact or all of its transitive dependencies.  They can use it to delegate
complex policy decisions to some trusted party and then simply trust that
party's decision regarding the artifact.

It also allows software publishers to keep the details of their build pipeline
confidential while still communicating that some verification has taken place.
This might be necessary for legal reasons (keeping a software supplier
confidential) or for security reasons (not revealing that an embargoed patch has
been included).

## Model

A Verification Summary Attestation (VSA) is an attestation that some entity
(`verifier`) verified one or more software artifacts (the `subject` of an
in-toto attestation [Statement]) by evaluating the artifact and a `bundle`
of attestations against some `policy`.  Users who trust the `verifier` may
assume that the artifacts met the indicated SLSA level without themselves
needing to evaluate the artifact or to have access to the attestations the
`verifier` used to make its determination.

The VSA also allows consumers to determine the verified levels of
all of an artifact’s _transitive_ dependencies.  The verifier does this by
either a) verifying the provenance of each non-source dependency listed in
the [materials](/provenance/v0.2#materials) of the artifact
being verified (recursively) or b) matching the non-source dependency
listed in `materials` (by subject.digest == materials.digest and, ideally,
subject.name == materials.uri) to a VSA _for that dependency_ and using
`vsa.policy_level` and `vsa.dependency_levels`.  Policy verifiers wishing
to establish minimum requirements on dependencies SLSA levels may use
`vsa.dependency_levels` to do so.

## Schema

```jsonc
// Standard attestation fields:
"_type": "https://in-toto.io/Statement/v0.1",
"subject": [{
  "name": <artifact-URI-in-request>,
  "digest": { <digest-in-request> }
}],

// Predicate
"predicateType": "https://slsa.dev/verification_summary/v0.1",
"predicate": {
  // Required
  "verifier": {
    "id": "<URI>"
  },
  "time_verified": <TIMESTAMP>,
  "policy": {
    "uri": "<URI>",
    "digest": { /* DigestSet */ }
  }
  "verification_result": "<PASSED|FAILED>",
  "policy_level": "<SlsaResult>",
  "dependency_levels": {
    "<SlsaResult>": <Int>,
    "<SlsaResult>": <Int>,
    ...
  }
}
```

### Parsing rules

This predicate follows the in-toto attestation [parsing rules]. Summary:

-   Consumers MUST ignore unrecognized fields.
-   The `predicateType` URI includes the major version number and will always
    change whenever there is a backwards incompatible change.
-   Minor version changes are always backwards compatible and "monotonic." Such
    changes do not update the `predicateType`.
-   Producers MAY add extension fields using field names that are URIs.

### Fields

_NOTE: This section describes the fields within `predicate`. For a description
of the other top-level fields, such as `subject`, see [Statement]._

<a id="verifier"></a>
`verifier` _object, required_

> Identifies the entity that performed the verification.
>
> The identity MUST reflect the trust base that consumers care about. How
> detailed to be is a judgment call.
>
> Consumers MUST accept only specific (signer, verifier) pairs. For example,
> "GitHub" can sign provenance for the "GitHub Actions" verifier, and "Google"
> can sign provenance for the "Google Cloud Deploy" verifier, but "GitHub" cannot
> sign for the "Google Cloud Deploy" verifier.
>
> The field is required, even if it is implicit from the signer, to aid readability and
> debugging. It is an object to allow additional fields in the future, in case one
> URI is not sufficient.

<a id="verifier.id"></a>
`verifier.id` _string ([TypeURI]), required_

> URI indicating the verifier’s identity.

<a id="time_verified"></a>
`time_verified` _string ([Timestamp]), required_

> Timestamp indicating what time the verification occurred.

<a id="policy"></a>
`policy` _object, required_

> Describes the policy that was used to verify this artifact.

<a id="policy.uri"></a>
`policy.uri` _string ([ResourceURI]), required_

> The URI of the policy used to perform verification.

<a id="policy.digest"></a>
`policy.digest` _object ([DigestSet]), optional_

> Collection of cryptographic digests for the contents of the policy used to perform verification.

<a id="verification_result"></a>
`verification_result` _string, required_

> Either “PASSED” or “FAILED” to indicate if the artifact passed or failed the policy verification.

<a id="policy_level"></a>
`policy_level` _string ([SlsaResult]), required_

> Indicates what SLSA level the artifact itself (and not its dependencies) was verified at or "FAILED" if policy verification failed.

<a id="dependency_levels"></a>
`dependency_levels` _object, required_

> A count of the dependencies at each SLSA level.
>
> Map from [SlsaResult] to the number of the artifact's _transitive_ dependencies
> that were verified at the indicated level. Absence of a given level of
> [SlsaResult] MUST be interpreted as reporting _0_ dependencies at that level.

## Example

WARNING: This is just for demonstration purposes.

```jsonc
"_type": "https://in-toto.io/Statement/v0.1",
"subject": [{
  "name": "https://example.com/example-1.2.3.tar.gz",
  "digest": {"sha256": "5678..."}
}],

// Predicate
"predicateType": "https://slsa.dev/verification_summary/v0.1",
"predicate": {
  "verifier": {
    "id": "https://example.com/publication_verifier"
  },
  "time_verified": "1985-04-12T23:20:50.52Z",
  "policy": {
    "uri": "https://example.com/example_tarball.policy",
    "digest": {"sha256": "1234..."}
  },
  "verification_result": "PASSED",
  "policy_level": "SLSA_LEVEL_3",
  "dependency_levels": {
    "SLSA_LEVEL_4": 1,
    "SLSA_LEVEL_3": 5,
    "SLSA_LEVEL_2": 7,
    "SLSA_LEVEL_1": 1,
  }
}
```

<div id="slsaresult">

## _SlsaResult (String)_

</div>

The result of evaluating an artifact (or set of artifacts) against SLSA.
Must be one of these values:

-   SLSA_LEVEL_0
-   SLSA_LEVEL_1
-   SLSA_LEVEL_2
-   SLSA_LEVEL_3
-   SLSA_LEVEL_4
-   FAILED (Indicates policy evaluation failed)

## Change history

-   0.1: Initial version.

[SlsaResult]: #slsaresult
[DigestSet]: https://github.com/in-toto/attestation/blob/main/spec/v0.1.0/field_types.md#DigestSet
[ResourceURI]: https://github.com/in-toto/attestation/blob/main/spec/v0.1.0/field_types.md#ResourceURI
[Statement]: https://github.com/in-toto/attestation/blob/main/spec/v0.1.0/README.md#statement
[Timestamp]: https://github.com/in-toto/attestation/blob/main/spec/v0.1.0/field_types.md#Timestamp
[TypeURI]: https://github.com/in-toto/attestation/blob/main/spec/v0.1.0/field_types.md#TypeURI
[in-toto attestation]: https://github.com/in-toto/attestation
[parsing rules]: https://github.com/in-toto/attestation/blob/main/spec/v0.1.0/README.md#parsing-rules
