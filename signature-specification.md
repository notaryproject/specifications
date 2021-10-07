# Signature Specification

This document describes how Notary v2 signatures are created and stored.
The document consists of following sections:
* **[Storage](#storage)**: Describes how signatures are stored in OCI registry
* **[Signature Envelope](#signature-envelope)**: Describes how signatures are created.


## Storage
This section describes how Notary v2 signatures are stored in the OCI Distribution conformant registry.
Notary v2 uses [ORAS artifact manifest](https://github.com/oras-project/artifacts-spec/blob/main/artifact-manifest.md) to store the signature in the repository. The signature artifact manifest consists of a signature type, a reference to the manifest of the artifact being signed, a blob referencing to the signature, and a collection of annotations.

-<img src="media/signature-specification.jpg" width="550" height="450">

- **`artifactType`**: Used to identify the Notary signature artifact. The supported value is `application/vnd.cncf.notary.v2`.
- **`blobs`**: A collection consisting of only one [artifact descriptor](https://github.com/oras-project/artifacts-spec/blob/main/descriptor.md) that refers to a signature envelope.
   - **`mediaType`**: Defines the media type of signature envelope blob. The supported value is `application/jose+json`
- **`subject`**: An artifact descriptor referencing to image manifest that is signed.
- **`annotations`**: This OPTIONAL property contains arbitrary metadata for the artifact manifest. It can be used to store information about the signature.

```
{
 "artifactType": "application/vnd.cncf.notary.v2",
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
   "org.cncf.notary.signature.subject": "wabbit-networks.io"
 }
}
```

### Signature Discovery
The client should be able to discover all the signatures belonging to an artifact(such as image manifest) by using [ORAS Manifest Referrers API](https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md). ORAS Manifest Referrers API returns a paginated list of all artifacts belonging to a target artifact(such as image manifest). The implementation can filter Notary signature artifacts by either using ORAS Manifest Referrers API or using custom logic on the client. Each Notary signature artifact refers to a signature envelope blob.  


## Signature Envelope
The Signature Envelope is a standard data structure for creating a signed message. A signature envelope consists of the following components:
* Payload(m): The data that is integrity protected - e.g. container image descriptor.
* Signed attributes(v): The signature metadata that is integrity protected - e.g. signature expiration time, signing time, etc.
* Unsigned attributes(u): This OPTIONAL property represents signature metadata that is not integrity protected - e.g. timestamp, certificates, etc.
* Cryptographic signatures(s): The digital signatures computed on payload and signed attributes.

A signature envelope is `e = {m, v, u, s}` where `s` is signature.

Notary v2 supports [JWS JSON Serialization](https://datatracker.ietf.org/doc/html/rfc7515) as signature envelope format with some additional constraints but is extensible to support any other signature envelope format.

### Payload
Notary v2 requires Payload to be the [descriptor](https://github.com/opencontainers/image-spec/blob/master/descriptor.md#properties) of the target manifest that is being signed.  
1. Descriptor MUST contain `mediaType`, `digest`, `size` fields.
1. Descriptor MAY contain `artifactType` field for artifact manifests, or the `config.mediaType` for `oci.image` based manifests or annotations.
```
{
   "mediaType": "application/vnd.oci.image.manifest.v1+json",
   "digest": "sha256:73c803930ea3ba1e54bc25c2bdc53edd0284c62ed651fe7b00369da519a3c333",
   "size": 16724
}
```

### Signed Attributes
Notary v2 requires the signature envelope to support the following signed attributes:
* **Creation time**: The time at which the signature was generated. Its value should be numeric representing the number of seconds(not milliseconds) since Epoch.
* **Expiration time**: The time after which signature shouldn't be considered valid. Its value should be numeric representing the number of seconds since Epoch. This is an OPTIONAL attribute.

### Unsigned Attributes
* **Certificates**: The ordered collection of X.509 certificates with a signing certificate as the first entry.
* **Timestamp**:  The time stamp token generated for a given signature. Only [RFC3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2) compliant TimeStampToken are supported. This is an OPTIONAL attribute.

### Supported Signature Envelopes
#### JWS JSON Serialization
In JWS JSON Serialization([RFC7515](https://datatracker.ietf.org/doc/html/rfc7515)), data is stored as either claims or headers(protected and unprotected).  
Notary v2 uses JWS JSON Serialization for the signature envelope with some additional constraints on the structure of claims and headers.

Unless explicitly specified as OPTIONAL, all fields are mandatory. Also, there shouldn’t be any additional fields other than ones specified in JWSPayload, ProtectedHeader, and UnprotectedHeader.

**JWSPayload a.k.a. Claims**:
Notary v2 is using one private claim(notary) and two public claims(iat and exp). An example of the claim is described below
```json
{
   "notary": {  
       "subject": {
           "mediaType": "application/vnd.oci.image.manifest.v1+json",
           "digest": "sha256:73c803930ea3ba1e54bc25c2bdc53edd0284c62ed651fe7b00369da519a3c333",
           "size": 16724
       },
       "attributes": {
           "reserved": {
               "key1": "value1",
               "key2": "value2"
               ...
           },
           "custom" : {
               "customKey1": "customValue1",
               "customKey2": "customValue2"
               ...
           }
       }
   },
   "iat": 0123456789101,
   "exp": 1234567891011
}
```
The payload contains the subject manifest and other attributes that have to be integrity protected.

* **`notary`**: The top-level node and a private claim encapsulating the notary v2 data.
* **`subject`**: The manifest that needs to be integrity protected.
* **`attributes`**: Contains additional attributes that need to be integrity protected.
   * **`reserved`**: The collection of attributes reserved for notary v2 use such as artifact revocation list, identity, etc. The attributes MUST be key(string)-value(string) pair. These attributes are yet to be defined.
   * **`custom`**: The collection of user-defined attributes such as buildId, imageScanned, etc. The attributes MUST be key(string)-value(string) pair. The use of this field is OPTIONAL.
* **`iat`**: Issued at identifies the time at which the signature was issued.
* **`exp`**: Expiration time identifies the expiration time on or after which the signature must not be considered valid. The use of this claim is OPTIONAL.

To leverage JWS claims validation functionality already provided by libraries, we have defined `iat`, `exp` as top-level nodes.

**ProtectedHeaders**:
Notary v2 uses only 3 protected headers: alg, cty, and crit.
```json
{
   "alg": "RS256",
   "cty": "application/vnd.cncf.notary.v2.jws.v0",
   "crit":["cty"]
}
```
* **`alg`**: JWS needs algorithm(alg) to be present in the header, so we have added it as a protected header.
* **`cty`**: Content type(cty) used to declare the media type of the secured content(the payload). This will be used to version different variotions of JWS signature. The supported value is `application/vnd.cncf.notary.v2.jws.v0`.
* **`crit`**: Indicates the list of headers that implementation MUST be understood and processed. The value MUST be `["cty"]`.

**UnprotectedHeaders**:
Notary v2 uses only 2 unprotected headers: timestamp and x5c.
```
{
   "timestamp": "<Base64Url(TimeStampToken)>",
   "x5c": ["<Base64(DER(leafCert))>", "<Base64(DER(intermediateCACert))>", "<Base64(DER(rootCert))>"]
}
```
* **`timestamp`**: Used to store time stamp token. Only [RFC3161]([rfc3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2)) compliant TimeStampToken are supported. Use of this header is OPTIONAL.
* **`x5c`**: Contains the X.509 public key certificate or certificate chain([RFC5280](https://datatracker.ietf.org/doc/html/rfc5280)) corresponding to the key used to digitally sign the JWS.

**Signature**:
In JWS signature is calculated by combining JWSPayload and protected headers. The process is described below:
1. Compute the base64url value of ProtectedHeaders.
2. Compute the base64url value of JWSPayload.
3. Build message to be signed by concatenating the values generated in step 1 and step 2 using '.'
`ASCII(BASE64URL(UTF8(ProtectedHeaders)) ‘.’ BASE64URL(JWSPayload))`
4. Compute the signature on the message constructed in the previous step by using the signature algorithm defined in the corresponding header element: alg.
5. Compute the base64url value of the signature produced in the previous step. This is the value of the signature property used in the signature envelope.

**Signature Envelope**:
The final signature envelope comprises of Claims, ProtectedHeaders, UnprotectedHeaders, and signature.
```json
{
   "payload": "<Base64Url(JWSPayload)>",
   "signatures": [
       {
           "protected": "<Base64Url(ProtectedHeaders)>",
           "header": {
               "timestamp": "<Base64Url(TimeStampToken)>",
               "x5c": ["<Base64(DER(leafCert))>", "<Base64(DER(intermediateCACert))>", "<Base64(DER(rootCert))>"]
           },
           "signature": "Base64Url( sign( ASCII( <Base64Url(JWSPayload)>.<Base64Url(ProtectedHeader)> )))"  
       }
   ]
}
```
If a JWS contains only one signature as above, the JWS can be flattened as

```json
{
   "payload": "<Base64Url(JWSPayload)>",
   "protected": "<Base64Url(ProtectedHeaders)>"
   "header": {
     "timestamp": "<Base64Url(TimeStampToken)>",
     "x5c": ["<Base64(DER(leafCert))>", "<Base64(DER(intermediateCACert))>", "<Base64(DER(rootCert))>"]
   },
   "signature": "Base64Url( sign( ASCII( <Base64Url(ProtectedHeader)>.<Base64Url(JWSPayload)> )))"  
}
```

++**Implementation Constraints**++
Notary v2 implementation MUST enforce the following constraints on signature generation and verification:
* `alg` header value MUST NOT be `none` or any symmetric-key algorithm such as `HMAC`.
* `alg` header value MUST be same as that of signature algorithm identified using signing certificate's public key algorithm and size.
* `alg` header values for various signature algorithms:
   | Signature Algorithm    | "alg" Param Value |
   | --------    | -------- |
   | ECDSA on secp256r1 with SHA-256 | ES256     |
   | ECDSA on secp256r1 with SHA-512 | ES256     |
   | RSASSA-PSS with SHA-256   | PS256     |
   | RSASSA-PSS with SHA-512   | PS512     |
* Signing certificate MUST be a valid codesigning certificate.

## Signature Algorithms
The implementation MUST support the following set of algorithms:
1. RSASSA-PSS with SHA-256
1. RSASSA-PSS with SHA-384
1. RSASSA-PSS with SHA-512
1. ECDSA on secp256r1 with SHA-256
1. ECDSA on secp384r1 with SHA-384
1. ECDSA on secp256r1 with SHA-512


### Algorithm Selection
The signing certificate's public key algorithm and size MUST be used to determine the signature algorithm.

|Public Key Algorithm| Key Size (bits)   | Signature Algorithm  |
| -------- | --------    | -------- |
|RSA| 2048 | RSASSA-PSS with SHA-256     |
|RSA| 3072 | RSASSA-PSS with SHA-384     |
|RSA| 4096 | RSASSA-PSS with SHA-512     |
|ECDSA| 256   | ECDSA on secp256r1 with SHA-256     |
|ECDSA| 384   | ECDSA on secp384r1 with SHA-384     |
|ECDSA| 512   | ECDSA on secp521r1 with SHA-512     |

### FAQ
1. How will Notary v2 support multiple signature envelope format?

    The general idea is to use `mediaType` of artifact manifest's blob to identify the signature artifact(like jws, cms, dsse,etc). The client implementation can use aformentioned `mediaType` to parse the signature envelope.
    
1. How will Noatary v2 handle non-backward compatible changes to signature format.
    
    Signature envelope MUST have versioning  mechanism to support non-backward compatible changes.