# Signature Specification

This document describes how Notary v2 signatures are created and stored.
The document has the following sections:

- **[Storage](#storage)**: Describes how signatures are stored in OCI registry.
- **[Signature Envelope](#signature-envelope)**: Describes how signatures are created.

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
  - **`io.cncf.notary.x509certs.fingerprint.sha256`**: A REQUIRED annotation whose value contains the list of SHA-256 fingerprint of signing certificate and certificate chain used for signature generation.
    The list of fingerprints is present as a JSON array string.

### Signature Discovery

The client should be able to discover all the signatures belonging to an artifact (such as image manifest) by using [ORAS Manifest Referrers API][oras-artifacts-referrers].
ORAS Manifest Referrers API returns a paginated list of all artifacts belonging to a target artifact (such as container images, SBoMs).
The implementation can filter Notary signature artifacts by either using ORAS Manifest Referrers API or using custom logic on the client.
Each Notary signature artifact refers to a signature envelope blob.

### Signature Filtering

An OCI artifact can have multiple signatures, Notary v2 uses annotations of the signature artifact to filter relevant signatures based on the applicable trust policy.
The Notary v2 signature artifact's `io.cncf.notary.x509certs.fingerprint.sha256` annotations key MUST contain the list of SHA-256 fingerprints of certificate and certificate chain used for signing.

## Signature Envelope

The Signature Envelope is a standard data structure for creating a signed message.
A signature envelope consists of the following components:

- Payload `m`: The data that is integrity protected - e.g. descriptor of the artifact being signed.
- Signed attributes `v`: The signature metadata that is integrity protected - e.g. signature expiration time, creation time, etc.
- Unsigned attributes `u`: This OPTIONAL property represents signature metadata that is not integrity protected by this envelope- e.g. timestamp, certificates, etc.
  We anticipate all unsigned attributes contain signed content.
- Cryptographic signatures `s`: The digital signatures computed on payload and signed attributes.

A signature envelope is `e = {m, v, u, s}` where `s` is signature.

Notary v2 supports multiple envelope formats, including:

- [COSE Sign1](./signature-envelope-cose.md)
- [JWT](./signature-envelope-jwt.md)

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
      "io.wabbit-networks.buildId": "123"  // user defined signed attribute.
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

### Signed Attributes

Notary v2 requires the signature envelope to support the following signed attributes:

- **Creation time**: The time at which the signature was generated.
  Its value should be numeric representing the number of seconds (not milliseconds) since Epoch.
- **Expiration time**: The time after which signature shouldn't be considered valid.
  Its value should be numeric representing the number of seconds since Epoch.
  The value MUST be equal or greater than the `Creation time`
  This is an OPTIONAL attribute.

### Unsigned Attributes

- **Certificates**: The ordered collection of X.509 certificates with a signing certificate as the first entry.

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

[annotation-rules]: https://github.com/opencontainers/image-spec/blob/main/annotations.md#rules
[artifact-descriptor]: https://github.com/oras-project/artifacts-spec/blob/main/descriptor.md
[ietf-cose]: https://datatracker.ietf.org/doc/html/rfc8152
[ietf-cose-section-4]: https://datatracker.ietf.org/doc/html/rfc8152#section-4
[ietf-cose-section-4.2]: https://datatracker.ietf.org/doc/html/rfc8152#section-4.2
[ietf-cose-appendix-c]: https://datatracker.ietf.org/doc/html/rfc8152#appendix-C
[ietf-rfc3161]: https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2
[oras-artifact-manifest]: https://github.com/oras-project/artifacts-spec/blob/main/artifact-manifest.md
[oras-artifacts-referrers]: https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md
