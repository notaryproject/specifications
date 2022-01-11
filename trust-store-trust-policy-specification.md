# Trust Store and Trust Policy Specification

This document describes how Notary v2 signatures are evaluated for trust. The document consists of following sections:

- **[Trust Store](#trust-store)**: Defines set of signing identity that user trusts.
- **[Trust Policy](#trust-policy)**: Defines how the artifact is evaluated for trust.

## Trust Store

Users who consume and execute the signed artifact from a registry need a mechanism to specify the trusted producers. This is where Trust Store is used.

Trust store allows users to specify two kinds of identities:

- **Certificates**: These are the signing certificates or certificates that chains to root CA.
- **Timestamping Certificates**: These are the timestamping certificates or certificates that chain to the root certificate of TSA.

The trust store is represented as JSON data structure as shown below:

```json
{
    "version": "1.0",
    "trustStores": {
        "trust-store-name-1": {
            "identities": {
                "x509Certs": [
                    "-----BEGIN CERTIFICATE-----\ncertificate1\n-----END CERTIFICATE-----\n",
                    "-----BEGIN CERTIFICATE-----\ncertificate2\n----END CERTIFICATE-----\n"
                ],
                "tsaX509Certs": [
                    "-----BEGIN CERTIFICATE-----\ntsaCertificate1\n-----END CERTIFICATE-----\n",
                    "-----BEGIN CERTIFICATE-----\ntsaCertificate2\n-----END CERTIFICATE-----\n"
                ]
            }
        },
        "trust-store-name-2": {
            "identities": {
                "x509Certs": [
                    "-----BEGIN CERTIFICATE-----\ncertificate1\n-----END CERTIFICATE-----\n",
                    "-----BEGIN CERTIFICATE-----\ncertificate2\n----END CERTIFICATE-----\n"
                ],
                "tsaX509Certs": [
                    "-----BEGIN CERTIFICATE-----\ntsaCertificate2\n-----END CERTIFICATE-----\n",
                    "-----BEGIN CERTIFICATE-----\ntsaCertificate3\n----END CERTIFICATE-----\n"
                ]
            }
        }
    }
}
```

Property Description:

- **`version`**(*string*): This REQUIRED property is the version of the trust store. The supported value is `1.0`
- **`trustStores`**(*object*): This REQUIRED property represents the parent node containing multiple trust stores. Each trust store is identified by the key associated with it like 'trust-store-name-1', 'trust-store-name-2'.
  - **`identities`**(*object*): This REQUIRED property represents the collection of different types of identities. There are two types of identifies that Notary v2 supports: x509 certificates and x509 timestamping certificates.
    - **`x509Certs`**(*array of strings*): This REQUIRED property specifies a list of x509 certificates in PEM format. The collection MUST contain at least one certificate.
    - **`tsaX509Certs`**(*array of strings*): This OPTIONAL property specifies a list of x509 timestamping certificates in PEM format. If the `tsaX509Certs` key is present then collection MUST contain at least one timestamping certificate.

## Trust Policy

Users who consume and execute the signed artifact from a registry need a mechanism to specify how the artifacts should be evaluated for trust, this is where a trust policy is used.
Trust policy allows users to control the artifact's integrity, expiry, and revocation aspect of signature evaluation.

### Artifact Integrity

The presence of a trust policy indicates that implementation MUST validate that artifact is signed and has not been altered.

### Artifact Expiry

Trust policy allows users to define how the system should behave when the artifact's signature is expired or signing identity is expired or timestamping identity is expired.

If artifact expiry validations are enforced the implementation MUST perform the following validations:

1. If signature expiry is present then signature MUST NOT be expired.
1. If signing identity is certificate and
    1. signing certificate and certificate chain are not expired then the implementation MUST ignore the timestamping signature even if it is present in the signature.
    1. signing certificate or certificate chain are expired then the implementation MUST validate that signature is timestamped and timestamping signature is valid. Also, validate that timestamping certificate and certificate chain MUST NOT be expired.

### Artifact Revocation

Trust policy also allows users to control how the system should behave when signing identity or  timestamping identity is revoked

If revocation validations are enforced implementation MUST perform the following validations:

1. If signing identity is a certificate
   1. then signing certificate and certificate-chain MUST NOT be revoked.
   1. and if the signing certificate and certificate chain are not expired then the implementation MUST ignore the revocation check for timestamping signature even if it's present in the signature.
   1. and if the signing certificate and certificate chain is expired then the implementation MUST validate that signature is timestamped and timestamping signature is valid. Also, validate that timestamping certificate and certificate chain MUST NOT be revoked.

The implementation MUST support both [OCSP](https://datatracker.ietf.org/doc/html/rfc6960) and [CRL](https://datatracker.ietf.org/doc/html/rfc5280) based revocations. Since revocation check requires network call and network call can fail because of a variety of reasons such as revocation endpoint is unavailable, network connectivity issue, DDoS attack, etc the implementation MUST support both `fail-open` or `fail-close` use cases.

- `fail-open`: If revocation endpoint is not reachable, consider artifact as revoked.
- `fail-close`: If revocation endpoint is not reachable, consider artifact as not revoked.

The trust policy is represented as JSON data structure as shown below:

```json
{
    "version": "1.0",
    "trustPolicies": [
        {
            "name": "verify-signature",
            "scopes": [
                "wabbit-networks.io/software/product1"
                "wabbit-networks.io/software/product2" ],
            "trustStores": [ "trust-store-name-1", "trust-store-name-2" ],
            "expiryValidations": {
                "signatureExpiry": "enforce | warn",
                "signingIdentityExpiry": "enforce | warn",
                "timestampExpiry": "enforce | warn"
            },
            "revocationValidations": {
                "signingIdentityRevocation": "enforceWithFailOpen | enforceWithFailClose | warn | skip",
                "timestampRevocation": "enforceWithFailOpen | enforceWithFailClose | warn | skip"
            }
        },
        {
            "name": "skip-signature-verification",
            "scopes": [ "wabbit-networks.io/software/unsigned/productA" ],
            "skipSignatureVerification": true,
        },
        {
            "name": "global-trust-policy",
            "scopes": [ "*" ],
            "trustStores": [ "trust-store-name-1", "trust-store-name-2" ],
            "expiryValidations": {
                "signatureExpiry": "enforce | warn",
                "signingIdentityExpiry": "enforce | warn",
                "timestampExpiry": "enforce | warn"
            },
            "revocationValidations": {
                "signingIdentityRevocation": "enforceWithFailOpen | enforceWithFailClose | warn | skip",
                "timestampRevocation": "enforceWithFailOpen | enforceWithFailClose | warn | skip"
            }
        }

    ]
}
```

Property descriptions

- **`version`**(*string*): This REQUIRED property is the version of the trust policy. The supported value is `1.0`.
- **`trustPolicies`**(*string-array of objects map*): This REQUIRED property represents a collection of trust policies.
  - **`name`**(*string*): This REQUIRED propert represents name of the trust policy.
  - **`scopes`**(*array of strings*): This REQUIRED property determines which trust policy is applicable for the given artifact. The scope field supports filtering based on fully qualified repository URI `${registry-name}/${namespace}/${repository-name}`. For more information, see [scopes constraints](#scope-constraints) section.
  - **`skipSignatureVerification`**(*boolean*): This OPTIONAL property dictates whether Notary v2 should skip signature verification or not. If set to `true` Notary v2 MUST NOT perform any signature validations including the custom validations performed using plugins. This is required to support the gradual rollout of signature validation i.e the case when the user application has a mix of signed and unsigned artifacts. When set to `false`, the following properties  MUST be present `trustStores`, `expiryValidations`, `revocationValidations`. The default value is `false`.
  - **`trustStores`**(*array of strings*): This OPTIONAL property specifies a list of names of trust stores that the user trusts.
  - **`expiryValidations`**(*object*): This OPTIONAL property represents a collection of artifact expiry-related validations.
    - **`signatureExpiry`**(*string*): This REQUIRED property specifies what implementation must do if the signature is expired.  Supported values are `enforce` and `warn`.
    - **`signingIdentityExpiry`**(*string*): This REQUIRED property specifies what implementation must do if signing identity(certificate and certificate-chain) is expired. Supported values are `enforce` and `warn`.
    - **`timestampExpiry`**(*string*): This REQUIRED property specifies what implementation must do if timestamping certificate and certificate-chain are expired. Supported values are `enforce` and `warn`.
  - **`revocationValidations`**(*object*): This OPTIONAL property represents collection of artifact revocation related validations.
    - **`signingIdentityRevocation`**(*string*): This REQUIRED property specifies whether implementation should check for signing identity(certificate and certificate-chain) revocation status or not and what implementation must do if this revocation check fails. Supported values are `enforceWithFailOpen`, `enforceWithFailClose`, `warn` and `skip`.
    - **`timestampRevocation`**(*string*): This REQUIRED property specifies whether implementation should check for timestamping certificate and certificate-chain revocation status or not and what implementation must do if this revocation check fails. Supported values are `enforceWithFailOpen`, `enforceWithFailClose`, `warn` and `skip`.

Value descriptions

- **`enforce`**: This means implementation MUST perform validation and throw an error if validation fails.
- **`enforceWithFailOpen`**: This means implementation MUST perform validation and if validation fails because the endpoint is not reachable, the implementation MUST throw an error and MUST fail the validation.
- **`enforceWithFailClose`**: This means implementation MUST perform validation and if validation fails because the endpoint is not reachable, the implementation MUST log an error and MUST NOT fail the validation.
- **`warn`**: This means implementation MUST perform the validation and if validation fails(because of any reason) the implementation MUST log an error and MUST NOT fail validation.
- **`skip`**: This means implementation MUST NOT perform the validation.

#### Scopes Constraints

- Each trust policy MUST contain scope property and the scope collection MUST contain at least one value.
- The scope MUST contain one of the following:
  - List of one or more fully qualified repository URIs. The repository URI MUST NOT contain the asterisk character `*`.
  - A single value with one asterisk character `*`. The scope with `*` value is called global scope. The trust policy with global scope applies to all the artifacts. There can only be one trust policy that uses a global scope.
- For a given artifact there MUST be only one applicable trust policy, with the exception of trust policy with global scope.
- For a given artifact, if there is no applicable trust policy then Notary v2 MUST consider the artifact as untrusted and fail signature verification.
- The scope MUST NOT support reference expansion i.e. URIs must be fully qualified. E.g. the scope should be `docker.io/library/registry` rather than `registry`.
- Evaluation order of trust policies:
  1. *Exact match*: If there exists a trust policy whose scope contains the artifact's repository URI then the aforementioned policy MUST be used for signature evaluation. Otherwise, continue to the next step.
  1. *Gobal*: If there exists a trust policy with global scope then use that policy for signature evaluation. Otherwise, fail the signature verification.

### Extended Validation

The implementation must allow the user to execute custom validations. These custom validation MUST have access to all the information available in the signature envelope like payload, signed attributes, unsigned attributes, and signature.

## Signature Evaluation

### Prerequisites

- User has configured [trust store](#trust-store) and [trust policy](#trust-policy).

### Steps

1. **Validate that the signature envelope format is supported.**
    1. Parse the signature envelope content based on the signature envelope type specified in the `[descriptors].descriptor.mediaType` attribute of the signature artifact manifest.
    1. Validate that the content type indicated by the `cty` property value of protected headers in the signature envelope is supported.
1. **Validate the signature envelope integrity.**
    1. Get the signing certificate from the parsed [signature envelope](https://github.com/notaryproject/notaryproject/blob/7b7d283038/signature-specification.md#signature-envelope).
    1. Get the signing algorithm(hash+encryption) from the signing certificate and validate that the signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements)
    1. Using the public key of the signing certificate and signing algorithm identified in the previous step, validate the integrity of the signature envelope.
1. **Validate the signature against trust policy and trust store.**
    1. Using the `scope` configured in trust policies, get the applicable trust policy. (Implementations might have this value precomputed, added it for completeness)
    1. For the applicable trust policy, **validate trust-store:**
        1. Validate that signature envelope contains complete certificate chain that start from a code signing certificate and terminate with the root certificate. Also, validate that code signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
        1. For each the trust-stores configured in applicable trust-policy perform the following steps.
            1. Validate that certificate and certificate-chain lead to a trusted certificate configured in the `x509Certs` field of trust-store.
            1. If the above verification succeeds then continue to the next step else iterate over the next trust store. If all of the trust stores have been evaluated then fail the signature validation and exit.
    1. **Validate trust policy:**
        1. If signature expiry is present in the signature envelope, using the local machine’s current time(in UTC) check whether the signature is expired or not. If the signature is not expired, continue to the next step. Otherwise, if `signatureExpiry` is set to `Enforce` then fail the signature validation and exit else log a warning and continue to the next step.
        1. Check for the timestamp signature in the signature envelope.
            1. If the timestamp exists, continue with the next step. Otherwise, store the local machine's current time(in  UTC) in variables `timeStampLowerLimit` and `timeStampUpperLimit` and continue with step 3.3.c.
            1. Validate that the timestamp hash in `TSTInfo.messageImprint` matches the hash of the signature to which the timestamp was applied.
            1. Validate that the timestamp signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
            1. Validate that the timestamp signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
            1. Validate the `signing-certificate`([RFC-2634](https://tools.ietf.org/html/rfc2634)) or `signing-certificate-v2`([RFC-5126](https://tools.ietf.org/html/rfc5126#section-5.7.3.2)) attribute of timestamp CMS.
            1. Check whether timestamping certificate and certificate chain are valid (not expired) or not. If timestamping certificate and certificate-chain are not expired, continue to the next step. Otherwise, if `timestampExpiry` in trust-policy is configured to `Enforce` then fail the signature validation and exit, else log a warning and continue to the next step.
            1. Validate that timestamp certificate and certificate chain leads to a trusted TSA certificate configured in trust policy.
            1. Validate timestamp certificate and certificate chain revocation status  using [certificate revocation evaluation](#certificate-revocation-evaluation) section as per `timestampRevocation` setting in trust-policy
            1. Retrieve the timestamp's time from `TSTInfo.genTime`.
            1. Retrieve the timestamp's accuracy.  If the accuracy is explicitly specified in `TSTInfo.accuracy`, use that value.  If the accuracy is not explicitly specified and `TSTInfo.policy` is the baseline time-stamp policy([RFC-3628](https://tools.ietf.org/html/rfc3628#section-5.2)), use accuracy of 1 second.  Otherwise, use accuracy of 0.
            1. Calculate the timestamp range using the lower and upper limits per [RFC-3161 section 2.4.2](https://tools.ietf.org/html/rfc3161#section-2.4.2) and store the limits as `timeStampLowerLimit` and `timeStampUpperLimit` variables respectively.
        1. Check that the time range from `timeStampLowerLimit` to `timeStampUpperLimit` timestamp is entirely within the certificate's validity period.  If the time range is entirely within the signing certificate and certificate chain's validity period, continue to the next step.  Otherwise, If `signingIdentityExpiry` in trust-policy is configured to `Enforce` then fail the signature validation and exit else log a warning and continue to the next step.
        1. Validate signing identity(certificate and certificate chain) revocation status using [certificate revocation evaluation](#certificate-revocation-evaluation) section as per `signingIdentityRevocation` setting in trust-policy.
        1. Perform extended validation using the applicable(if any) plugin.
        1. If you have reached this step then treat the OCI artifact signature as a valid signature.

### Certificate Revocation Evaluation

If the certificate revocation trust-store setting is set to `skip`, skip the below steps. Otherwise, check for revocation status for certificate and certificate chain.

1. If the revocation status of any of the certificates cannot be determined (revocation unavailable) and `signingIdentityRevocation` is set to either `enforceWithFailClose` or `warn` then log a warning and skip the below steps. Otherwise, fail the signature validation and exit.
1. If any of the certificates are revoked and `signingIdentityRevocation` is set to either `enforceWithFailOpen` or `enforceWithFailClose` then fail signature validation and exit else log a warning.

Starting from Root to leaf certificate, for each certificate in the certificate chain, perform the following steps to check its revocation status:

- If the certificate being validated doesn't include information OCSP or CRLs then no revocation check is performed and the certificate is considered valid(not revoked).
- If the certificate being validated includes either OCSP or CRL information, then the one which is present is used for revocation check.
- If both OCSP URLs and CDP URLs are present, then OCSP is preferred over CRLs. If revocation status cannot be determined using OCSP because of any reason such as unavailability then fallback to using CRLs for revocation check.

#### CRLs

There are two types of CRLs (per RFC 3280), Base CRLs and Delta CRls.

- **Base CRLs:** Contains the revocation status of all certificates that have been issued by a given CA. BaseCRLs are signed by the certificate issuing CAs.
- **Delta CRLs:** Contains only certificates that have changed status since the last base CRL was published. Delta CRLs are signed by the certificate issuing CAs.
- **Indirect CRLs:** Special Case in which BaseCRLs and Delta CRLs are not signed by the certificate issuing CAs, instead they are signed with a completely different certificate.

Notary v2 MUST support BaseCRLs and Delta CRLs. Notary v2 MAY support Indirect CRLs. Notary v2 supports only HTTP CRL URLs.

Implementations MAY add support for caching CRLs and OCSP response to improve availability, latency and avoid network overhead.

##### CRL Download

CRL download location (URL) can be obtained from the certificate's CRL Distribution Point (CDP) extension. If the certificate contains multiple CDP locations then each location download is attempted in sequential order. For each CDP location, Notary V2 will try to download the CRL for the default threshold of 10 seconds. If the CRL cannot be downloaded within the timeout threshold the revocation result will be "revocation unavailable". The user may be able to configure this threshold.

##### Revocation Checking with CRL

To check the revocation status of a certificate against CRL, the following steps must be performed:

1. Verify the CRL signature.
1. Verify that the CRL is valid (not expired). A CRL is considered expired if the current date is after the `NextUpdate` field in the CRL.
1. Look up the certificate’s serial number in the CRL.
    1. If the certificate’s serial number is listed in the CRL, look for `InvalidityDate`. If the invalidity date is present and timestamp signature is also present then if the invalidity date is before the timestamping date, the certificate is considered revoked. If the invalidity date is not present in CRL, the certificate is considered revoked.
    1. If the CRL is expired and the certificate is listed in the CRL for any reason other than `certificate hold`, the certificate is considered revoked.
    1. If the certificate is not listed in the CRL or the revocation reason is `certificate hold`, a new CRL is retrieved if the current time is past the time in the `NextUpdate` field in the current CRL. The new CRL is then checked to determine if the certificate is revoked. If the original reason was `certificate hold`, the CRL is checked to determine if the certificate is unrevoked by looking for the `RemoveFromCRL` revocation code.

##### Revocation Checking with Delta CRLs

If a delta CRL exists for a base CRL, the `Freshest CRL` extension in the base CRL provides the URL from where the delta CRL can be downloaded. The delta CRLs have their identifying number, plus the "Delta CRL Indicator" extension indicating the version number of the base CRL that must be used with the delta CRL.

When delta CRLs are implemented, the following results can occur during revocation checking.

- If the delta CRL cannot be retrieved for some reason, the revocation result will be "revocation unavailable".
- If the delta CRL’s indicator is less than the current base CRL, the revocation result returned will be "revocation unavailable"
- To check revocation of a certificate in Delta CRLs follow steps similar to [Revocation Checking with CRL](#revocation-checking-with-crl).

#### OCSP

##### OCSP Download

OCSP URLs can be obtained from the certificate's authority information access (AIA) extension as defined in [RFC-2560](https://datatracker.ietf.org/doc/html/rfc2560). Notary V2 will wait for a default threshold of 5 seconds to receive an OCSP response. If OCSP response is not available within the timeout threshold the revocation result will be "revocation unavailable". The user may be able to configure this threshold.

##### Revocation Checking with OCSP

To check the revocation status of a certificate using OCSP, the following steps must be performed.

1. Verify the signature of the OCSP response. This step also includes verifying the OSCP response signing certificate is valid and trusted. The OCSP signing certificate must be issued by the same CA as the certificate being verified or the OCSP response must be signed by the issuing CA.
1. Verify that the OCSP response is valid (not expired). A CRL is considered expired if the current date is after the `NextUpdate` field in the CRL.
1. Verify that the OCSP response indicates that the certificate is not revoked i.e `CertStatus` is `good`.
    1. If the certificate is revoked i.e `CertStatus` is `revoked`, look for `InvalidityDate`. If the invalidity date is present and timestamp signature is also present then if the invalidity date is before the timestamping date, the certificate is considered revoked. If the invalidity date is not present in OCSP response, the certificate is considered revoked.
1. If `id-pkix-ocsp-nocheck`(1.3.6.1.5.5.7.48.1.5) extension is not present on the OCSP signing certificate then revocation checking must be performed using CRLs for the OCSP signing certificate.

## FAQ

**Q: How should multiple signatures requirements be represented in the trust policy?**

**A:** Notary v2 doesn't support n out m signature requirement verification scheme. Validation succeeds if verification succeeds for at least one signature.

**Q: Should local revocation and TSA servers be listed in the trust policy to support disconnected environments?**

**A:** Not natively supported but a user can configure `revocationValidations` to `skip` and then use extended validations to check for revocation.

**Q: Why do we need to include a complete certificate chain (leading to root) in the signature?**

**A:** Without a complete certificate chain, the implementation won't be able to perform an exhaustive revocation check, which will lead to security issues, and that's the reason for enforcing a complete certificate chain.

**Q: Why are we validating artifact signature first instead of signing identity?**

**A:** Ideally, we should validate the signing identity first and then use the public key in the signing identity to validate the artifact signature. However, this will lead to poor performance in the case where the signature is not valid as there are lots of validations against the signing identity including network calls for revocations, and possibly we won't even need to read the trust store/trust policy if the signature validation fails.
Also, by validating artifact signature first we will still fail the validation if the signing identity is not trusted.
