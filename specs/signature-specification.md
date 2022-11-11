# Signature Specification

This document provides the following details for Notary v2 signatures:

- **[Storage](#storage)**: Describes how signatures are stored and retrieved from an OCI registry.
- **[Signature Envelope](#signature-envelope)**: Describes the structure of a Notary v2 signature.

## Storage

This section describes how Notary v2 signatures are stored in the OCI Distribution conformant registry.
Notary v2 uses [ORAS artifact manifest][oras-artifact-manifest] to store the signature in the repository.
The media type of the signature manifest is `application/vnd.cncf.oras.artifact.manifest.v1+json`.
The signature manifest has an artifact type that specifies it's a Notary V2 signature, a reference to the manifest of the artifact being signed, a blob referencing the signature, and a collection of annotations.

![Signature storage inside registry](media/signature-specification.svg)

Signature Manifest Example

```jsonc
{
    "mediaType": "application/vnd.cncf.oras.artifact.manifest.v1+json",
    "artifactType": "application/vnd.cncf.notary.signature",
    "blobs": [
        {
            "mediaType": "application/jose+json",
            "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0",
            "size": 32654
        }
    ],
    "subject": {
        "mediaType": "application/vnd.oci.image.manifest.v1+json",
        "digest": "sha256:73c803930ea3ba1e54bc25c2bdc53edd0284c62ed651fe7b00369da519a3c333",
        "size": 16724
    },
    "annotations": {
        "io.cncf.notary.x509chain.thumbprint#S256": 
        "[\"B7A69A70992AE4F9FF103EBE04A2C3BA6C777E439253CE36562E6E98375068C3\",\"932EB6F5598435D4EF23F97B0B5ACB515FAE2B8D8FAC046AB813DDC419DD5E89\"]"
    }
}
```

- **`artifactType`** (*string*): This REQUIRED property references the Notary version of the signature: `application/vnd.cncf.notary.signature`.
- **`blobs`** (*array of objects*): This REQUIRED property contains collection of only one [artifact descriptor][artifact-descriptor] referencing signature envelope.
  - **`mediaType`** (*string*): This REQUIRED property contains media type of signature envelope blob. Following values are supported
    - `application/jose+json`
    - `application/cose`
- **`subject`** (*descriptor*): A REQUIRED artifact descriptor referencing the signed manifest, including, but not limited to image manifest, image index, oras-artifact manifest.
- **`annotations`** (*string-string map*): This REQUIRED property contains metadata for the artifact manifest.
  It is being used to store information about the signature.
  Keys using the `io.cncf.notary` namespace are reserved for use in Notary and MUST NOT be used by other specifications.
  - **`io.cncf.notary.x509chain.thumbprint#S256`**: A REQUIRED annotation whose value contains the list of SHA-256 fingerprint of signing certificate and certificate chain (including root) used for signature generation. The annotation name contains the hash algorithm as a suffix (`#S256`) and can be extended to support other hashing algorithms in future.
    The list of fingerprints is present as a JSON array string, corresponding to ordered certificates in [*Certificate Chain* unsigned attribute](#unsigned-attributes) in the signature envelope.

### Signature Discovery

The client should be able to discover all the signatures belonging to an artifact (such as image manifest) by using [ORAS Manifest Referrers API][oras-artifacts-referrers].
ORAS Manifest Referrers API returns a paginated list of all artifacts belonging to a target artifact (such as container images, SBoMs).
The implementation can filter Notary signature artifacts by either using ORAS Manifest Referrers API or using custom logic on the client.
Each Notary signature artifact refers to a signature envelope blob.

### Signature Filtering

An OCI artifact can have multiple signatures, Notary v2 uses annotations of the signature artifact to filter relevant signatures based on the applicable trust policy.
The Notary v2 signature artifact's `io.cncf.notary.x509chain.thumbprint#S256` annotations key MUST contain the list of SHA-256 fingerprints of certificate and certificate chain used for signing.

## Signature Envelope

The Signature Envelope is a standard data structure for creating a signed message.
A signature envelope consists of the following components:

- Payload/Message `m`: The data that is integrity protected - e.g. descriptor of the artifact being signed.
- Signed attributes `v`: The signature metadata that is integrity protected - e.g. signature expiration time, creation time, etc.
- Unsigned attributes `u`: These attributes are not signed by the signing key that generates the signature. We anticipate unsigned attributes contain content that may be signed by an different party e.g. Certificate chain signed by a CA, or TSA countersignature signed by the TSA.
- Cryptographic signatures `s`: The digital signatures computed on payload and signed attributes.

A signature envelope is `e = {m, v, u, s}` where `s` is signature.

This specification defines the set of signed and unsigned attributes that make up a valid The Notary v2 signature. This specification aims to be be agnostic of signature envelope format (e.g. COSE, JWS), details of encoding the envelope in a specific signature envelope format are covered in in separate specs.

Notary v2 supports the following envelope formats:

- [JWS](./signature-envelope-jws.md)
- [COSE Sign1](./signature-envelope-cose.md)

### Payload

Notary v2 payload is a JSON document with media type `application/vnd.cncf.notary.payload.v1+json` and has following properties.

- `targetArtifact` : Required property whose value is the descriptor of the target artifact manifest that is being signed. Both [OCI descriptor][oci-descriptor] and [ORAS artifact descriptors][artifact-descriptor] are supported.
  - Descriptor MUST contain `mediaType`, `digest`, `size` fields.
  - Descriptor MAY contain `annotations` and if present it MUST follow the [annotation rules][annotation-rules]. Notary v2 uses annotations for storing both Notary specific and user defined metadata. The prefix `io.cncf.notary` in annotation keys is reserved for use in Notary v2 and MUST NOT be used outside this specification.
  - Descriptor MAY contain `artifactType` field for artifact manifests, or the `config.mediaType` for `oci.image` based manifests.

#### Examples

```jsonc
{
  "targetArtifact": {
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "digest": "sha256:73c803930ea3ba1e54bc25c2bdc53edd0284c62ed651fe7b00369da519a3c333",
    "size": 16724,
    "annotations": {
        "io.wabbit-networks.buildId": "123"  // user defined metadata
    }
  }
}
```

```jsonc
{
  "targetArtifact": {
    "mediaType": "sbom/example",
    "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0",
    "size": 32654
  }
}
```

### Signed Attributes

Signed attributes/claims are additional metadata apart from the payload, which are required to support the signature verification process.

- Any metadata that is used to verify the payload itself, and establish trust MUST be stored separately from the payload itself, as signed attributes or claims. Such metadata MUST NOT be stored/appended in the payload, as the payload is only parsed and processed once the signature has been verified and trust is established.
- Specific claims can be either required or optional.
- Claims that MUST be processed by a verifier MUST be marked as critical. Some claims may be optional and critical, i.e. they MUST be processed by a verifier only if they are present.
- Claims which are informational and do not influence signature verification MUST NOT be marked critical.

Notary v2 requires the signature envelope to support the following signed attributes/claims.

#### Standard attributes

- **Signing Scheme** (critical): A REQUIRED claim that defines the [Notary v2 Signing Scheme](./signing-scheme.md) used by the signature. This attribute dictates the rest of signature schema - the set of signed and unsigned attributes to be included in the signature. Supported values are `notary.x509` and `notary.x509.signingAuthority`.
- **Signing Time**: A claim that indicates the time at which the signature was generated. Though this claim is signed by the signing key, it’s considered unauthenticated as a signer can modify local time and manipulate this claim. More details [here](#signing-time). This claim is REQUIRED and only valid when signing scheme is `notary.x509` .
- **Authentic Signing Time** (critical): The authenticated time at which the signature was generated. This claim is REQUIRED and only valid when signing scheme is `notary.x509.signingAuthority` . More details [here](#signing-time).
- **Expiry** (critical): An OPTIONAL claim that provides a “best by use” time for the artifact, as defined by the signer. More details [here](#expiry).
- **Content Type** (critical): A REQUIRED claim that indicates the content type of the [payload](#payload). The supported value is `application/vnd.cncf.notary.payload.v1+json`. Other payload types MAY be supported in future.

#### Extended attributes

Implementations of Notary v2 signature spec MAY include additional signed attributes in the signature envelope.
These attributes MAY be marked critical, i.e. the attribute MUST be understood and processed by a verifier, unknown critical attributes MUST cause signature verification to fail.
Usage of extended signed attributes which are marked critical in signature will have implications on portability of the signature, these are discussed in [Signature Portability](#signature-portability) section.

#### Extended attributes for *Notation* Plugins

This section documents extended attributes used by Notary v2 reference implementation *Notation* to support plugins.
Plugins is a *Notation* concept that allows parts of signing and verification logic to be performed by an external provider.
*Signing plugins* allow *Notation* to be extended for integration with remote keys remote key management services and signing services, where as *verification plugins* allow for customization of verification logic.
Detailed specification for plugins can be found [here](https://github.com/notaryproject/notaryproject/blob/main/specs/plugin-extensibility.md#notation-extensibility-for-signing-and-verification).
These extended attributes are documented in this spec, as other Notary V2 implementations may encounter these attributes if they verify a signature that indicated it required a verification plugin for complete signature verification.

- **Verification Plugin** (critical): An OPTIONAL attribute that specifies the name of the verification plugin that MAY be used to verify the signature e.g. “com.example.nv2plugin”.
[Notation plugin](https://github.com/notaryproject/notaryproject/blob/main/specs/plugin-extensibility.md#plugin-contract) aware implementations use this attribute to load and execute a *Notation* compliant plugin.
The plugin participates in the overall signature verification workflow and performs specific steps in it.
- **Verification Plugin Minimum Version** (critical): An OPTIONAL attribute that specifies the minimum version of the verification plugin that MUST be used to verify the signature.
A Notation plugin aware implementations MUST use this attribute to verify the signature with a plugin with matching or higher plugin version.
The plugin MUST use [Semantic Versioning](https://semver.org/) (SemVer) to use this feature i.e the `get-plugin-metadata` plugin command MUST return a SemVer compliant version in the response.
A use case for this feature is for a plugin publisher to address security bug in older plugin version, by setting the minimum version to the plugin version with fixes.

See [Guidelines for Notary v2 Implementors](#guidelines-for-notary-v2-implementors) for options to handle these attributes during signature verification.


### Unsigned Attributes

These attributes are considered unsigned with respect to the signing key that generates the signature. These attributes are typically signed by a third party (e.g. CA, TSA).

- **Certificate Chain**: This is a REQUIRED attribute that contains the ordered list of X.509 public certificates associated with the signing key used to generate the signature. The ordered list starts with the signing certificates, any intermediate certificates and ends with the root certificate. The certificate chain MUST be authenticated against a trust store as part of signature validation. Specific requirements for the certificates in the chain are provided [here](#certificate-requirements).
- **Timestamp signature** : An OPTIONAL counter signature which provides [authentic timestamp](#signing-time)e.g. Time Stamp Authority (TSA) generated timestamp signature. Only [RFC3161](ietf-rfc3161) compliant TimeStampToken are currently supported.
- **Signing Agent**: An OPTIONAL claim that provides the identifier of the software (e.g. Notation) that produced the signature on behalf of the user. It is an opaque string set by the software that produces the signature. It's intended primarily for diagnostic and troubleshooting purposes, this attribute is unsigned, the verifier MUST NOT validate formatting, or fail validation based on the content of this claim. The suggested format is one or more tokens of the form `{id}/{version}` containing identifier and version of the software, seperated by spaces. E.g. “notation/1.0.0”, “notation/1.0.0 com.example.nv2plugin/0.8”.

### Supported Signature Envelopes

#### JWS JSON Serialization

In JWS JSON Serialization ([RFC7515](https://datatracker.ietf.org/doc/html/rfc7515)), data is stored as either claims or headers (protected and unprotected).
Notary v2 uses JWS JSON Serialization for the signature envelope with some additional constraints on the structure of claims and headers.

Unless explicitly specified as OPTIONAL, all fields are required.
Also, there shouldn’t be any additional fields other than ones specified in JWSPayload, ProtectedHeader, and UnprotectedHeader.

**JWSPayload a.k.a. Claims**:
Notary v2 is using one private claim (`notary`) and two public claims (`iat` and `exp`).
An example of the claim is described below

```json
{
   "subject": {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:73c803930ea3ba1e54bc25c2bdc53edd0284c62ed651fe7b00369da519a3c333",
      "size": 16724,
      "annotations": {
         "key1": "value1",
         "key2": "value2",
         ...
      }
   },
   "iat": 1234567891000,
   "exp": 1234567891011
}
```

The payload contains the subject manifest and other attributes that have to be integrity protected.

- **`subject`**(*descriptor*): A REQUIRED top-level node consisting of the manifest that needs to be integrity protected.
  Please refer [Payload](#payload) section for more details.
- **`iat`**(*number*): The REQUIRED property Issued-at(`iat`) identifies the time at which the signature was issued.
- **`exp`**(*number*): This OPTIONAL property contains the expiration time on or after which the signature must not be considered valid.

To leverage JWS claims validation functionality already provided by libraries, we have defined `iat`, `exp` as top-level nodes.

**ProtectedHeaders**: Notary v2 supports only three protected headers: `alg`, `cty`, and `crit`.

```json
{
    "alg": "RS256",
    "cty": "application/vnd.cncf.notary.v2.jws.v1",
    "crit":["cty"]
}
```

- **`alg`**(*string*): This REQUIRED property defines which algorithm was used to generate the signature.
  JWS needs an algorithm(`alg`) to be present in the header, so we have added it as a protected header.
- **`cty`**(*string*): The REQUIRED property content-type(cty) is used to declare the media type of the secured content(the payload).
  This will be used to version different variations of JWS signature.
  The supported value is `application/vnd.cncf.notary.v2.jws.v1`.
- **`crit`**(*array of strings*): This REQUIRED property lists the headers that implementation MUST understand and process.
  The value MUST be `["cty"]`.

**UnprotectedHeaders**: Notary v2 supports only two unprotected headers: timestamp and x5c.

```json
{
    "timestamp": "<Base64(TimeStampToken)>",
    "x5c": ["<Base64(DER(leafCert))>", "<Base64(DER(intermediateCACert))>", "<Base64(DER(rootCert))>"]
}
```

- **`timestamp`** (*string*): This OPTIONAL property is used to store time stamp token.
  Only [RFC3161]([rfc3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2)) compliant TimeStampToken are supported.
- **`x5c`** (*array of strings*): This REQUIRED property contains the list of X.509 certificate or certificate chain([RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)) corresponding to the key used to digitally sign the JWS.
  The certificate containing the public key corresponding to the key used to digitally sign the JWS MUST be the first certificate.

- **`timestamp`** (*string*): This OPTIONAL property is used to store time stamp token.
  Only [RFC3161]([rfc3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2)) compliant TimeStampToken are supported.
- **`x5c`** (*array of strings*): This REQUIRED property contains the list of X.509 certificate or certificate chain([RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)) corresponding to the key used to digitally sign the JWS.
  The certificate containing the public key corresponding to the key used to digitally sign the JWS MUST be the first certificate.

**Signature**: In JWS signature is calculated by combining JWSPayload and protected headers.
The process is described below:

1. Compute the Base64Url value of ProtectedHeaders.
1. Compute the Base64Url value of JWSPayload.
1. Build message to be signed by concatenating the values generated in step 1 and step 2 using '.'
`ASCII(BASE64URL(UTF8(ProtectedHeaders)) ‘.’ BASE64URL(JWSPayload))`
1. Compute the signature on the message constructed in the previous step by using the signature algorithm defined in the corresponding header element: `alg`.
1. Compute the Base64Url value of the signature produced in the previous step.
   This is the value of the signature property used in the signature envelope.

**Signature Envelope**: The final signature envelope comprises of Claims, ProtectedHeaders, UnprotectedHeaders, and signature.

Since Notary v2 restricts one signature per signature envelope, the compliant signature envelope MUST be in flattened JWS JSON format.

```json
{
    "payload": "<Base64Url(JWSPayload)>",
    "protected": "<Base64Url(ProtectedHeaders)>",
    "header": {
        "timestamp": "<Base64(TimeStampToken)>",
        "x5c": ["<Base64(DER(leafCert))>", "<Base64(DER(intermediateCACert))>", "<Base64(DER(rootCert))>"]
    },
    "signature": "Base64Url( sign( ASCII( <Base64Url(ProtectedHeader)>.<Base64Url(JWSPayload)> )))"  
}
```

**Implementation Constraints**: Notary v2 implementation MUST enforce the following constraints on signature generation and verification:

1. `alg` header value MUST NOT be `none` or any symmetric-key algorithm such as `HMAC`.
1. `alg` header value MUST be same as that of signature algorithm identified using signing certificate's public key algorithm and size.
1. `alg` header values for various signature algorithms:
  | Signature Algorithm             | `alg` Param Value |
  | ------------------------------- | ----------------- |
  | RSASSA-PSS with SHA-256         | PS256             |
  | RSASSA-PSS with SHA-384         | PS384             |
  | RSASSA-PSS with SHA-512         | PS512             |
  | ECDSA on secp256r1 with SHA-256 | ES256             |
  | ECDSA on secp384r1 with SHA-384 | ES384             |
  | ECDSA on secp521r1 with SHA-512 | ES512             |
1. Signing certificate MUST be a valid codesigning certificate.
1. Only JWS JSON flattened format is supported.
   See 'Signature Envelope' section.

## Signature Algorithm Requirements

The implementation MUST support the following set of algorithms:

1. RSASSA-PSS with SHA-256
1. RSASSA-PSS with SHA-384
1. RSASSA-PSS with SHA-512
1. ECDSA on secp256r1 with SHA-256
1. ECDSA on secp384r1 with SHA-384
1. ECDSA on secp521r1 with SHA-512

For ECDSA equivalent NIST curves and ANSI curves can be found at [RFC4492 Appendix A](https://tools.ietf.org/search/rfc4492#appendix-A).

### Algorithm Selection

The signing certificate's public key algorithm and size MUST be used to determine the signature algorithm.

| Public Key Algorithm | Key Size (bits) | Signature Algorithm             |
| -------------------- | --------------- | ------------------------------- |
| RSA                  | 2048            | RSASSA-PSS with SHA-256         |
| RSA                  | 3072            | RSASSA-PSS with SHA-384         |
| RSA                  | 4096            | RSASSA-PSS with SHA-512         |
| EC                   | 256             | ECDSA on secp256r1 with SHA-256 |
| EC                   | 384             | ECDSA on secp384r1 with SHA-384 |
| EC                   | 521             | ECDSA on secp521r1 with SHA-512 |

### Certificate Requirements

The **signing certificate** MUST meet the following minimum requirements:

- The keyUsage extension MUST be present and MUST be marked critical.
  The bit positions for digitalSignature MUST be set ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.3)).
- The extKeyUsage extension MUST be present and its value MUST be id-kp-codeSigning ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.12)).
- If the basicConstraints extension is present, the cA field MUST be set false ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.9)).
- The certificate MUST abide by the following key length restrictions:
- For RSA public key, the key length MUST be 2048 bits or higher.
  - For ECDSA public key, the key length MUST be 256 bits or higher.

The codesigning and timestamping certificates MUST meet the following requirements. These requirements are validated both at signature generation time and signature verification time, and are applied to the certificate chain in the signature envelope. These validations are independent of certificate chain validation against a trust store.

#### Root and Intermediate CA Certificates

The CA certificates MUST meet the following requirements

1. **[Basic Constraints:](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.9)**
The `basicConstraints` extension MUST be present and MUST be marked as critical. The `cA` field MUST be set `true`.
The [`pathLenConstraint`](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.9) field is OPTIONAL. If present, it MUST be verified against the depth of the chain below that CA certificate. (If value is null consider it as not present)
1. **[Key Usage:](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.3)**
The `keyUsage` extension MUST be present and MUST be marked critical. Bit positions for `keyCertSign` MUST be set.

#### Leaf Certificates

The leaf or end certificates MUST meet the following requirements

1. **[Basic Constraints:](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.9)**
The `basicConstraints` extension is OPTIONAL and can OPTIONALLY be marked as critical. If present, the `cA` field MUST be set to `false`.
1. **[Key Usage:](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.3)**
The `keyUsage` extension MUST be present and MUST be marked critical. Bit positions for `digitalSignature` MUST be set. The Bit positions for `keyEncipherment`, `dataEncipherment`, `keyAgreement`, `keyCertSign`, `encipherOnly`, `decipherOnly` and `cRLSign` MUST NOT be set.
1. **[Extended Key Usage:](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.12)**  The `extendedKeyUsage` extension is OPTIONAL and can OPTIONALLY be marked as critical.
    - **For signing certificate:** If present, the value MAY contain `id-kp-codeSigning` and MUST NOT contain `anyExtendedKeyUsage`, `serverAuth`, `clientAuth`, `emailProtection` and `timeStamping`.
    - **For timestamping certificate:** If present, the value MUST contain `id-kp-timeStamping` and MUST NOT contain `anyExtendedKeyUsage`, `serverAuth`, `clientAuth`, `emailProtection` and `codeSigning`.
1. **Key Length** The certificate MUST abide by the following key length restrictions:
    - For RSA public key, the key length MUST be 2048 bits or higher.
    - For ECDSA public key, the key length MUST be 256 bits or higher.

#### Other requirements

1. Valid certificate chain MUST contain a root certificate. If the certificate chain contains only a root certificate then the root certificate MUST meet [Leaf Certificates](#leaf-certificates) requirements and ignore [Root and Intermediate CA Certificates](#root-and-intermediate-ca-certificates) requirements.
1. The certificates in the signature MUST be ordered list of X.509 certificate or certificate chain i.e. the certificate containing the public key used to digitally sign the payload must be the first certificate, followed by the intermediate and root certificates in the correct order. This also means
    - The certificate MUST NOT chain to multiple parents/roots.
    - The certificate chain MUST NOT contain a certificate that is unrelated to the certificate chain.
1. Any certificate in the certificate chain MUST NOT use SHA1WithRSA and ECDSAWithSHA1 signatures.
1. Only Basic Constraints, Key Usage, and Extended Key Usage extensions of X.509 certificates are honored. For rest of the extensions, Notary MUST fail open i.e. they MUST NOT be evaluated or honored.
1. The certificates in the certificate chain MUST be valid at signing time. Notary MUST NOT enforce validity period nesting, i.e the validity period for a given certificate may not fall entirely within the validity period of that certificate's issuer certificate.
1. In the absence of an Authentic Timestamp, each and every certificate in the certificate chain i.e. signing certificate, intermediate certificates, and the root certificate must be valid i.e. not expired at the time of signature verification.

## FAQ

**Q: How will Notary v2 support multiple signature envelope formats?**

**A:** The `mediaType` of artifact manifest's blob identifies the signature envelope type.  
The client implementation can use the aforementioned `mediaType` to parse the signature envelope.

**Q: How will Notary v2 support multiple payload formats?**

**A:** The Signature envelope MUST have a versioning mechanism to support multiple payload formats.

- For [JWS JSON serialization](./signature-envelope-jwt.md) signature envelope, versioning is achieved by the `cty` field in ProtectedHeaders.
- For [COSE_Sign1_Tagged](./signature-envelope-cose.md) signature envelope, versioning is achieved by the `content type` (label: `3`) field in ProtectedHeaders.  

## Appendix

### Signing time

The signing time denotes the time at which the signature was generated. A X509 certificate has a defined [validity](https://datatracker.ietf.org/doc/html/rfc5280#section-4.1.2.5) during which it can be used to generate signatures. The signing time must be greater than or equal to certificate's `notBefore` attribute, and signing time must be less than or equal to certificate's `notAfter` attribute. Signatures generated after the certificate expires are considered invalid. An authentic timestamp, like TSA countersignature, allows a verifier to determine if the signature was generated when the certificate was valid. It also allows a verifier to determine if a signature be treated as valid when a certificate is revoked, if the certificate was revoked after the signature was generated. In the absence of an authentic timestamp, signatures are considered invalid after certificate expires, and all signatures are considered revoked when a certificate is revoked.

### Expiry

This is an optional feature that provides a “best by use” time for the artifact, as defined by the signer. Notary v2 allows users to include an optional expiry time when they generate a signature. The expiry time is not set by default and requires explicit configuration by users at the time of signature generation. The artifact is considered expired when the current time is greater than or equal to expiry time, users performing verification can either configure their trust policies to fail the verification or even accept the artifact with expiry date in the past using policy. This is an advanced feature that allows implementing controls for user defined semantics like deprecation for older artifacts, or block older artifacts in a production environment. Users should only include an expiry time in the signed artifact after considering the behavior they expect for consumers of the artifact after it expires. Users can choose to consume an artifact even after the expiry time based on their specific needs.

### Signature Portability

Portability of signatures is associated with the portability of associated artifacts which are being signed.
OCI artifacts are inherently location agnostic, artifacts can be pulled from and pushed to any OCI compliant registry to which a user has access.
The artifacts themselves can be classified as follow.

1. *Public Artifacts* -  Artifacts that are distributed publicly for broad consumption.
Artifacts distributed via public registries fall in this category.
E.g. Public images for software distributed by software vendors, and open source projects.
Signatures associated with these artifacts require broad portability.
1. *Private Artifacts* - Artifacts that are private to a user or organization, and may be shared with limited parties.
E.g. Images for containerized applications and services used within an organization, or shared with limited authorized parties.
Based on user requirements a private artifact can have different levels of portability, the signature’s portability should at least match the the artifact’s portability.

*Notary v2 signature portability* is based on the following

**Signature discovery**

Notary v2 addressed signature discovery by storing signatures in the same registry (location) where an artifact is present.
This is supported through [ORAS artifact spec](https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md) which allows reference artifacts such as signatures, SBOMs to be associated with existing artifacts like Images.
Notary v2 allows multiple signatures to be associated with an artifact, and clients may automatically push signatures for an artifact to a destination registry when a signed artifact moves from one registry to other.

**Verification requirements**

Notary v2 supports a range of signed artifacts intended for public and private distribution.
Signatures generated by Notary v2 without extended signature attributes marked critical can be verified in any environment where Notation client or another Notary v2 standards compliant verification tool is available, without any additional dependencies.
It should be noted that revocations checks, which usually relies on an external mechanism such as CRL/OCSP may require the verification environment to have access to local network or public internet, to have access to the CRL/OCSP endpoint.

Notary v2 also supports signatures generated using compliant signing plugins, which allow vendors to optionally provide additional features on top of Notary v2 standard features.
Verification of these signatures may require additional dependencies like Notary v2 compliant verification plugin, making these signatures more appropriate to use where broad portability may not be required for the associated signed artifact.
This allows users to implement security controls required for their organizations, that are not broadly applicable and may take time to standardize.
E.g. Integration with a signature transparency log as part of signature verification.

Based on user’s requirements, a user can select appropriate signing mechanism that produces signatures with desired portability.
Notation signatures without any critical extended attributes do not impose any additional dependency requirements for verifiers as these can be validated with just the Notation client.
Whereas, Notation signatures that contain critical extended attributes will require additional dependencies for signature validation, either on Notary v2 compliant plugins or equivalent tooling which may not be available in all environments.
Similarly, Notary v2 compliant plugin vendors should be aware that usage of extended signed attributes which are marked critical in signature will have implications on portability of the signature.

### Guidelines for Notary v2 Implementors

Implementations of Notary v2, can choose to be [Notation plugin protocol](./specs/plugin-extensibility.md#plugin-contract) aware or not. If an implementation chooses to be plugin protocol aware, and it encounters the Verification Plugin and Verification Plugin minimum version attributes during signature verification, it MUST process these attributes. This involves finding the appropriate plugin and the version to use, and executing `verify-signature` plugin command with correct inputs and processing the plugin response, as per the [Verification Plugin interface](../specs/../notaryproject-specs/specs/plugin-extensibility.md#verification-extensibility).

Alternatively, an implementation of Notary v2 can choose not to implement plugin protocol.

- The implementation MUST itself perform equivalent verification logic that is usually performed by plugin specified in the signature.
- An implementation MUST fail signature verification if it cannot perform the equivalent verification logic, as skipping the plugin equivalent verification logic will cause incorrect and inconsistent signature verification behavior.
- An implementation MAY choose to support a set of known plugin’s verification logic and fail others.

[annotation-rules]: https://github.com/opencontainers/image-spec/blob/main/annotations.md#rules
[oci-descriptor]: https://github.com/opencontainers/image-spec/blob/main/descriptor.md
[artifact-descriptor]: https://github.com/oras-project/artifacts-spec/blob/main/descriptor.md
[ietf-rfc3161]: https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2
[oras-artifact-manifest]: https://github.com/oras-project/artifacts-spec/blob/main/artifact-manifest.md
[oras-artifacts-referrers]: https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md
