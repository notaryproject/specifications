# Trust Store and Trust Policy Specification

Notary v2 currently supports X509 based PKI and identities, and uses a trust store and trust policy to determine if a signed artifact is considered authentic.

The document consists of the following sections:

- **[Trust Store](#trust-store)**: Contains a set of trusted identities through which trust is derived for the rest of the system. For X509 PKI, the trust store contains a set of root certificates.
- **[Trust Policy](#trust-policy)**: A policy language which indicates the which identities are trusted to produce artifacts.
- **[Signature Evaluation](#signature-evaluation)**: Describes how signatures are evaluated using the policy, to determine if a signed artifact is authentic.

Other types of identities and trust models may be supported in future, which may introduce other constructs/policy elements to support signature evaluation.

All examples use the actors defined in Notary v2 [scenario](https://github.com/notaryproject/notaryproject/blob/main/scenarios.md#scenario-0-build-publish-consume-enforce-policy-deploy)

- Wabbit Networks company builds, signs and distributes their `net-monitor` software though public registries.
- ACME Rockets consumes the `net-monitor` software from a public registry importing the  artifacts and reference artifacts (signatures, SBoMs) into their private registry. The private registry also contains additional artifacts that ACME Rockets themselves sign.

## Trust Store

Contains a set of trusted identities through which trust is derived for the rest of the system. For X509 PKI, the trust store typically contains root certificates.

- The Notary v2 trust store consists of multiple named collections of certificates, called named stores.
- The trust store is a directory location, under which each sub directory is considered a named store, that contains zero or more certificates. The name of this sub directory is used to reference the specfic store in trust policy.
- Certificates in a trust store are typically root certificates. Intermediate certificates can be used, but not recommended as intermediate certificates can be rotated more often and can break signature verification.

Notary v2 uses following directory structure to represent the trust store. The example shows named stores `acme-rockets` and `wabbit-networks`, which are subseqently references in the trust policy. Without this reference, presence of a named store and certificates in it does not confer trust automatically to the named store. The trust store is configured ahead of verification time, by an out of band mechanism that is beyond the scope of this document. Different entities and organizations have their own processes and policies to configure and distribute trust stores.

```text
~/.notation/trust_store
    /x509
        /ca_roots
            /acme-rockets
                cert1.pem
                cert2.pem
                  /sub-dir       # ignored
                    cert-3.pem   # ignored
            /wabbit-networks
                cert3.pem
        /tsa_roots
            /publicly-trusted-tsa
                tsa-cert1.pem
```

The Trust store currently supports two kinds of identities, additional identities may be supported in future :

- **Certificates**: The `x509/ca` trust store contains named stores that contain Certificate Authority (CA) root and intermediate certificates.
- **Timestamping Certificates**: The `x509/tsa_roots` trust store contains named stores with Time Stamping Authority (TSA) root and intermediate certificates. **NOTE** TSA based timestamping will not be available in Notation RC1.

Any additional sub directories under names stores and certificates in it are ignored.

## Trust Policy

Describes how signatures are evaluated using the policy, to determine if a signed artifact is authentic. Users who consume and execute the signed artifact from a registry need a mechanism to specify how the artifacts should be evaluated for trust, this is where a trust policy is used.
Trust policy allows users to control the artifact's integrity, expiry, and revocation aspect of signature evaluation.

### Trust Policy Schema

The trust policy is as JSON document, example shown below:

```jsonc
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "policy-for-wabbit-networks-images",
            "registryScopes": [
                "registry.acme-rockets.io/software/net-monitor"
                "registry.acme-rockets.io/software/net-logger" ],
            "signatureVerification" : "strict",
            "trustStore": "ca/wabbit-networks",
            "trustedIdentity": [
                "x509.subject: C=US, ST=WA, L=Seattle, O=wabbit-networks.io"
            ],
        },
        {
            "name": "policy-for-unsigned-image",
            "registryScopes": [ "registry.wabbit-networks.io/software/unsigned/productA" ],
            "signatureVerification": "skip",
        },
        {
            "name": "global-policy-for-all-other-images",
            "registryScopes": [ "*" ],
            "signatureVerification" : "audit",
            "trustStores": "ca/acme-rockets"
        }
    ]
}
```

### Trust Policy Properties

- **`version`**(*string*): This REQUIRED property is the version of the trust policy.
  The supported value is `1.0`.
- **`trustPolicies`**(*string-array of objects map*): This REQUIRED property represents a collection of trust policies.
  - **`name`**(*string*): This REQUIRED propert represents the name of the trust policy.
  - **`registryScopes`**(*array of strings*): This REQUIRED property determines which trust policy is applicable for the given artifact.
    The scope field supports filtering based on fully qualified repository URI `${registry-name}/${namespace}/${repository-name}`.
    For more information, see [scopes constraints](#scope-constraints) section.
  - **`signatureVerification`**(*string*): This REQUIRED property dictates how signature verification is performed. Supported values are `strict`, `permissive`, `audit` and `skip`. Detailed explaination of each value is present [here](#signatureverification-details).
  - **`trustStore`**(*string*): This REQUIRED property specifies named trust store.
  - **`trustAnchors`**(*array of strings*): This OPTIONAL property specifies a list of elements/attributes of the signing certificate's subject.
    If present, the collection MUST contain at least one value.
    For more information, see [trust anchors constraints](#trust-anchors-constraints) section.

#### Signature Verification details

- `strict` : Signature verification is performed at `strict` level, which enforces all of the signature verification checks - integrity, authenticity, revocation check. If any of these checks fails, the signature verification fails. This is the recommended level in environments where a signature verification failure does not have high impact to other concerns (like application avaiability). It is recommended that build and development environments where images are initially injested, or at high assurance at deploy time  use `strict` level.
- `permissive` : The `permissive` level enforces integrity and authenticity, but will only audit for revocation and expiry, whose failures are only logged. The `permissive` level is recommended to be used if signature verification is done at deploy time or runtime, and the user only needs integrity and authenticity guarantees.
- `audit` : The `audit` level only enforces integrity check if a signature is present. Failure of all other checks are only logged.
- `skip` : The `skip` level does not fetch signatures for artifacts and does not perform any signature verification. This is useful when an application uses multiple artifacts, and has a mix of signed and unsigned artifacts. Note that `skip` cannot be used with a global scope (`*`), the value of `registryScopes` MUST contain fully qualified registry URL(s).

|Level     |Recommended Usage|Integrity|Authenticity|Trusted timestamp|Expiry|Revocation check|
|----------|-----------------|---------|------------|-----------------|------|----------------|
|strict    |Use at development, build and deploy time|Enforce|Enforce|Enforce|Enforce|Enforce|
|permissive|Use at deploy time or runtime|Enforce|Enforce|Audit|Audit|Audit|
|audit     |Use when adopting signed images, without breaking existing workflows|Enforce|Audit|Audit|Audit|Audit|
|skip      |Use to exclude verification for unsigned images|Skip|Skip|Skip|Skip|Skip|

**Integrity** : Guarantees that the artifact wasn't altered after it was signed. All signature verifications levels always enforce integrity. Invalid signatures are rejected.
**Authenticity** : Guarantees that the artifact was signed by an identity trusted by the verifier. Its definition does not include revocation, which is when a trusted identity is subsequently untrusted because of a compromise.
**Trusted timestamp** : Guarantees that the signature was generated when the certificate was valid. It also allows a verifier to determine if a signature be treated as valid when a certificate is revoked, if the certificate was revoked after the signature was generated. In the absence of a trusted timestamp, signatures are considered invalid after certificate expires, and all signatures are considered revoked when a certificate is revoked.
**Expiry** : This is an optional feature that gurantees that an artifact is considered fresh, if the signing identity indicates an optional artifact expiry time.
**Revocation check** : Guarantees that the signing identity is still trusted at signature verification time. Events such as key or system compromise can make a signing identity that was previously trusted, to be subsequrntly untrusted. This guarantee typically requires a verification time call to an external system, which may not be consistently reliable. The `permissive` verification level only audits for revocation check and does not enforce it. If a particular revocation mechnanism provides is reliable, use `strict` verification level instead.

#### Scopes Constraints

- Each trust policy MUST contain scope property and the scope collection MUST contain at least one value.
- The scope MUST contain one of the following:
  - List of one or more fully qualified repository URIs.
    The repository URI MUST NOT contain the asterisk character `*`.
  - A single value with one asterisk character `*`.
    The scope with `*` value is called global scope.
    The trust policy with global scope applies to all the artifacts.
    There can only be one trust policy that uses a global scope.
- For a given artifact there MUST be only one applicable trust policy, except for trust policy with global scope.
- For a given artifact, if there is no applicable trust policy then Notary v2 MUST consider the artifact as untrusted and fail signature verification.
- The scope MUST NOT support reference expansion i.e. URIs must be fully qualified.
  E.g. the scope should be `docker.io/library/registry` rather than `registry`.
- Evaluation order of trust policies:
  1. *Exact match*: If there exists a trust policy whose scope contains the artifact's repository URI then the aforementioned policy MUST be used for signature evaluation.
     Otherwise, continue to the next step.
  1. *Global*: If there exists a trust policy with global scope then use that policy for signature evaluation.
     Otherwise, fail the signature verification.

### Trust Anchors Constraints

A distinguished name (usually just shortened to "DN") uniquely identifies an entry and in the case of the certificate's subject, DN uniquely identifies the requestor/holder of the certificate.
The DN is comprised of zero or more comma-separated components called relative distinguished names, or RDNs.
For example, the DN `C=US, ST=WA, O=wabbit-network.io, OU=org1`"` has four RDNs.
The RDN consists of an attribute type name followed by an equal sign and the string representation of the corresponding attribute value.

- Trust anchor MUST support a full and partial list of all the attribute types present in [subject DN](https://www.rfc-editor.org/rfc/rfc5280.html#section-4.1.2.6) of x509 certificate.
- If the subject DN of the signing certificate is used in the trust anchor, then it MUST meet the following requirements:
  - The value of `trustAnchors` MUST begin with `subject:` followed by comma-separated one or more RDNs.
    For example, `x509.subject: C=${country}, ST=${state}, L=${locallity}, O={organization}, OU=${organization-unit}, CN=${common-name}`.
  - Trust anchor MUST contain country (CN), state Or province (ST), and organization (O) RDNs.
    All other RDNs are optional.
    The minimal possible trust anchor is `subject: C=${country}, ST=${state}, O={organization}`,
  - Trust anchor MUST NOT have overlapping values.
    Trust anchors are considered overlapping if there exists a certificate for which multiple trust anchors evaluate true.
    For example, the following two trust anchors are overlapping:
    - `x509.subject: C=US, ST=WA, O=wabbit-network.io, OU=org1`
    - `x509.subject:  C=US, ST=WA, O=wabbit-network.io`
  - In some special cases trust anchor MUST escape one or more characters in an RDN.
    Those cases are:
    - If a value starts or ends with a space, then that space character MUST be escaped as `\`.
    - All occurrences of the comma character (`,`) MUST be escaped as `\,`.
    - All occurrences of the semicolon character (`;`) MUST be escaped as `\;`.
    - All occurrences of the backslash character (`\`) MUST be escaped as `\\`.

### Extended Validation

The implementation must allow the user to execute custom validations.
These custom validation MUST have access to all the information available in the signature envelope like payload, signed attributes, unsigned attributes, and signature.

## Signature Evaluation

### Prerequisites

- User has configured a valid [trust store](#trust-store) and [trust policy](#trust-policy).

### Steps

1. **Validate that the signature envelope format is supported.**
    1. Parse the signature envelope content based on the signature envelope type specified in the `[descriptors].descriptor.mediaType` attribute of the signature artifact manifest.
    1. Validate that the content type indicated by the `cty` property value of protected headers in the signature envelope is supported.
1. **Validate the signature envelope integrity.**
    1. Get the signing certificate from the parsed [signature envelope](https://github.com/notaryproject/notaryproject/blob/7b7d283038/signature-specification.md#signature-envelope).
    1. Get the signing algorithm(hash+encryption) from the signing certificate and validate that the signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements)
    1. Using the public key of the signing certificate and signing algorithm identified in the previous step, validate the integrity of the signature envelope.
1. **Validate the signature against trust policy and trust store.**
    1. Using the `scope` configured in trust policies, get the applicable trust policy.
       (Implementations might have this value precomputed, added it for completeness)
    1. For the applicable trust policy, **validate trust-store:**
        1. Validate that the signature envelope contains a complete certificate chain that starts from a code signing certificate and terminates with the root certificate.
           Also, validate that code signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
        1. For each of the trust-stores configured in applicable trust-policy perform the following steps.
            1. Validate that certificate and certificate-chain lead to a trusted certificate configured in the `x509Certs` field of trust-store.
            1. If the above verification succeeds then continue to the next step else iterate over the next trust store.
               If all of the trust stores have been evaluated then fail the signature validation and exit.
    1. For the applicable trust policy, **validate trust anchors** (if present):
        1. If trust anchors are present, validate that the value of subject attributes configured in `trustAnchors` matches with the value of corresponding attributes in the signing certificate’s subject.
           If trust anchors are not present continue to step 4.
        1. If the above verification succeeds then continue to the next step.
           Otherwise, fail the signature validation and exit.
    1. **Validate trust policy:**
        1. If signature expiry is present in the signature envelope, using the local machine’s current time(in UTC) check whether the signature is expired or not.
           If the signature is not expired, continue to the next step.
           Otherwise, if `signatureExpiry` is set to `Enforce` then fail the signature validation and exit else log a warning and continue to the next step.
        1. Check for the timestamp signature in the signature envelope.
            1. If the timestamp exists, continue with the next step.
               Otherwise, store the local machine's current time(in  UTC) in variables `timeStampLowerLimit` and `timeStampUpperLimit` and continue with step 3.3.c.
            1. Validate that the timestamp hash in `TSTInfo.messageImprint` matches the hash of the signature to which the timestamp was applied.
            1. Validate that the timestamp signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
            1. Validate that the timestamp signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
            1. Validate the `signing-certificate`([RFC-2634](https://tools.ietf.org/html/rfc2634)) or `signing-certificate-v2`([RFC-5126](https://tools.ietf.org/html/rfc5126#section-5.7.3.2)) attribute of timestamp CMS.
            1. Check whether timestamping certificate and certificate chain are valid (not expired) or not.
               If timestamping certificate and certificate-chain are not expired, continue to the next step.
               Otherwise, if `timestampExpiry` in trust-policy is configured to `Enforce` then fail the signature validation and exit, else log a warning and continue to the next step.
            1. Validate that timestamp certificate and certificate chain leads to a trusted TSA certificate configured in trust policy.
            1. Validate timestamp certificate and certificate chain revocation status  using [certificate revocation evaluation](#certificate-revocation-evaluation) section as per `timestampRevocation` setting in trust-policy
            1. Retrieve the timestamp's time from `TSTInfo.genTime`.
            1. Retrieve the timestamp's accuracy.
               If the accuracy is explicitly specified in `TSTInfo.accuracy`, use that value.
               If the accuracy is not explicitly specified and `TSTInfo.policy` is the baseline time-stamp policy([RFC-3628](https://tools.ietf.org/html/rfc3628#section-5.2)), use accuracy of 1 second.
               Otherwise, use an accuracy of 0.
            1. Calculate the timestamp range using the lower and upper limits per [RFC-3161 section 2.4.2](https://tools.ietf.org/html/rfc3161#section-2.4.2) and store the limits as `timeStampLowerLimit` and `timeStampUpperLimit` variables respectively.
        1. Check that the time range from `timeStampLowerLimit` to `timeStampUpperLimit` timestamp is entirely within the certificate's validity period.
           If the time range is entirely within the signing certificate and certificate chain's validity period, continue to the next step.
           Otherwise, If `signingIdentityExpiry` in trust-policy is configured to `Enforce` then fail the signature validation and exit else log a warning and continue to the next step.
        1. Validate signing identity(certificate and certificate chain) revocation status using [certificate revocation evaluation](#certificate-revocation-evaluation) section as per `signingIdentityRevocation` setting in trust-policy.
        1. Perform extended validation using the applicable(if any) plugin.
        1. If you have reached this step then treat the OCI artifact signature as a valid signature.

### Certificate Revocation Evaluation

If the certificate revocation trust-store setting is set to `skip`, skip the below steps.
otherwise, check for revocation status for certificate and certificate chain.

1. If the revocation status of any of the certificates cannot be determined (revocation unavailable) and `signingIdentityRevocation` is set to either `enforceWithFailOpen` or `warn` then log a warning and skip the below steps.
   Otherwise, fail the signature validation and exit.
1. If any of the certificates are revoked and `signingIdentityRevocation` is set to either `enforceWithFailOpen` or `enforceWithFailClose` then fail signature validation and exit else log a warning.

Starting from Root to leaf certificate, for each certificate in the certificate chain, perform the following steps to check its revocation status:

- If the certificate being validated doesn't include information OCSP or CRLs then no revocation check is performed and the certificate is considered valid(not revoked).
- If the certificate being validated includes either OCSP or CRL information, then the one which is present is used for revocation check.
- If both OCSP URLs and CDP URLs are present, then OCSP is preferred over CRLs.
  If revocation status cannot be determined using OCSP because of any reason such as unavailability then fallback to using CRLs for revocation check.

#### CRLs

There are two types of CRLs (per RFC 3280), Base CRLs and Delta CRls.

- **Base CRLs:** Contains the revocation status of all certificates that have been issued by a given CA.
  BaseCRLs are signed by the certificate issuing CAs.
- **Delta CRLs:** Contains only certificates that have changed status since the last base CRL was published.
  Delta CRLs are signed by the certificate issuing CAs.
- **Indirect CRLs:** Special Case in which BaseCRLs and Delta CRLs are not signed by the certificate issuing CAs, instead they are signed with a completely different certificate.

Notary v2 MUST support BaseCRLs and Delta CRLs.
Notary v2 MAY support Indirect CRLs.
Notary v2 supports only HTTP CRL URLs.

Implementations MAY add support for caching CRLs and OCSP response to improve availability, latency and avoid network overhead.

##### CRL Download

CRL download location (URL) can be obtained from the certificate's CRL Distribution Point (CDP) extension.
If the certificate contains multiple CDP locations then each location download is attempted in sequential order.
For each CDP location, Notary V2 will try to download the CRL for the default threshold of 10 seconds.
If the CRL cannot be downloaded within the timeout threshold the revocation result will be "revocation unavailable".
The user may be able to configure this threshold.

##### Revocation Checking with CRL

To check the revocation status of a certificate against CRL, the following steps must be performed:

1. Verify the CRL signature.
1. Verify that the CRL is valid (not expired).
   A CRL is considered expired if the current date is after the `NextUpdate` field in the CRL.
1. Look up the certificate’s serial number in the CRL.
    1. If the certificate’s serial number is listed in the CRL, look for `InvalidityDate`.
       If CRL has an invalidity date and artifact signature is timestamped then compare the invalidity date with the timestamping date.
       If the invalidity date is before the timestamping date, the certificate is considered revoked.
       If the invalidity date is not present in CRL, the certificate is considered revoked.
    1. If the CRL is expired and the certificate is listed in the CRL for any reason other than `certificate hold`, the certificate is considered revoked.
    1. If the certificate is not listed in the CRL or the revocation reason is `certificate hold`, a new CRL is retrieved if the current time is past the time in the `NextUpdate` field in the current CRL.
       The new CRL is then checked to determine if the certificate is revoked.
       If the original reason was `certificate hold`, the CRL is checked to determine if the certificate is unrevoked by looking for the `RemoveFromCRL` revocation code.

##### Revocation Checking with Delta CRLs

If a delta CRL exists for a base CRL, the `Freshest CRL` extension in the base CRL provides the URL from where the delta CRL can be downloaded.
The delta CRLs have their identifying number, plus the "Delta CRL Indicator" extension indicating the version number of the base CRL that must be used with the delta CRL.

When delta CRLs are implemented, the following results can occur during revocation checking.

- If the delta CRL cannot be retrieved for some reason, the revocation result will be "revocation unavailable".
- If the delta CRL’s indicator is less than the current base CRL, the revocation result returned will be "revocation unavailable"
- To check revocation of a certificate in Delta CRLs follow steps similar to [Revocation Checking with CRL](#revocation-checking-with-crl).

#### OCSP

##### OCSP Download

OCSP URLs can be obtained from the certificate's authority information access (AIA) extension as defined in [RFC-2560](https://datatracker.ietf.org/doc/html/rfc2560).
Notary V2 will wait for a default threshold of 5 seconds to receive an OCSP response.
If OCSP response is not available within the timeout threshold the revocation result will be "revocation unavailable".
The user may be able to configure this threshold.

##### Revocation Checking with OCSP

To check the revocation status of a certificate using OCSP, the following steps must be performed.

1. Verify the signature of the OCSP response.
   This step also includes verifying the OSCP response signing certificate is valid and trusted.
   The OCSP signing certificate must be issued by the same CA as the certificate being verified or the OCSP response must be signed by the issuing CA.
1. Verify that the OCSP response is valid (not expired).
   A CRL is considered expired if the current date is after the `NextUpdate` field in the CRL.
1. Verify that the OCSP response indicates that the certificate is not revoked i.e `CertStatus` is `good`.
    1. If the certificate is revoked i.e `CertStatus` is `revoked`, look for `InvalidityDate`.
       If the invalidity date is present and timestamp signature is also present then if the invalidity date is before the timestamping date, the certificate is considered revoked.
       If the invalidity date is not present in OCSP response, the certificate is considered revoked.
1. If `id-pkix-ocsp-nocheck`(1.3.6.1.5.5.7.48.1.5) extension is not present on the OCSP signing certificate then revocation checking must be performed using CRLs for the OCSP signing certificate.

## FAQ

**Q: Does Notary v2 supports `n` out of `m` signatures verification requirement?**

**A:** Notary v2 doesn't support n out m signature requirement verification scheme.
Signature verification workflow succeeds if verification succeeds for at least one signature.

**Q: Does Notary v2 support overriding of revocation endpoints to support signature verification in disconnected environments?**

**A:** Not natively supported but a user can configure `revocationValidations` to `skip` and then use extended validations to check for revocation.

**Q: Why user needs to include a complete certificate chain (leading to root) in the signature?**

**A:** Without a complete certificate chain, the implementation won't be able to perform an exhaustive revocation check, which will lead to security issues, and that's the reason for enforcing a complete certificate chain.

**Q: Why are we validating artifact signature first instead of signing identity?**

**A:** Ideally, we should validate the signing identity first and then use the public key in the signing identity to validate the artifact signature.
However, this will lead to poor performance in the case where the signature is not valid as there are lots of validations against the signing identity including network calls for revocations, and possibly we won't even need to read the trust store/trust policy if the signature validation fails.
Also, by validating artifact signature first we will still fail the validation if the signing identity is not trusted.
