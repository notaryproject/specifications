# Signature Specification

This document describes how Notary v2 signatures are created and stored.
The document has the following sections:

- **[Storage](#storage)**: Describes how signatures are stored in OCI registry.
- **[Signature Envelope](#signature-envelope)**: Describes how signatures are created.

## Storage

This section describes how Notary v2 signatures are stored in the OCI Distribution conformant registry.
Notary v2 uses [ORAS artifact manifest](https://github.com/oras-project/artifacts-spec/blob/main/artifact-manifest.md) to store the signature in the repository.
The media type of the signature manifest is `application/vnd.cncf.oras.artifact.manifest.v1+json`.
The signature manifest has an artifact type that specifies it's a Notary V2 signature, a reference to the manifest of the artifact being signed, a blob referencing the signature, and a collection of annotations.

![Signature storage inside registry](media/signature-specification.svg)

- **`artifactType`** (*string*): This REQUIRED property references the Notary version of the signature: `application/vnd.cncf.notary.v2.signature`.
- **`blobs`** (*array of objects*): This REQUIRED property contains collection of only one [artifact descriptor](https://github.com/oras-project/artifacts-spec/blob/main/descriptor.md) referencing signature envelope.
  - **`mediaType`** (*string*): This REQUIRED property contains media type of signature envelope blob. The supported value is `application/cose`.
- **`subject`** (*descriptor*): A REQUIRED artifact descriptor referencing the signed manifest, including, but not limited to image manifest, image index, oras-artifact manifest.
- **`annotations`** (*string-string map*): This REQUIRED property contains metadata for the artifact manifest.
  It is being used to store information about the signature.
  Keys using the `org.cncf.notary` namespace are reserved for use in Notary and MUST NOT be used by other specifications.
  - **`org.cncf.notary.x509certs.fingerprint.sha256`**: A REQUIRED annotation whose value contains the list of SHA-256 fingerprint of signing certificate and certificate chain used for signature generation.
    The list of fingerprints is present as a JSON array string.

```json
{
   "artifactType": "application/vnd.cncf.notary.v2.signature",
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
        "org.cncf.notary.x509certs.fingerprint.sha256": "[\"B7A69A70992AE4F9FF103EBE04A2C3BA6C777E439253CE36562E6E98375068C3\" \"932EB6F5598435D4EF23F97B0B5ACB515FAE2B8D8FAC046AB813DDC419DD5E89\"]"
    }
}
```

### Signature Discovery

The client should be able to discover all the signatures belonging to an artifact (such as image manifest) by using [ORAS Manifest Referrers API](https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md).
ORAS Manifest Referrers API returns a paginated list of all artifacts belonging to a target artifact (such as container images, SBoMs).
The implementation can filter Notary signature artifacts by either using ORAS Manifest Referrers API or using custom logic on the client.
Each Notary signature artifact refers to a signature envelope blob.

### Signature Filtering

An OCI artifact can have multiple signatures, Notary v2 uses annotations of the signature artifact to filter relevant signatures based on the applicable trust policy.
The Notary v2 signature artifact's `org.cncf.notary.x509certs.fingerprint.sha256` annotations key MUST contain the list of SHA-256 fingerprints of certificate and certificate chain used for signing.

## Signature Envelope

The Signature Envelope is a standard data structure for creating a signed message.
A signature envelope consists of the following components:

- Payload `m`: The data that is integrity protected - e.g. descriptor of the artifact being signed.
- Signed attributes `v`: The signature metadata that is integrity protected - e.g. signature expiration time, signing time, etc.
- Unsigned attributes `u`: This OPTIONAL property represents signature metadata that is not integrity protected - e.g. timestamp, certificates, etc.
- Cryptographic signatures `s`: The digital signatures computed on payload and signed attributes.

A signature envelope is `e = {m, v, u, s}` where `s` is signature.

Notary v2 supports [COSE_Sign1_Tagged](https://datatracker.ietf.org/doc/html/rfc8152#section-4) as signature envelope format with some additional constraints but makes provisions to support additional signature envelope format.

### Payload

Notary v2 requires Payload to be the content **descriptor** of the subject manifest that is being signed.

1. Descriptor MUST contain `mediaType`, `digest`, `size` fields.
1. Descriptor MAY contain `annotations` and if present it MUST follow the [annotation rules](https://github.com/opencontainers/image-spec/blob/main/annotations.md#rules). Notary v2 uses annotations for storing both Notary specific and user defined signed attributes. The prefix `org.cncf.notary` in annotation keys is reserved for use in Notary v2 and MUST NOT be used outside this specification.
1. Descriptor MAY contain `artifactType` field for artifact manifests, or the `config.mediaType` for `oci.image` based manifests.

Examples: The media type of the content descriptor is `application/vnd.cncf.oras.artifact.descriptor.v1+json`.

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

```json
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
  This is an OPTIONAL attribute.

### Unsigned Attributes

- **Certificates**: The ordered collection of X.509 certificates with a signing certificate as the first entry.
- **Timestamp**:  The time stamp token generated for a given signature.
  Only [RFC3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2) compliant *TimeStampToken* are supported.
  This is an OPTIONAL attribute.

### Supported Signature Envelopes

#### COSE_Sign1_Tagged

In COSE ([rfc8152](https://datatracker.ietf.org/doc/html/rfc8152)), data is stored as either payload or headers (protected and unprotected).
Notary v2 uses [COSE_Sign1_Tagged](https://datatracker.ietf.org/doc/html/rfc8152#section-4.2) object as the signature envelope with some additional constraints on the header fields.

Unless explicitly specified as OPTIONAL, all fields are required.

**Payload**: COSE signs the payload as defined in the [Payload](#payload) section.

**ProtectedHeaders**: Notary v2 supports only the following protected headers:

```
{
  / crit / 2: [
    / cty / 3,
    'signingtime',
    'expiry'
  ],
  / cty / 3: 'application/vnd.cncf.oras.artifact.descriptor.v1+json',
  'signingtime': 1234567891000,
  'expiry': 1234567891011
}
```

Note: The above example is represented using the [extended CBOR diagnostic notation](https://datatracker.ietf.org/doc/html/rfc8152#appendix-C).

- **`crit`** (*array of integers or strings*): This REQUIRED property (label: `2`) lists the headers that implementation MUST understand and process.
  The array MUST contain `3` (`cty`), and `signingtime`. If `expiry` is presented, the array MUST also contain `expiry`.
- **`cty`** (*string*): The REQUIRED property content-type (label: `3`) is used to declare the media type of the secured content (the payload).
- **`signingtime`** (*integer*): The REQUIRED property identifies the time at which the signature was generated.
- **`expiry`** (*integer*): This OPTIONAL property contains the expiration time on or after which the signature must not be considered valid.

**UnprotectedHeaders**: Notary v2 supports only two unprotected headers: `timestamp` and `x5chain`.

```
{
  'timestamp': << TimeStampToken >>,
  / x5chain / 33: [
    << DER(leafCert) >>,
    << DER(intermediate CACert) >>,
    << DER(rootCert) >>
  ]
}
```

Note: `<<` and `>>` are used to notate the byte string resulting from encoding the data item.

- **`timestamp`** (*byte string*): This OPTIONAL property is used to store time stamp token.
  Only [RFC3161]([rfc3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2)) compliant TimeStampToken are supported.
- **`x5chain`** (*array of byte strings*): This REQUIRED property (label: `33` by [IANA](https://www.iana.org/assignments/cose/cose.xhtml#header-parameters)) contains the list of X.509 certificate or certificate chain ([RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)) corresponding to the key used to digitally sign the COSE.
  The certificate containing the public key corresponding to the key used to digitally sign the COSE MUST be the first certificate.
  Optionally, this header can be presented in the protected header.

**Signature**: In COSE, signature is calculated by constructing the `Sig_structure` for `COSE_Sign1`.
The process is described below:

1. Encode the protected header into a CBOR object as a byte string named `body_protected`.
2. Construct the `Sig_structure` for `COSE_Sign1`.
    ```
    Sig_structure = [
        / context / 'Signature1',
        / body_protected / << ProtectedHeaders >>,
        / external_aad / h'',
        / payload / << Payload >>,
    ]
    ```
3. Encode `Sig_structure` into a CBOR object as a byte string named `ToBeSigned`.
4. Compute the signature on the `ToBeSigned` constructed in the previous step by using the signature algorithm defined in the corresponding header element: `alg`.
   This is the value of the signature property used in the signature envelope.

**Signature Envelope**: The final signature envelope is a `COSE_Sign1_Tagged` object, consisting of Payload, ProtectedHeaders, UnprotectedHeaders, and Signature.

```
18(
  [
    / protected / << {
      / crit / 2: [
        / cty / 3,
        'signingtime',
        'expiry'
      ],
      / cty / 3: 'application/vnd.cncf.oras.artifact.descriptor.v1+json',
      'signingtime': 1234567891000,
      'expiry': 1234567891011
    } >>,
    / unprotected / {
      'timestamp': << TimeStampToken >>,
      / x5chain / 33: [
        << DER(leafCert) >>,
        << DER(intermediate CACert) >>,
        << DER(rootCert) >>
      ]
    },
    / payload / << descriptor >>,
    / signature / << sign( << Sig_structure >> ) >>
  ]
)
```

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

**Q: How will Notary v2 support multiple signature envelope format?**

**A:** The idea is to use `mediaType` of artifact manifest's blob to identify the signature envelope type (like COSE, JWS, CMS, DSSE, etc).
The client implementation can use the aforementioned `mediaType` to parse the signature envelope.

**Q: How will Notary v2 handle non-backward compatible changes to signature format?**

**A:** The Signature envelope MUST have a versioning mechanism to support non-backward compatible changes.
For [COSE_Sign1_Tagged](#cose_sign1_tagged) signature envelope it is achieved by `cty` field in ProtectedHeaders.
