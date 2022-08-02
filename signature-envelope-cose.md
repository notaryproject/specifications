# COSE Sign1 Signature Envelope

This specification implements the [Notary v2 Signature specification](signature-specification.md) using 
CBOR Object Signing and Encryption (COSE). COSE ([RFC8152](https://datatracker.ietf.org/doc/html/rfc8152)) is a CBOR based envelope format for digital signatures over any type of payload (e.g. CBOR, JSON, binary).
Notary v2 specifically supports [COSE_Sign1_Tagged](https://datatracker.ietf.org/doc/html/rfc8152#section-4.2) as a signature envelope.

## Storage

A COSE signature envelope will be stored in an OCI registry as a blob, and referenced in the signature manifest as a blob with `mediaType` of `"application/cose"`.

Signature Manifest Example

```jsonc
{
    "mediaType": "application/vnd.cncf.oras.artifact.manifest.v1+json",
    "artifactType": "application/vnd.cncf.notary.signature",
    "blobs": [
        {
            "mediaType": "application/cose",
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

## COSE Payload

The COSE envelope contains a [Notary v2 Payload](./signature-specification.md#payload).

## Protected Header

The COSE envelope for Notary v2 uses the following header parameters:

- [Common parameters](https://www.iana.org/assignments/cose/cose.xhtml#header-parameters)
  - Label `1`: `alg`
  - Label `2`: `crit`
  - Label `3`: `content type`
- Additional parameters with collision resistant names.
  - `io.cncf.notary.signingScheme`
  - `io.cncf.notary.signingTime`
  - `io.cncf.notary.authenticSigningTime`
  - `io.cncf.notary.expiry`

Example with Signing Scheme `notary.x509`

```yaml
{
  / alg / 1: / PS384 / -38,
  / crit / 2: [
    'io.cncf.notary.signingScheme',
    'io.cncf.notary.expiry'
  ],
  / cty / 3: 'application/vnd.cncf.notary.payload.v1+json',
  'io.cncf.notary.signingScheme': 'notary.x509',
  'io.cncf.notary.signingTime': 1234567890,
  'io.cncf.notary.expiry': 1234567891
}
```

Example with Signing Scheme `notary.x509.signingAuthority`

```yaml
{
  / alg / 1: / PS384 / -38,
  / crit / 2: [
    'io.cncf.notary.signingScheme',
    'io.cncf.notary.authenticSigningTime',
    'io.cncf.notary.expiry'
  ],
  / cty / 3: 'application/vnd.cncf.notary.payload.v1+json',
  'io.cncf.notary.signingScheme': 'notary.x509.signingAuthority',
  'io.cncf.notary.authenticSigningTime': 1234567890,
  'io.cncf.notary.expiry': 1234567891
}
```

Note: The above examples are represented using the [extended CBOR diagnostic notation](https://datatracker.ietf.org/doc/html/rfc8152#appendix-C).

- **[`alg`](https://datatracker.ietf.org/doc/html/rfc8152#section-3.1)** (*int*): This REQUIRED parameter (label `1`) defines which signing algorithm was used to generate the signature. The signature algorithm of the signing key (first certificate in `x5chain`) is the source of truth, and during signing the value of `alg` MUST be set corresponding to signature algorithm of the signing key using [this mapping](#supported-alg-header-values) that lists the Notary v2 allowed subset of `alg` values supported by COSE. Similarly verifier of the signature MUST match `alg` with signature algorithm of the signing key to mitigate algorithm substitution attacks.
- **[`crit`](https://datatracker.ietf.org/doc/html/rfc8152#section-3.1)** (*array of int/tstr*): This REQUIRED parameter (label `2`) lists the header parameters that implementations MUST understand and process. It MUST only contain parameters apart from integer labels in the range of 0 to 8. This header MUST contain `io.cncf.notary.signingScheme` which is a required critical header, and optionally contain `io.cncf.notary.authenticSigningTime` and `io.cncf.notary.expiry` if these critical headers are present in the signature.
- **[`content type`](https://datatracker.ietf.org/doc/html/rfc8152#section-3.1)** (*tstr*): The REQUIRED parameter content type (label `3`) is used to declare the media type of the secured content (the payload). The supported value is `application/vnd.cncf.notary.payload.v1+json`.
- **`io.cncf.notary.signingScheme`** (*tstr*, critical): This REQUIRED header specifies the [Notary v2 Signing Scheme](./signing-scheme.md) used by the signature. Supported values are `notary.x509` and `notary.x509.signingAuthority`.
- **`io.cncf.notary.signingTime`** (*uint*): This header specifies the time at which the signature was generated. This is an untrusted timestamp, and therefore not used in trust decisions. Its value is the number of seconds from `1970-01-01T00:00Z` in UTC time, commonly known as UNIX timestamp. This claim is REQUIRED and only valid when signing scheme is `notary.x509`.
- **`io.cncf.notary.authenticSigningTime`** (*uint*, critical): This header specifies the authenticated time at which the signature was generated. Its value is the number of seconds from `1970-01-01T00:00Z` in UTC time, commonly known as UNIX timestamp. This claim is REQUIRED and only valid when signing scheme is `notary.x509.signingAuthority` .
- **`io.cncf.notary.expiry`** (*uint*, critical): This OPTIONAL header provides a "best by use" time for the artifact, as defined by the signer. Its value is the number of seconds from `1970-01-01T00:00Z` in UTC time, commonly known as UNIX timestamp.

## Unprotected Headers

Notary v2 supports the following unprotected header parameters:

- `io.cncf.notary.timestampSignature`
- Label `33`: `x5chain`
- `io.cncf.notary.signingAgent`

```yaml
{
  / x5chain / 33: [
    << DER(leafCert) >>,
    << DER(intermediate CACert) >>,
    << DER(rootCert) >>
  ],
  'io.cncf.notary.timestampSignature': << TimeStampToken >>,
  'io.cncf.notary.signingAgent': 'notation/1.0.0'
}
```

Note: `<<` and `>>` are used to notate the byte string resulting from encoding the data item.

- **[`x5chain`](https://datatracker.ietf.org/doc/html/draft-ietf-cose-x509-08#section-2)** (*array of bstr*): This REQUIRED parameter (label `33` by [IANA](https://www.iana.org/assignments/cose/cose.xhtml#header-parameters)) contains the ordered list of X.509 certificate or certificate chain ([RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)) corresponding to the key used to digitally sign the COSE. The certificate chain is represented as an array of certificate value byte string, each certificate in the array is DER encoded. The certificate containing the public key corresponding to the key used to digitally sign the COSE MUST be the first certificate, followed by the intermediate and root certificates in the correct order. Refer [*Certificate Chain* unsigned attribute](signature-specification.md#unsigned-attributes) for more details. Optionally, this header can be presented in the protected header.
  - **TODO** update signature specification to allow chains in protected header
- **`io.cncf.notary.timestampSignature`** (*bstr*): This OPTIONAL header is used to store countersignature that provides authentic signing time. Only [RFC3161]([rfc3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2)) compliant `TimeStampToken` are supported.
  - **TODO** Define the opaque datum (hash of envelope) that is sent to TSA, and how TSA response (time stamp token) is represented in this header.
- **`io.cncf.notary.signingAgent`** (*tstr*): This OPTIONAL header provides the identifier of a client (e.g. Notation) that produced the signature. E.g. `notation/1.0.0`. Refer [*Signing Agent* unsigned attribute](signature-specification.md#unsigned-attributes) for more details.

## Signature

In COSE, signature is calculated by constructing the `Sig_structure` for `COSE_Sign1`.
The process is described below:

1. Encode the protected header into a CBOR object as a byte string named `body_protected`.
2. Construct the `Sig_structure` for `COSE_Sign1`.
    ```yaml
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

## Signature Envelope

The final signature envelope is a `COSE_Sign1_Tagged` object, consisting of Payload, ProtectedHeaders, UnprotectedHeaders, and Signature.

```yaml
18(
  [
    / protected / << {
      / alg / 1: / PS384 / -38,
      / crit / 2: [
        'io.cncf.notary.signingScheme',
        'io.cncf.notary.authenticSigningTime',
        'io.cncf.notary.expiry'
      ],
      / cty / 3: 'application/vnd.cncf.notary.payload.v1+json',
      'io.cncf.notary.signingScheme': 'notary.x509.signingAuthority',
      'io.cncf.notary.authenticSigningTime': 1234567890,
      'io.cncf.notary.expiry': 1234567891
    } >>,
    / unprotected / {
      / x5chain / 33: [
        << DER(leafCert) >>,
        << DER(intermediate CACert) >>,
        << DER(rootCert) >>
      ],
      'io.cncf.notary.timestampSignature': << TimeStampToken >>,
      'io.cncf.notary.signingAgent': 'notation/1.0.0'
    },
    / payload / << descriptor >>,
    / signature / << sign( << Sig_structure >> ) >>
  ]
)
```

## Implementation Constraints

### Supported `alg` header values

Notary v2 implementation MUST enforce the following constraints on signature generation and verification:

1. `alg` parameter value MUST NOT be a symmetric-key algorithm such as `HMAC`.
1. `alg` parameter value MUST be same as that of signature algorithm identified using signing certificate's public key algorithm and size.
1. `alg` parameter values for various signature algorithms is a subset of values supported by [COSE](https://www.iana.org/assignments/cose/cose.xhtml#algorithms).

**Mapping of Notary v2 approved algorithms to COSE `alg` header parameter values**

  | Signature Algorithm             | `alg` Label       |
  | ------------------------------- | ----------------- |
  | RSASSA-PSS with SHA-256         | -37               |
  | RSASSA-PSS with SHA-384         | -38               |
  | RSASSA-PSS with SHA-512         | -39               |
  | ECDSA on secp256r1 with SHA-256 | -7                |
  | ECDSA on secp384r1 with SHA-384 | -35               |
  | ECDSA on secp521r1 with SHA-512 | -36               |
