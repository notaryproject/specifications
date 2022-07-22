# JWS Signature Envelope

This specification implements the [Notary v2 Signature specification](signature-specification.md) using JSON Web Signature (JWS). JWS ([RFC7515](https://datatracker.ietf.org/doc/html/rfc7515)) is a JSON based envelope format for digital signatures over any type of payload (e.g. JSON, binary). JWS is a Notary v2 supported signature format and specifically uses the *JWS JSON Serialization* representation.

## Storage

A JWS signature envelope will be stored in an OCI registry as a blob, and referenced in the signature manifest as a blob with `mediaType` of `"application/jose+json"`.

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

## JWS Payload

The JWS envelope contains a [Notary v2 Payload](./signature-specification.md#payload).

Example of Notary v2 payload

```jsonc
{
  "subject": {
    "mediaType": "application/vnd.oci.image.manifest.v1+json",
    "digest": "sha256:73c803930ea3ba1e54bc25c2bdc53edd0284c62ed651fe7b00369da519a3c333",
    "size": 16724,
    "annotations": {
        "io.wabbit-networks.buildId": "123"  // user defined metadata
    }
  }
}
```

## Protected Headers

The JWS envelope for Notary v2 uses following headers

- Registered headers - `alg`, `cty`, and `crit`
- [Public headers](https://datatracker.ietf.org/doc/html/rfc7515#section-4.2) with collision resistant names.
  - `io.cncf.notary.signingScheme`
  - `io.cncf.notary.signingTime`
  - `io.cncf.notary.authenticSigningTime`
  - `io.cncf.notary.expiry`

Example with Signing Scheme `notary.x509`

```jsonc
{
    "alg": "PS384",
    "cty": "application/vnd.cncf.notary.payload.v1+json",
    "io.cncf.notary.signingScheme": "notary.x509",
    "io.cncf.notary.signingTime": "2022-04-06T07:01:20Z",
    "io.cncf.notary.expiry": "2022-10-06T07:01:20Z",
    "crit":[
      "io.cncf.notary.signingScheme",
      "io.cncf.notary.expiry"
    ]
}
```

Example with Signing Scheme `notary.x509.signingAuthority`

```jsonc
{
    "alg": "PS384",
    "cty": "application/vnd.cncf.notary.payload.v1+json",
    "io.cncf.notary.signingScheme": "notary.x509.signingAuthority",
    "io.cncf.notary.authenticSigningTime": "2022-04-06T07:01:20Z",
    "io.cncf.notary.expiry": "2022-10-06T07:01:20Z",
    "crit":[
      "io.cncf.notary.signingScheme",
      "io.cncf.notary.authenticSigningTime",
      "io.cncf.notary.expiry"]
}
```

- **[`alg`](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.1)**(*string*): This REQUIRED header defines which signing algorithm was used to generate the signature. JWS specification defines `alg` as a required header, that MUST be present and MUST be understood and processed by verifier. The signature algorithm of the signing key (first certificate in `x5c`) is the source of truth, and during signing the value of `alg` MUST be set corresponding to signature algorithm of the signing key using [this mapping](#supported-alg-header-values) that lists the Notary v2 allowed subset of `alg` values supported by JWS. Similarly verifier of the signature MUST match `alg` with signature algorithm of the signing key to mitigate algorithm substitution attacks.
- **[`cty`](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.10)**(*string*): The REQUIRED header content-type is used to declare the media type of the secured content (the payload). The supported value is `application/vnd.cncf.notary.payload.v1+json`.
- **`io.cncf.notary.signingScheme`**(*string*)(critical): This REQUIRED header specifies the [Notary v2 Signing Scheme](./signing-scheme.md) used by the signature. Supported values are `notary.x509` and `notary.x509.signingAuthority`.
- **`io.cncf.notary.signingTime`**(*string*): This header specifies the time at which the signature was generated. This is an untrusted timestamp, and therefore not used in trust decisions. Its value is a [RFC 3339][rfc3339] formatted date time, the optional fractional second ([time-secfrac][rfc3339][[1](https://datatracker.ietf.org/doc/html/rfc3339#section-5.3)]) SHOULD NOT be used. This claim is REQUIRED and only valid when signing scheme is `notary.x509`.
- **`io.cncf.notary.authenticSigningTime`**(*string*)(critical): This header specifies the authenticated time at which the signature was generated. Its value is a [RFC 3339][rfc3339] formatted date time, the optional fractional second ([time-secfrac][rfc3339][[1](https://datatracker.ietf.org/doc/html/rfc3339#section-5.3)]) SHOULD NOT be used. This claim is REQUIRED and only valid when signing scheme is `notary.x509.signingAuthority` .
- **`io.cncf.notary.expiry`**(*string*)(critical): This OPTIONAL header provides a “best by use” time for the artifact, as defined by the signer. Its value is a [RFC 3339][rfc3339] formatted date time, the optional fractional second ([time-secfrac][rfc3339][[1](https://datatracker.ietf.org/doc/html/rfc3339#section-5.3)]) SHOULD NOT be used.
- **[`crit`](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.11)**(*array of strings*): This REQUIRED (optional as per JWS spec, but required in Notary v2 JWS signature) header lists the headers that implementation MUST understand and process. It MUST only contain headers apart from registered headers (e.g. `alg`, `cty`) in JWS specification. This header MUST contain `io.cncf.notary.signingScheme` which is a required critical header, and optionally contain `io.cncf.notary.authenticSigningTime` and `io.cncf.notary.expiry` if these critical headers are present in the signature.

## Extended Protected Headers for *Notation* Plugins

See [Extended attributes for *Notation* Plugins](./signature-specification.md#extended-attributes-for-notation-plugins) for detailed description of these headers.

- **`io.cncf.notary.verificationPlugin`**(*string*)(critical): An OPTIONAL header that specifies the name of the verification plugin that MAY be used to verify the signature.
- **`io.cncf.notary.verificationPluginMinVersion`**(*string*)(critical): An OPTIONAL header that specifies the minimum version of the verification plugin that MUST be used to verify the signature.

## Unprotected Headers

Notary v2 supports following unprotected headers: `timestamp`, `x5c` and `io.cncf.notary.signingAgent`

```jsonc
{
    "x5c": ["<Base64(DER(leafCert))>", "<Base64(DER(intermediateCACert))>", "<Base64(DER(rootCert))>"],
    "io.cncf.notary.timestampSignature": "<Base64(TimeStampToken)>",
    "io.cncf.notary.signingAgent": "notation/1.0.0"
}
```

- **[`x5c`](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.6)** (*array of strings*): This REQUIRED header contains the ordered list of X.509 certificate or certificate chain([RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)) corresponding to the key used to digitally sign the JWS. The certificate chain is represented as a JSON array of certificate value strings, each string in the array is a base64-encoded DER certificate value. The certificate containing the public key corresponding to the key used to digitally sign the JWS MUST be the first certificate, followed by the intermediate and root certificates in the correct order. Refer [*Certificate Chain* unsigned attribute](signature-specification.md#unsigned-attributes) for more details.
- **`io.cncf.notary.timestampSignature`** (*string*): This OPTIONAL header is used to store countersignature that provides authentic signing time. Only [RFC3161]([rfc3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2)) compliant `TimeStampToken` are supported.
  - **TODO** Define the opaque datum (hash of envelope) that is sent to TSA, and how TSA response (time stamp token) is represented in this header.
- **`io.cncf.notary.signingAgent`**(*string*): This OPTIONAL header provides the identifier of a client (e.g. Notation) that produced the signature. E.g. “notation/1.0.0”. Refer [*Signing Agent* unsigned attribute](signature-specification.md#unsigned-attributes) for more details.

## Signature

In JWS signature is calculated by combining JWSPayload and protected headers.
The process is described below:

### Create the *JWS Signing Input*

1. Compute the Base64Url value of ProtectedHeaders, this is the value of `protected` property in the signature envelope.
1. Compute the Base64Url value of JWSPayload, this is the value of `payload` property in the signature envelope.
1. Build *JWS Signing Input* to be signed by concatenating the values generated in step 1 and step 2 using '.'
`ASCII(BASE64URL(UTF8(ProtectedHeaders)) ‘.’ BASE64URL(JWSPayload))`

Base64Url encoding used by JWS (*Base64url Encoding* in [RFC 7515 section 2][jws-terminology]) is a URL safe Base64 encoding as defined in [RFC 4648][rfc4648], with all trailing '=' characters omitted and without the inclusion of any additional characters.

### Generate the signature

1. Compute the signature on the *JWS Signing Input* constructed in the previous step by using the signature algorithm of the signing key, which MUST match the corresponding protected header `alg`.
2. Compute the Base64Url value of the signature produced in the previous step.
   This is the value of the `signature` property in the signature envelope.

## Signature Envelope

The final signature envelope comprises of Payload, ProtectedHeaders, UnprotectedHeaders, and Signature, no additional top level fields are supported.

Since Notary v2 restricts one signature per signature envelope, the compliant signature envelope MUST be in flattened JWS JSON format.

```jsonc
{
    "payload": "<Base64Url(JWSPayload)>",
    "protected": "<Base64Url(ProtectedHeaders)>",
    "header": {
        "io.cncf.notary.timestamp": "<Base64(TimeStampToken)>",
        "x5c": ["<Base64(DER(leafCert))>", "<Base64(DER(intermediateCACert))>", "<Base64(DER(rootCert))>"]
    },
    "signature": "Base64Url( sign( ASCII( <Base64Url(ProtectedHeader)>.<Base64Url(JWSPayload)> )))"  
}
```

## Implementation Constraints

### Supported `alg` header values

Notary v2 implementation MUST enforce the following constraints on signature generation and verification:

1. `alg` header value MUST NOT be `none` or any symmetric-key algorithm such as `HMAC`.
1. `alg` header value MUST be same as that of signature algorithm identified using signing certificate's public key algorithm and size.
1. `alg` header values for various signature algorithms is a subset of values supported by [JWS][jws-alg-values].

**Mapping of Notary v2 approved algorithms to JWS `alg` header values**

  | Signature Algorithm             | `alg` Header Value|
  | ------------------------------- | ----------------- |
  | RSASSA-PSS with SHA-256         | PS256             |
  | RSASSA-PSS with SHA-384         | PS384             |
  | RSASSA-PSS with SHA-512         | PS512             |
  | ECDSA on secp256r1 with SHA-256 | ES256             |
  | ECDSA on secp384r1 with SHA-384 | ES384             |
  | ECDSA on secp521r1 with SHA-512 | ES512             |

1. Signing certificate MUST be a valid codesigning certificate.
1. Only JWS JSON flattened format is supported.

## FAQ

**Q:** Why JWT is not used as the signature envelope format?

**A:** JWT uses JWS compact serialization which do not support unsigned attributes. Notary v2 signature requires support for unsigned attributes. Instead we use the *JWS JSON Serialization* representation, which supports unsigned attributes.

**Q:** Why JWT `exp` and `iat` claims are not used?

**A:** Unlike JWT which always contains a JSON payload, Notary v2 envelope can support payloads other than JSON, like binary. Reusing the JWT payload structure and claims, limits the Notary v2 JWS envelope to only support JSON payload, which is undesirable. Also, reusing JWT claims requires following same claim semantics as defined in JWT specifications. The [`exp`](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.4) claim requires that verifier MUST reject the signature if current time equals or is greater than `exp`, where as Notary v2 allows verification policy to define how expiry is handled.

[jws-alg-values]: https://datatracker.ietf.org/doc/html/rfc7518#section-3.1
[rfc3339]: https://datatracker.ietf.org/doc/html/rfc3339#section-5.6
[rfc4648]: https://datatracker.ietf.org/doc/html/rfc4648#section-5
[jws-terminology]: https://datatracker.ietf.org/doc/html/rfc7515#section-2
