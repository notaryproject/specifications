# JWS Signature Envelope Serialization

JWS JSON Serialization ([RFC7515](https://datatracker.ietf.org/doc/html/rfc7515)) data is stored as either claims or headers (protected and unprotected).
Notary v2 supports JWS JSON Serialization as a signature envelope with some additional constraints on the structure of claims and headers.

A JWS based signature will be persisted with a blob `mediaType` of `"application/jose+json"`

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
        "io.cncf.notary.x509certs.fingerprint.sha256": "[\"B7A69A70992AE4F9FF103EBE04A2C3BA6C777E439253CE36562E6E98375068C3\" \"932EB6F5598435D4EF23F97B0B5ACB515FAE2B8D8FAC046AB813DDC419DD5E89\"]"
    }
}
```

Unless explicitly specified as OPTIONAL, all fields are required.
Also, there shouldn’t be any additional fields other than ones specified in JWSPayload, ProtectedHeader, and UnprotectedHeader.

**JWSPayload a.k.a. Claims**:
Notary v2 is using one private claim (`notary`) and two public claims (`iat` and `exp`).
An example of the claim is described below

```jsonc
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

```jsonc
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

```jsonc
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
2. Compute the Base64Url value of JWSPayload.
3. Build message to be signed by concatenating the values generated in step 1 and step 2 using '.'
`ASCII(BASE64URL(UTF8(ProtectedHeaders)) ‘.’ BASE64URL(JWSPayload))`
1. Compute the signature on the message constructed in the previous step by using the signature algorithm defined in the corresponding header element: `alg`.
1. Compute the Base64Url value of the signature produced in the previous step.
   This is the value of the signature property used in the signature envelope.

**Signature Envelope**: The final signature envelope comprises of Claims, ProtectedHeaders, UnprotectedHeaders, and signature.

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

1. Only JWS JSON flattened format is supported.
   See 'Signature Envelope' section.