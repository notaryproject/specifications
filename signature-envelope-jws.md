# JWS Signature Envelope

JWS ([RFC7515](https://datatracker.ietf.org/doc/html/rfc7515)) is a JSON based envelope format for digital signatures over any type of payload (e.g. JSON, binary). Notary v2 uses JWS as a supported signature format. It specifically uses the *JWS JSON Serialization* representation, which supports unsigned attributes. JWS Compact serialization, and JWT (also based on Compact serialization), do not support unsigned attributes.

## Storage

A JWS based signature will be persisted with a blob `mediaType` of `"application/jose+json"`

Example

```jsonc
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
        "io.cncf.notary.x509.fingerprint.sha256": "[\"B7A69A70992AE4F9FF103EBE04A2C3BA6C777E439253CE36562E6E98375068C3\" \"932EB6F5598435D4EF23F97B0B5ACB515FAE2B8D8FAC046AB813DDC419DD5E89\"]"
    }
}
```

## JWS Payload

The payload in the JWS envelope is set to BASE64URL encoded value of [Notary v2 Payload](./signature-specification.md#payload).

Example of payload before BASE64URL encoding.

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

- Registered headers `alg`, `cty`, and `crit`
- Public headers `io.cncf.notary.signingTime`, `io.cncf.notary.expiry`, `io.cncf.notary.clientID`

Example

```jsonc
{
    "alg": "PS384",
    "cty": "application/vnd.cncf.notary.v2.jws.v1",
    "io.cncf.notary.signingTime": "2022-04-06 07:01:20.52Z",
    "io.cncf.notary.expiry": "2022-10-06 07:01:20.52Z",
    "io.cncf.notary.clientID": "notation/1.0.0",
    "crit":["io.cncf.notary.expiry"]
}
```

- **[`alg`](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.1)**(*string*): This REQUIRED header defines which algorithm was used to generate the signature.JWS specification defines `alg` as a required header, that MUST be present and MUST be understood and processed by verifier. This [section](#supported-alg-header-values) lists the Notary v2 allowed subset of `alg` values supported by JWS.
- **[`cty`](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.10)**(*string*): The REQUIRED header content-type is used to declare the media type of the secured content (the payload). The supported value is `application/vnd.cncf.notary.payload.v1+json`.
- **`io.cncf.notary.signingTime`**(*string*): This REQUIRED header specifies the time at which the signature was generated. Its value is a RFC 3339 formatted date time.
- **`io.cncf.notary.expiry`**(*string*): This OPTIONAL header provides a “best by use” time for the artifact, as defined by the signer. Its value is a RFC 3339 formatted date time.
- **`io.cncf.notary.clientID`**(*string*): This OPTIONAL header provides The version of a client (e.g. Notation) that produced the signature. E.g. “notation/1.0.0”.
- **[`crit`](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.11)**(*array of strings*): This OPTIONAL header lists the headers that implementation MUST understand and process. It MUST only contain headers apart from registered headers (e.g. `alg`, `cty`) in JWS specification, therefore this header is only present when the optional `io.cncf.notary.expiry` header is present in the protected headers collection.
  If present, the value MUST be `["io.cncf.notary.expiry"]`.

## Unprotected Headers

Notary v2 supports only two unprotected headers: `timestamp` and `x5c`.

```jsonc
{
    "timestamp": "<Base64(TimeStampToken)>",
    "x5c": ["<Base64(DER(leafCert))>", "<Base64(DER(intermediateCACert))>", "<Base64(DER(rootCert))>"]
}
```

- **`timestamp`** (*string*): This OPTIONAL property is used to store time stamp token.
  Only [RFC3161]([rfc3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2)) compliant TimeStampToken are supported.
- **[`x5c`](https://datatracker.ietf.org/doc/html/rfc7515#section-4.1.6)** (*array of strings*): This REQUIRED property contains the list of X.509 certificate or certificate chain([RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)) corresponding to the key used to digitally sign the JWS. The certificate or certificate chain is represented as a JSON array of certificate value strings. Each string in the array is a base64-encoded DER  certificate value. The certificate containing the public key corresponding to the key used to digitally sign the JWS MUST be the first certificate.

## Signature

In JWS signature is calculated by combining JWSPayload and protected headers.
The process is described below:

### Create the *Payload to Sign*

1. Compute the Base64Url value of ProtectedHeaders.
2. Compute the Base64Url value of JWSPayload.
3. Build message to be signed by concatenating the values generated in step 1 and step 2 using '.'
`ASCII(BASE64URL(UTF8(ProtectedHeaders)) ‘.’ BASE64URL(JWSPayload))`

### Generate the signature

1. Compute the signature on the message constructed in the previous step by using the signature algorithm defined in the corresponding header element: `alg`.
2. Compute the Base64Url value of the signature produced in the previous step.
   This is the value of the signature property used in the signature envelope.

## Signature Envelope

The final signature envelope comprises of Payload, ProtectedHeaders, UnprotectedHeaders, and Signature, no additional top level fields are supported.

Since Notary v2 restricts one signature per signature envelope, the compliant signature envelope MUST be in flattened JWS JSON format.

```jsonc
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

## Implementation Constraints

### Supported `alg` header values

Notary v2 implementation MUST enforce the following constraints on signature generation and verification:

1. `alg` header value MUST NOT be `none` or any symmetric-key algorithm such as `HMAC`.
2. `alg` header value MUST be same as that of signature algorithm identified using signing certificate's public key algorithm and size.
3. `alg` header values for various signature algorithms is a subset of values supported by [JWS][jws-alg-values]:

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

## FAQ

**Q:** Why JWT `exp` and `iat` claims are not used?

**A:** Unlike JWT which always contains a JSON payload, Notary v2 envelope can support payloads other than JSON, like binary. Reusing the JWT payload structure and claims, limits the Notary v2 JWS envelope to only support JSON payload, which is undesirable. Also, reusing JWT claims requires following same claim semantics as defined in JWT specifications. The [`exp`](https://datatracker.ietf.org/doc/html/rfc7519#section-4.1.4) claim requires that verifier MUST reject the signature if current time equals or is greater than `exp`, where as Notary v2 allows verification policy to define how expiry is handled.

[jws-alg-values]: https://datatracker.ietf.org/doc/html/rfc7518#section-3.1
