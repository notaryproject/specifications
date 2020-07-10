Key management for container signing can be broadly categorized into three general use cases:
- Signing (How are keys made available when Notary v2 signatures are generated?)
- Trust store configuration (How do runtime environments determine which keys to trust?)
- Key setup/revocation (How do key owners create/revoke keys? How do runtime environments get this information?)

Signing Use Cases:
- Developer runs docker build and notary v2 sign on local machine. Keys are stored in key store or plain text on local machine (Need to identify keystores we want to support. Should we preclude signing with plain text keys to improve security posture?).
- Developer runs docker build and notary v2 sign on local machine. Keys are stored in a USB Token.
- Developer runs docker build and notary v2 sign on local machine. Keys are stored in a network HSM.
- Developer runs docker build and notary v2 sign on local machine. Keys are stored in a Cloud Key Management Service.
- Automated build system runs docker build and notary v2 sign on build machine.
- Automated build system runs docker build on a build machine. Signing is done on a second machine that verifies the container meets developer's release criteria.

Trust Store Configuration:
- Developer adds/removes their root public keys to the trust store used on a runtime host. The trust store will validate images pulled from any registry.
- Developer adds/removes root public keys for third parties they trust to their trust stores. The trust store will validate images pulled from any registry.
- Enterprises can restrict users to add/remove their root public keys to the trust store used across a fleet of runtime hosts. The trust store will validate images pulled from any registry.
- Enterprises can restrict users to add/remove root public keys for trusted third parties to their trust stores. The trust store will validate images pulled from any registry.
- [TBD] Do we envision a scenario where the trust store will need to define subordinate keys in addition to the root?

Key setup/revocation use cases:
- Developer sets up a single root key and delegates additional keys from it.
- Developer revokes/rotates chained keys. The revocation/rotation information is automatically available to run time environments.
- Developer revokes root keys. The run time environment administrator needs to update their trust store.
- Run time environments can automatically pull revocation lists for keys in their trust store.
- Air gapped run time environments can periodically pull and cache revocation lists for keys in their trust store.


In addition we also need to consider the following for cryptographic security:
- Supported key types
- Supported signing algorithms

Identify use cases to clarify:
1. Where can keys be stored? What interfaces need to be supported?
2. Any limitations ok key types/sizes supported?
3. How will public keys (root of trust) be distributed?
4. How will key revocation information be distributed?
5a. What is the minimum number of keys needed to succesfully sign a container?
5b. What is the recommended number of keys to sign a container?
6. Any additional requirements for timestamping?
