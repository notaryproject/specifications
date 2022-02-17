# Signing and Verification Workflow

This document describes how Notary v2 signs and verifies OCI artifacts.

## Signing workflow

The user wants to sign an OCI artifact and push the signature to a repository.

### Signing Prerequisites

- User has access to the signing certificate and private key or a remote signing service through a notation plug-in.

### Signing Steps

1. **Generate signature:** Using notation CLI or any other compliant signing tool, sign the OCI artifact. The signing tool should follow the following guideline.
    1. Verify that the signing certificate is valid and satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
    1. Verify that the signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
    1. Generate signature.
        1. Generate signature using signature formats specified in [supported signature envelopes](./signature-specification.md#supported-signature-envelopes).
        1. If the user wants to timestamp the signature, obtain an [RFC-3161](https://datatracker.ietf.org/doc/html/rfc3161.html) compliant timestamp for the signature generated in the previous step.
           Otherwise, continue to the next step.
            1. Verify that the timestamp signing certificate satisfies [certificate requirements](./signature-specification.md#certificate-requirements).
            1. Verify that the timestamp signing algorithm satisfies [algorithm requirements](./signature-specification.md#signature-algorithm-requirements).
        1. Embed timestamp to the signature envelope.
1. **Push the signature envelope:** Push the signature envelope generated in the previous step to the repository.
1. **Generate signature artifact manifest:** As decribed in [signature specification](./signature-specification.md#storage) create the Notary v2 signature artifact manifest for the signature envelope generated in step 1.
1. **Push signature artifact manifest:** Push Notary v2 signature artifact manifest to the repository.

The user pushes the OCI artifact to the repository before the signature generation process as the signature reference must exist for the signature push to succeed.
See: [ORAS Artifact Push Validation](https://github.com/oras-project/artifacts-spec/blob/main/artifact-manifest.md#push-validation) for more info.

## Verification workflow

The user wants to pull an OCI artifact only if they are signed by a trusted publisher and the signature is valid.

### Verification Prerequisites

- User has a fully qualified reference to an OCI artifact they want to pull.
  If the fully qualified artifact reference contains a tag then the user needs to resolve this tag to a digest.
  E.g. If a fully qualified reference is `wabbit-networks.io/software:latest` where `latest` is a tag pointing to an artifact.
  The user must resolve the `latest` tag to a digest and construct a new artifact reference using the resolved digest `wabbit-networks.io/software@sha256:${digest}`.
- User has configured [trust store and trust policy](./trust-store-trust-policy-specification.md) required for signature verification.

### Verification Steps

1. **Should Notary v2 verify the signature? :** Depending upon [trust-policy](./trust-store-trust-policy-specification.md#trust-policy) configuration, determine whether Notary v2 needs to verify the signature or not.
   If signature verification should be skipped for the given artifact, skip the below steps and directly jump to step 4.
1. **Get signature artifact descriptors:** Using the ORAS [Manifest Referrers API](https://github.com/oras-project/artifacts-spec/blob/main/manifest-referrers-api.md) download the Notary v2 signature artifact descriptors.
   The  `artifactType` parameter is set to Notary v2 signature's artifact type `application/vnd.cncf.notary.v2.signature`.
   Note that all ORAS implementations may not support filtering using `artifactType`.
1. For each signature artifact descriptor, perform the following steps:
    1. **Get signature artifact manifest:** Download the Notary v2 signature's artifact manifest for the given artifact descriptor.
    1. **Filter signature artifact manifest:**
        1. Filter out the unsupported signature formats by comparing the signature envelope format type (`[descriptors].descriptor.mediaType`) in the signature manifest, with the supported formats defined in [signature specification](./signature-specification.md#storage).
        1. Depending upon the trust-store and trust-policy configuration, further filter out signature manifests.
            1. Using the `scopes` configured in trust policies, get the applicable trust policy.
            1. Get the list of trusted certificates from the trust stores specified in the applicable trust policy.
               If the trust policy contains multiple trust stores, create a list of trusted certificates by merging the trusted certificate list of each trust store.
                1. Calculate the SHA-256 fingerprint of all the trusted certificates and compare them against the list of SHA-256 certificate fingerprints present in  `org.cncf.notary.x509certs.fingerprint.sha256` annotation of artifact manifest.
                1. If there is at least one match, continue to the next step.
                   Otherwise, move to the next signature artifact descriptor(step 3.1).
                   If all signature artifact descriptors have already been processed, fail the signature verification and exit.
        1. If the artifact manifest is filtered out, skip the below steps and move to the next signature artifact descriptor(step 3.1).
           If all signature artifact descriptors have already been processed, fail the signature verification and exit.
    1. **Get and verify signatures:** On the filtered Notary v2 signature artifact manifest, perform the following steps:
        1. Download the signature envelope.
        1. Verify the signature envelope using trust-store and trust-policy as mentioned in [signature evaluation](./trust-store-trust-policy-specification.md#signature-evaluation) section.
        1. If the signature verification fails, skip the below steps and move to the next signature artifact descriptor(step 3.1).
           If all signature artifact descriptors have already been processed, fail the signature verification and exit.
        1. If signature verification succeeds, compare the digest derived from the given OCI artifact reference with the signed digest present in the signature envelope's payload.
           If digests are equal, signature verification is considered successful.
           Otherwise, move to the next signature artifact descriptor(step 3.1).
           If all signature artifact descriptors have already been processed, fail the signature verification and exit.
1. **Get OCI artifact:** Using the verified digest, download the OCI artifact.
   This step is not in the purview of Notary v2.
