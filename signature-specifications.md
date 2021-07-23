# Signature Specification

This document specifies the signature format and its protocol.

## Definition 

A complete signature $\Sigma = (m, \mu, \hat\mu, \sigma) \gets \mathbf{Sign}(sk, m)$ is a tuple of
- Signed payload $m$.
- Signed signature metadata $\mu$.
- Unsigned signature metadata $\hat\mu$.
- Cryptographic signature $\sigma$.

where $sk$ stands for signing key.

![A complete signature in OCI registry](media/signature-structure.svg)

A signature envelope $\varepsilon_n = \{ m, (\mu_0, \hat\mu_0, \sigma_0), \dots, (\mu_{n-1}, \hat\mu_{n-1}, \sigma_{n-1}) \}$ is a compact form of $n$ signatures signing the same payload $m$. It can be expanded to $\Sigma_i = (m, \mu_i, \hat\mu_i, \sigma_i),\ \forall i \in \mathbb{Z}_n$.

A complete signature $\Sigma$ can be viewed as a signature envelope $\varepsilon_1$.

## Requirements

Since payload is about the metadata of the artifact manifest to be signed and it is not specific to any signature scheme, the payload requirements are discussed separately.

### Payload

- The payload MUST be a byte sequence.
- The payload MUST contain a [descriptor](https://github.com/opencontainers/image-spec/blob/master/descriptor.md#properties) of the target manifest.
    - A descriptor MUST contain `mediaType`, `digest`, `size` fields.
    - A descriptor MAY contain a `annotations` field.
    - A descriptor MAY contain the `artifactType` field for artifact manifests, or the `config.mediaType` for `oci.image` based manifests.
- Artifact identity MAY be indicated in the payload.
    - When indicated, the verifier SHOULD verify the origin against the signer.

### Signature

- A signature MUST be stored or transferred in a signature envelope.
- A signature envelope SHOULD be encoded in a single file.
- A signature envelope MUST contain one and only one complete signature.
- Multiple signatures for a subject manifest MUST be represented as seperate signature manifest and signature envelope (blob) pairs.
- The signing algorithm MAY be indicated in the signed signature metadata.
    - There is a concern on the signing algorithm not matching the verification key.
        - See [CS-DRNT15-1 Section 3.3](https://github.com/theupdateframework/notary/blob/master/docs/resources/ncc_docker_notary_audit_2015_07_31.pdf).
        - Similar vulnerability is introduced in [A cross-protocol attack on the TLS protocol](https://doi.org/10.1145/2382196.2382206) by Mavrogiannopoulos et al.
        - Detailed discussions can be found at [secure-systems-lab/dsse#35](https://github.com/secure-systems-lab/dsse/issues/35).
- The signing timestamp MAY be in the signed signature metadata.
- The signature expiry time MAY be in the signed signature metadata.
- The signed signature metadata MAY contain information that allows verifiers to determine if the artifact is revoked.
    - The contents of this attribute are pending the outcome of artifact level revocation design.
- If the signature is timestamped by a TSA, the timestamp and the corresponding signatures MUST be stored in the unsigned signature metadata.
- If the signature was produced by a X.509 certificate, the signature MUST contain the complete certificate chain of the signing certificate as an unsigned attribute.
- The thumbprint of the signing certificate MAY be indicated in the unsigned signature metadata.

### Signature Format

- A signature format MUST be able to store the payload.
- A signature format MUST be able to store the cryptographic signature.
- A signature format MUST be able to store unsigned signature metadata.
    - _This requirement is essential for TSA._ To time stamp a signature, the original signature must be generated first and then sent to a TSA for time stamping. Once obtained the timestamp signature, it will be attached to the original signature.
- A signature format MAY be able to store signed signature metadata.
    - In case of a signature format candidate being unable to store signed signature metadata, the signed signature metadata can be combined with the payload before signing. Since the payload contains the signature specific metadata, the signature format loses the ability to store multiple signatures.
- Signature format MUST support referencing and inlining of different identity types such as key(s) and certificate(s).
- Signature format SHOULD be easy to implement using existing Go libraries(Docker CLIs are developed in Go).
- Signature format MAY be able to store multiple signatures generated using same or different cryptographicmaterials(identity types and signing algoritms).
- Signature format MAY support counter signature.
