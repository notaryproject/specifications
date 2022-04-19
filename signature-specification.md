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

- **`artifactType`** (*string*): This REQUIRED property references the Notary version of the signature: `application/vnd.cncf.notary.v2.signature`.
- **`blobs`** (*array of objects*): This REQUIRED property contains collection of only one [artifact descriptor][artifact-descriptor] referencing signature envelope.
  - **`mediaType`** (*string*): This REQUIRED property contains media type of signature envelope blob. The currently supported values are  `application/cose` and  `application/jose+json`.
- **`subject`** (*descriptor*): A REQUIRED artifact descriptor referencing the signed manifest, including, but not limited to image manifest, image index, oras-artifact manifest.
- **`annotations`** (*string-string map*): This REQUIRED property contains metadata for the artifact manifest.
  It is being used to store information about the signature.
  Keys using the `io.cncf.notary` namespace are reserved for use in Notary and MUST NOT be used by other specifications.
  - **`io.cncf.notary.x509.fingerprint.sha256`**: A REQUIRED annotation whose value contains the list of SHA-256 fingerprint of signing certificate and certificate chain used for signature generation.
    The list of fingerprints is present as a JSON array string.

### Signature Discovery

The client should be able to discover all the signatures belonging to an artifact (such as image manifest) by using [ORAS Manifest Referrers API][oras-artifacts-referrers].
ORAS Manifest Referrers API returns a paginated list of all artifacts belonging to a target artifact (such as container images, SBoMs).
The implementation can filter Notary signature artifacts by either using ORAS Manifest Referrers API or using custom logic on the client.
Each Notary signature artifact refers to a signature envelope blob.

### Signature Filtering

An OCI artifact can have multiple signatures, Notary v2 uses annotations of the signature artifact to filter relevant signatures based on the applicable trust policy.
The Notary v2 signature artifact's `io.cncf.notary.x509.fingerprint.sha256` annotations key MUST contain the list of SHA-256 fingerprints of certificate and certificate chain used for signing.

## Signature Envelope

The Signature Envelope is a standard data structure for creating a signed message.
A signature envelope consists of the following components:

- Payload/Message `m`: The data that is integrity protected - e.g. descriptor of the artifact being signed.
- Signed attributes `v`: The signature metadata that is integrity protected - e.g. signature expiration time, creation time, etc.
- Unsigned attributes `u`: Unsigned attributes `u`: These attributes are not signed by the signing key that generates the signature. We anticipate unsigned attributes contain content that may be signed by an different party e.g. Certificate chain signed by a CA, or TSA countersignature signed by the TSA.
- Cryptographic signatures `s`: The digital signatures computed on payload and signed attributes.

A signature envelope is `e = {m, v, u, s}` where `s` is signature.

This specification defines the set of signed and unsigned attributes that make up a valid The Notary v2 signature. This specification aims to be be agnostic of signature envelope format (e.g. COSE, JWS), details of encoding the envelope in a specific signature envelope format are covered in in separate specs.

Notary v2 supports the following envelope formats:

- [COSE Sign1](./signature-envelope-cose.md)
- [JWS](./signature-envelope-jws.md)

### Payload

Notary v2 requires Payload to be the content **descriptor** of the subject manifest that is being signed.

1. Descriptor MUST contain `mediaType`, `digest`, `size` fields.
2. Descriptor MAY contain `annotations` and if present it MUST follow the [annotation rules][annotation-rules]. Notary v2 uses annotations for storing both Notary specific and user defined signed attributes. The prefix `io.cncf.notary` in annotation keys is reserved for use in Notary v2 and MUST NOT be used outside this specification.
3. Descriptor MAY contain `artifactType` field for artifact manifests, or the `config.mediaType` for `oci.image` based manifests.

**Examples**: The media type of the content descriptor is `application/vnd.cncf.oras.artifact.descriptor.v1+json`.

```jsonc
{
   "mediaType": "application/vnd.oci.image.manifest.v1+json",
   "digest": "sha256:73c803930ea3ba1e54bc25c2bdc53edd0284c62ed651fe7b00369da519a3c333",
   "size": 16724,
   "annotations": {
      "io.wabbit-networks.buildId": "123"  // user defined metadata
   }
}
```

```jsonc
{
   "mediaType": "sbom/example",
   "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0",
   "size": 32654
}
```

### Signature Scheme

Notary will initially support X509 PKI based identities, which has well established standards for establishing and trusting signing identities, managing key lifetimes and rotation, and revocation. There is interest in the community to support additional trust models based on other standards and techniques (Notary TUF, ledger based). Notary addresses future support for additional trust models through an abstraction called Signature Scheme. A Signature Scheme defines the the specific trust model used for generation and verification of signatures. It includes the supported identity type (e.g. X509 certificates), set of signed and unsigned attributes in the signature to support the trust model, which attributes are required/optional and critical/informational, the signature verification logic, and verification policy switches that are supported (e.g. revocation checks can be skipped, audited, enforced) and how they are processed during signature verification. Notary v2 defines the following signature schemes.

`notary.signer.x509` - This signature scheme defines the trust model that uses traditional X509 based PKI. Users owns signing keys and use CA issued certificates to represent identity.

`notary.signingservice.x509` - This signature scheme defines the trust model for a signing service that uses X509 based PKI. In this model, a trusted signing service manages the keys of behalf of the user, and generates signatures. The end user does not have direct access to signing keys.

When Notary supports additional signature schemes, Notary verification policy may have breaking changes to support newer concepts introduced by the signature scheme, and existing signed artifacts will need to be resigned if they are required to be validated using the new trust system introduced by the signature scheme.

### Signed Attributes

Signed attributes/claims are additional metadata apart from the payload, which are required to support the signature verification process.

- Any claims MUST NOT be stored/appended in the payload itself, as the payload is only parsed and processed once the signature has been verified (signature is valid, and from a trusted key) and trust is established.
- Specific claims can be either required or optional.
- Claims that if present, MUST be processed by a verifier MUST be marked as critical.
- Claims which are informational and do not influence signature verification MUST NOT be marked critical.

Notary v2 requires the signature envelope to support the following signed attributes/claims.

#### Standard attributes

- **Notary Signature scheme** (critical): The trust model used by Notary currently supports the following values for this claim `notary.signer.x509` and `notary.signingservice.x509`. Both signature schemes use the same set of claims defined in this document. One difference is that `notary.signer.x509` uses a TSA signature to determine authenticated signing time (timestamp), and `notary.signingservice.x509` uses the trusted signing time claim instead, with optional support for TSA signatures. Signature verification MUST consider the signature to be invalid if any other value is used.
- **Signing time**: The time at which the signature was generated. This is a REQUIRED claim for signature scheme `notary.signer.x509`. Though this claim is signed by the signing key, it’s considered unauthenticated as a signer can modify local time and manipulate this claim. More details [here](#signing-time).
- **Trusted signing time** (critical) : The time at which the signature was generated. This is an authenticated signing time which can be generated by a trusted time-stamper, like a signing service. This is a REQUIRED claim for signature scheme `notary.signingservice.x509` . More details [here](#signing-time).
- **Signature Expiry** (critical): The time when a signature is considered to be expired. This is an OPTIONAL claim. [Open issue](https://github.com/notaryproject/notaryproject/issues/141) tracking if this attribute should be removed.
- **Content type** (critical): The content type of the payload. Notary currently supports OCI descriptor of a subject manifest as the payload, supported value is `application/vnd.cncf.oras.artifact.descriptor.v1+json`, other payload types MAY be supported in future. This is a REQUIRED claim.
- **Client identifier**: The version of a client (e.g. Notation) that produced the signature. This is an OPTIONAL claim. It uses the following format `{client}/{version}` e.g. “notation/1.0.0”. This claim in intended to be used for diagnostic and troubleshooting purposes.

### Unsigned Attributes

These attributes are considered unsigned with respect to the signing key that generates the signature. These attributes are typically signed by a third party (e.g. CA, TSA).

- **Certificate Chain**: This property contains the list of X.509 certificate or certificate chain. This is a REQUIRED attribute. The certificate chain is authenticated using against a trust store as part of signature validation.
- **TSA counter signature** : The time stamp token generated for a given signature. Only [RFC3161](ietf-rfc3161) compliant TimeStampToken are supported. This is an OPTIONAL attribute.

### Algorithm Selection

The signing certificate's public key algorithm and size MUST be used to determine the signature algorithm.

| Public Key Algorithm | Key Size (bits) | Signature Algorithm             |
| -------------------- | --------------- | ------------------------------- |
| RSA                  | 2048            | RSASSA-PSS with SHA-256         |
| RSA                  | 3072            | RSASSA-PSS with SHA-384         |
| RSA                  | 4096            | RSASSA-PSS with SHA-512         |
| EC                   | 256             | ECDSA on secp256r1 with SHA-256 |
| EC                   | 384             | ECDSA on secp384r1 with SHA-384 |
| EC                   | 512             | ECDSA on secp521r1 with SHA-512 |

### Certificate Requirements

The **signing certificate** MUST meet the following minimum requirements:

- The keyUsage extension MUST be present and MUST be marked critical.
  The bit positions for digitalSignature MUST be set ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.3)).
- The extKeyUsage extension MUST be present and its value MUST be id-kp-codeSigning ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.12)).
- If the basicConstraints extension is present, the cA field MUST be set false ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.9)).
- The certificate MUST abide by the following key length restrictions:
- For RSA public key, the key length MUST be 2048 bits or higher.
  - For ECDSA public key, the key length MUST be 256 bits or higher.

The **timestamping certificate** MUST meet the following minimum requirements:

- The keyUsage extension MUST be present and MUST be marked critical.
  The bit positions for digitalSignature MUST be set ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.3)).
- The extKeyUsage extension MUST be present and MUST be marked critical.
  The value of extension MUST be id-kp-timeStamping ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.12)).
- If the basicConstraints extension is present, the cA field MUST be set false ([RFC-5280](https://datatracker.ietf.org/doc/html/rfc5280#section-4.2.1.9)).
- The certificate MUST abide by the following key length restrictions:
  - For RSA public key, the key length MUST be 2048 bits or higher.
  - For ECDSA public key, the key length MUST be 256 bits or higher.

## FAQ

**Q: How will Notary v2 support multiple signature envelope formats?**

**A:** The `mediaType` of artifact manifest's blob identifies the signature envelope type.  
The client implementation can use the aforementioned `mediaType` to parse the signature envelope.

**Q: How will Notary v2 handle non-backward compatible changes to signature format?**

**A:** The Signature envelope MUST have a versioning mechanism to support non-backward compatible changes.  
[COSE_Sign1_Tagged](./signature-envelope-cose.md) signature envelope versioning is achieved by the `cty` field in ProtectedHeaders.  
[JWS JSON serialization](./signature-envelope-jwt.md) signature envelope versioning is achieved by the `cty` field in ProtectedHeaders.

## Appendix

### Signing time

The signing time denotes the time at which the signature was generated. A X509 certificate has a defined [validity](https://datatracker.ietf.org/doc/html/rfc5280#section-4.1.2.5) during which it can be used to generate signatures. The signing time must be greater than or equal to certificate's `notBefore` attribute, and signing time must be less than or equal to certificate's `notAfter` attribute. Signatures generated after the certificate expires are considered invalid. A trusted timestamp allows a verifier to determine if the signature was generated when the certificate was valid. It also allows a verifier to determine if a signature be treated as valid when a certificate is revoked, if the certificate was revoked after the signature was generated. In the absence of a trusted timestamp, signatures are considered invalid after certificate expires, and all signatures are considered revoked when a certificate is revoked.

[annotation-rules]: https://github.com/opencontainers/image-spec/blob/main/annotations.md#rules
[artifact-descriptor]: https://github.com/oras-project/artifacts-spec/blob/main/descriptor.md
[ietf-cose]: https://datatracker.ietf.org/doc/html/rfc8152
[ietf-cose-section-4]: https://datatracker.ietf.org/doc/html/rfc8152#section-4
[ietf-cose-section-4.2]: https://datatracker.ietf.org/doc/html/rfc8152#section-4.2
[ietf-cose-appendix-c]: https://datatracker.ietf.org/doc/html/rfc8152#appendix-C
[ietf-rfc3161]: https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2
[oras-artifact-manifest]: https://github.com/oras-project/artifacts-spec/blob/main/artifact-manifest.md
[oras-artifacts-referrers]: https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md
