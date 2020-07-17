Key management for container signing can be broadly categorized into three general use cases:
- Key Setup/Signing (How are keys set up? How are keys accessed by Notary v2 client when signatures are generated?)
- Trust store configuration (How do runtime environments determine which keys to trust?)
- Revocation (How do key owners create/revoke keys? How do runtime environments get this information?)

Personas:
- Publisher: User who builds containers.
- Deployer: User who deploys containers.

Signing Use Cases:
- Local Key Store
    - Publisher creates new root key on local machine. Keys are stored in key store or plain text on local machine (Need to identify keystores we want to support. Should we preclude signing with plain text keys to improve security posture?).
    - Publisher uploads root public key to registry (registry can verify user for container uploads). 
    - Publisher shares root public key (other users can verify containers before running).
    - Publisher creates new delegate keys on local machine that chain to root key.
    - Publisher runs docker build and notary v2 sign on local machine. Notary v2 client generates signatures with key in memory.
- USB Token Key Store
    - Publisher creates new root key on USB token. 
    - Publisher uploads root public key to registry (registry can verify user for container uploads). 
    - Publisher shares root public key (other users can verify containers before running).
    - Publisher creates new delegate keys on USB token that chain to root key.
    - Publisher runs docker build and notary v2 sign on local machine. Notary v2 client generates signatures with key in USB Token.
- Network HSM
    - Publisher creates new root key on Network HSM. 
    - Publisher uploads root public key to registry (registry can verify user for container uploads). 
    - Publisher shares root public key (other users can verify containers before running).
    - Publisher creates new delegate keys on Network HSM that chain to root key.
    - Publisher runs docker build and notary v2 sign on local machine. Notary v2 client generates signatures with key in Network HSM.
- Cloud Key Management Service
    - Publisher creates new root key on Cloud Key Management Service 
    - Publisher uploads root public key to registry (registry can verify user for container uploads). 
    - Publisher shares root public key (other users can verify containers before running).
    - Publisher creates new delegate keys on Cloud Key Management Service that chain to root key.
    - Publisher runs docker build and notary v2 sign on local machine. Notary v2 client generates signatures with key in Cloud Key Management Service.
- Hybrid
    - Publisher creates new root key on USB Token/Network HSM/Cloud Key Management Service 
    - Publisher uploads root public key to registry (registry can verify user for container uploads). 
    - Publisher shares root public key (other users can verify containers before running).
    - Publisher creates new delegate keys on local machine that chain to root key.
    - Publisher runs docker build and notary v2 sign on local machine. Notary v2 client generates signatures with keys on local machine.
- Signing by Build Machines
    - Publisher creates new root, uploads root public key to registry, and shares root public key.
    - Publisher creates new delegate keys on build machine or on USB Token/Network HSM/Cloud Key Management Service
    - Publisher configures build scripts with location of delegate keys.
    - Build machine runs docker build and then notary v2 sign with keys in configured location.
- Signing by Dedicated Machines
    - Publisher creates new root, uploads root public key to registry, and shares root public key.
    - Publisher creates new delegate keys on build machine or on USB Token/Network HSM/Cloud Key Management Service
    - Publisher configures signing machine with location of delegate keys.
    - Build machine runs docker build and passes objects to be signed to signing host.
    - Signing host generates signatures with keys in configured location.

Trust Store Configuration:
- Specify trusted public root keys
    - Deployer gets root public key from publisher.
    - Deployer adds root public key to runtime configuration.
    - Container pulled from any registry is be validated with listed root public keys before execution.
- Specify location of trusted public root keys
    - [TODO]
- Enterprises can restrict users to add/remove their root public keys to the trust store used across a fleet of runtime hosts. The trust store will validate images pulled from any registry.
- Enterprises can restrict users to add/remove root public keys for trusted third parties to their trust stores. The trust store will validate images pulled from any registry.
- [TBD] Do we envision a scenario where the trust store will need to define subordinate keys in addition to the root?
- Air gapped environments

Key rotation/revocation use cases:
- Root Revocation
    - Publisher deletes revoked root public key on registry (registry can stop sharing containers with revoked key).
    - Registry stops vending containers signed with old root key.
    - Publisher removes revoked root public key from shared list (other users stop trusting artifacts with revoked key).
    - Publisher creates new root key.
    - Publisher uploads new root public key to registry (registry can verify user for container uploads). 
    - Publisher updates shared root public key (other users can verify containers with new key before running).
    - Publisher creates new delegate keys.
- Delegate Key Revocation
    - Publisher lists revoked delegate keys as untrusted. Revocation list is signed by root key.
    - Publisher shares revocation list.
    - Publisher creates new delegate keys from existing root.
- Root Rotation
    - Publisher creates new root key.
    - Publisher uploads new root public key to registry (registry can verify user for container uploads). 
    - Publisher updates shared root public key (other users can verify containers with new key before running).
    - Publisher creates new delegate keys.
- Delegate Key Rotation
    - Publisher creates new delegate keys from existing root.

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
