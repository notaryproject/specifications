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

#### COSE_Sign1_Tagged

In COSE ([rfc8152](https://datatracker.ietf.org/doc/html/rfc8152)), data is stored as either payload or headers (protected and unprotected).
Notary v2 uses [COSE_Sign1_Tagged](https://datatracker.ietf.org/doc/html/rfc8152#section-4.2) object as the signature envelope with some additional constraints on the header fields.

Unless explicitly specified as OPTIONAL, all fields are required.

**Payload**:
Both `JSON` and `CBOR` payload (as defined in the [Payload](#payload) section) are accepted by Notary v2.

**ProtectedHeaders**: Notary v2 supports only the following protected headers:

```sql
a5 -- Map of size 5
   01       -- Key:   Integer: 1    -- `alg`: algorithm
   38 24    -- Value: Integer: -37  -- `PS256`
   02       -- Key:   Integer: 2    -- `cty`: content type
   78 35    -- Value: UTF-8 text: 53 bytes
      61 70 70 6c 69 63 61 74 69 6f 6e 2f 76 6e 64 2e -- application/vnd.
      63 6e 63 66 2e 6f 72 61 73 2e 61 72 74 69 66 61 -- cncf.oras.artifa
      63 74 2e 64 65 73 63 72 69 70 74 6f 72 2e 76 31 -- ct.descriptor.v1
      2b 63 62 6f 72                                  -- +cbor
   03       -- Key:   Integer: 3    -- `crit`: critical headers
   82       -- Value: Array of length 2
      02 -- Integer: 2  -- `cty`: content type
      6b -- UTF-8 text: 11 bytes
         73 69 67 6e 69 6e 67 74 69 6d 65                -- signingtime
   6b       -- Key:   UTF-8 text: 11 bytes
      73 69 67 6e 69 6e 67 74 69 6d 65                -- signingtime
   1b 00 00 01 1f 71 fb 08 38 -- Value: Integer: 1234567891000
   66       -- Key:   UTF-8 text: 6 bytes
      65 78 70 69 72 79                               -- expiry
   1b 00 00 01 1f 71 fb 08 43 -- Value: Integer: 1234567891011
```

- **`alg`**(*integer*): This REQUIRED property (label: `1`) defines which algorithm was used to generate the signature.
  COSE needs an algorithm(`alg`) to be present in the header, so we have added it as a protected header.
- **`cty`**(*string*): The REQUIRED property content-type(cty) (label: `2`) is used to declare the media type of the secured content(the payload).
- **`crit`**(*array of integers or strings*): This REQUIRED property (label: `3`) lists the headers that implementation MUST understand and process.
  The array MUST contain `2` (`cty`), and `signingtime`. If `expiry` is presented, the array MUST also contain `expiry`.
- **`signingtime`**(*integer*): The REQUIRED property identifies the time at which the signature was generated.
- **`expiry`**(*integer*): This OPTIONAL property contains the expiration time on or after which the signature must not be considered valid.

**UnprotectedHeaders**: Notary v2 supports only two unprotected headers: `timestamp` and `x5c`.

```sql
a2 -- Map of size 2
   69       -- Key:   UTF-8 text: 9 bytes
      74 69 6d 65 73 74 61 6d 70                      -- timestamp
   4e       -- Value: Binary string: 14 bytes
      54 69 6d 65 53 74 61 6d 70 54 6f 6b 65 6e       -- TimeStampToken
   18 21    -- Key:   Integer: 33   -- `x5chain`
   83       -- Value: Array of length 3
      4d -- Binary string: 13 bytes
         44 45 52 28 6c 65 61 66 43 65 72 74 29          -- DER(leafCert)
      57 -- Binary string: 23 bytes
         44 45 52 28 69 6e 74 65 72 6d 65 64 69 61 74 65 -- DER(intermediate
         43 41 43 65 72 74 29                            -- CACert)
      4d -- Binary string: 13 bytes
         44 45 52 28 72 6f 6f 74 43 65 72 74 29          -- DER(rootCert)
```

- **`timestamp`** (*binary string*): This OPTIONAL property is used to store time stamp token.
  Only [RFC3161]([rfc3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2)) compliant TimeStampToken are supported.
- **`x5chain`** (*array of binary strings*): This REQUIRED property (label: `33` by [IANA](https://www.iana.org/assignments/cose/cose.xhtml)) contains the list of X.509 certificate or certificate chain([RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)) corresponding to the key used to digitally sign the COSE.
  The certificate containing the public key corresponding to the key used to digitally sign the COSE MUST be the first certificate.
  Optionally, this header can be presented in the protected header.

**Signature**: In COSE signature is calculated by constructing the `Sig_structure` for `COSE_Sign1`.
The process is described below:

1. Encode the protected header into a CBOR object as a binary string named `body_protected`.
1. Construct the `Sig_structure` for `COSE_Sign1`.
    ```
    Sig_structure = [
        context : "Signature1",
        body_protected : bstr,
        external_aad : empty_bstr,
        payload : bstr
    ]
    ```
1. Encode `Sig_structure` into a CBOR object as a binary string named `ToBeSigned`.
1. Compute the signature on the `ToBeSigned` constructed in the previous step by using the signature algorithm defined in the corresponding header element: `alg`.
   This is the value of the signature property used in the signature envelope.

**Signature Envelope**: The final signature envelope is a `COSE_Sign1_Tagged` object, consisting of Payload, ProtectedHeaders, UnprotectedHeaders, and Signature.

```
header_map = {
   * label => values
}

COSE_Sign1 = [
   protected : bstr .cbor header_map,
   unprotected : {
       timestamp: TimeStampToken,
       x5chain: [
           leafCert,
           intermediateCACert,
           rootCert
       ]
   },
   payload : bstr,
   signature : bstr
]

COSE_Sign1_Tagged = #6.18(COSE_Sign1)
```

**Implementation Constraints**: Notary v2 implementation MUST enforce the following constraints on signature generation and verification:

1. `alg` header value MUST have the same constraints as the **JWS JSON Serialization** envelope format.

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

**A:** The idea is to use `mediaType` of artifact manifest's blob to identify the signature envelope type (like JWS, CMS, DSSE, etc).
The client implementation can use the aforementioned `mediaType` to parse the signature envelope.

**Q: How will Notary v2 handle non-backward compatible changes to signature format?**

**A:** The Signature envelope MUST have a versioning mechanism to support non-backward compatible changes.
For [JWS JSON serialization](#jws-json-serialization) signature envelope it is achieved by `cty` field in ProtectedHeaders.
