# Signature Specification

This document specifies the signature format and its protocol.


## Terminology

SignatureEnvelope `e` : The standard data structure for storing a signed message. Signature envelope consist of following components:

- Payload `m`: The data that is integrity protected - e.g. container image descriptor.
- Signed attributes `v`: The signature metadata that is integrity protected - e.g. signature expiration time, signing time, etc.
- Unsigned attributes `u`: The signature metadata that is not integrity protected - e.g. keyid, publickey, etc.
- Cryptographic signatures `s`: The digital signatures computed on payload and signed attributes.

A signature `s = Sign(sk, m, v)` where `sk` stands for signing key.
A signature envelope `e_n = { m, (v_0, u_0, s_0), ..., (v_{n-1}, u_{n-1}, s_{n-1})}` is a compact form of `n` signatures signing the same payload `m`.

![A complete signature in OCI registry](media/signature-structure.svg)


## Requirements

### Signature Envelope and Signature

1. The signature envelope payload MUST contain a [descriptor](https://github.com/opencontainers/image-spec/blob/master/descriptor.md#properties) of the target manifest.
    1. A descriptor MUST contain `mediaType`, `digest`, `size` fields.
    2. A descriptor MAY contain the `artifactType` field for artifact manifests, or the `config.mediaType` for `oci.image` based manifests.
    3. descriptor MAY contain a `annotations` field.
2. The signature envelope MUST be self contained to validate the integrity of the payload and signed attributes.
3. The signature envelope MAY contain arbitrary user supplied information, as key value pairs. If included, it MUST be integrity protected.
4. The signature envelope MAY contain information to determine the signature expiration time. If included, it MUST be integrity protected.
5. The signature envelope MAY contain information to determine the signature revocation status. If included, it MUST be integrity protected.
6. The signature MUST be detached a signature, with respect to the target manifest being signed.
7.  The signature MUST be stored and transferred in a signature envelope.
8.  If the signature was produced by a X.509 certificate, the signature MUST contain a certificate chain of the signing certificate as an unsigned attribute. This MAY be a partial certificate chain.
9.  The signature MAY be timestamped by TSA, and this counter signature MUST be stored as an unsigned attribute.

### Signature Format

1. The signature format MUST be able to store payload.
2. The signature format MUST be able to store cryptographic signature.
3. The signature format MUST support referencing and in-lining of different signing identity types represented as keyIds or X.509 certificate chains.
4. The signature format MUST be able store unsigned signature attributes such as timestamp signature (TSA).
5. Signature format SHOULD be easy to implement using existing Go libraries (Docker CLI is implemented in Go).
6. The signature format MAY be able store signed attributes. In case signature format doesn't support signed attributes, they can be combined with the payload before signing. Since the payload contains the signature specific metadata, the signature format loses the ability to store multiple signatures.
7. The signature format MAY store multiple signatures generated using same or different cryptographic materials (signing identity types and signing algorithms).
8. The signature format SHOULD NOT indicate signing algorithm, if present it MUST be a signed attribute. (There are concerns on the signing algorithm not matching the verification key, see [CS-DRNT15-1 Section 3.3](https://github.com/theupdateframework/notary/blob/master/docs/resources/ncc_docker_notary_audit_2015_07_31.pdf), [A cross-protocol attack on the TLS protocol](https://doi.org/10.1145/2382196.2382206), [secure-systems-lab/dsse#35](https://github.com/secure-systems-lab/dsse/issues/35)).


## Recommendation

### Signature format

Recommendation is to **use JWS JSON Serialization** because of following reasons:

- Specifications are parts of [RFC](https://datatracker.ietf.org/doc/html/rfc7515#section-3.2) standards.
- Native support for certificate(s), key id, signed and unsigned attributes.
- Signing libraries available in Go: [square/go-jose](https://github.com/square/go-jose), [lestrrat-go/jwx](https://github.com/lestrrat-go/jwx), [golang-jwt/jwt](github.com/golang-jwt/jwt) library to produce/consume a JWS with compact serialization and convert it to/from JSON serialization by custom code.
- Security related concerns can be addressed with proper implementations.

Concerns with DSSE:

- No published RFC (or draft).
- New format that is used in few projects.
- Does not currently support TSA signature and certificate(s) chain as part of the envelope. See [secure-systems-lab/dsse#42](https://github.com/secure-systems-lab/dsse/issues/42),  [secure-systems-lab/dsse#33](https://github.com/secure-systems-lab/dsse/issues/33).
- Additional effort required to certify implementations with security groups of cloud service providers before adoption.
- Being a new format, additional effort is involved to get the implementations reviewed and certified by the compliance & security boards of cloud service providers before its usage. This is a part of component governance which is considered as a strict requirement to adopt technologies and implementations in cloud environments.

### Payload

Recommendation for payload is TBD.
