# A Set of Notary v2 Requirements

In addition to the Notary v2 goals and supported scenarios, a set of requirements are identified enabling discussions across various design options.

## 1.0 Support for Vend/Cloud Key Management Solutions

Users will operate registries across multiple clouds and on-prem environments. Notary v2 must provide extensibility enabling key management to be provided by any one of many solution.

## 2.0 Immutable Tags and Digests

Artifact users must decide between immutable digest, which are long, not user-friendly for troubleshooting and do not allow break-glass servicing scenarios. Notary v2 must account for users wishing to sign tags, digests or a combination of both without fear of an updated tag may not be signed by a given entity.

## 3.0 Support for Ephemeral Clients

The initial pull of an artifact must support a newly allocated, ephemeral client. On the initial pull, when the client is discovered to need initialization, a secured process is initiated providing a secure discovery and pull of content.

## 4.0 Verification Signature Copies

Verification objects (signatures), must be capable of being copied within a registry and across registries.

## 5.0 Revocation

Keys must be capable of being revoked across public, private and air-gapped registries.

