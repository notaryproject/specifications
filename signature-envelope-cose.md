# COSE Sign1 Envelope Serialization

In COSE ([rfc8152](https://datatracker.ietf.org/doc/html/rfc8152)), data is stored as either payload or headers (protected and unprotected).
Notary v2 supports [COSE_Sign1_Tagged](https://datatracker.ietf.org/doc/html/rfc8152#section-4.2) as a signature envelope with some additional constraints on the header fields.

A JWS based signature will be persisted with a blob `mediaType` of `"application/cose"`

```jsonc
{
   "artifactType": "application/vnd.cncf.notary.v2.signature",
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
        "io.cncf.notary.x509certs.fingerprint.sha256": "[\"B7A69A70992AE4F9FF103EBE04A2C3BA6C777E439253CE36562E6E98375068C3\" \"932EB6F5598435D4EF23F97B0B5ACB515FAE2B8D8FAC046AB813DDC419DD5E89\"]"
    }
}
```

Unless explicitly specified as OPTIONAL, all fields are required.

**Payload**: COSE signs the payload as defined in the [Payload](#payload) section. Detached payloads are not supported.

**ProtectedHeaders**: Notary v2 supports the following protected headers. Other header fields can be included but will be ignored.

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
- **`signingtime`** (*datetime*): The REQUIRED property identifies the time at which the signature was generated.
- **`expiry`** (*datetime*): This OPTIONAL property contains the expiration time on or after which the signature must not be considered valid.

**UnprotectedHeaders**: Notary v2 supports two unprotected headers: `timestamp` and `x5chain`.

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
