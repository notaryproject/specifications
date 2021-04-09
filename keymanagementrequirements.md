# Overview
Key management for container signing can be broadly categorized into three general use cases:
- Key Setup/Signing (Key managment and integrations with client tooling for generating signatures.)
- Trust store configuration (Configuring which keys to trust for artifacts from a predefined source.)
- Signature Validity (Expiry, Revocation and/or Trusted Update List. Mechanism to update information on the validity of signatures.)

## Personas:
- Publisher: User who builds and signs containers.
    - Publisher Admin: In some scenarios a publisher will include a group of users (i.e. teams or enterprises). Admin users (i.e. security administrator) will be responsible for configuring roots, provide access to use or generate delegate keys, and make decisions for key revocation.
    - Publisher Builder: In some scenarios a publisher will include a group of users (i.e. teams or enterprises). Builders (i.e. developers, build hosts) will be responsible for creating artifacts. 
    - Publisher Signer: In some scenarios a publisher will include a group of users (i.e. teams or enterprises). Signers (i.e. developers, signing hosts) will be responsible for signing artifacts.
- Deployer: User who deploys containers.
    - Deployer Admin: In some scenarios a deployer will include a group of users (i.e. teams or enterprises). Admin users (i.e. security administrators) will be responsible for configuring deployment policies, including potentially a set of trusted roots or required signatures. 
    - Deployer Operator: In some scenarios a publisher will include a group of users (i.e. teams or enterprises). Operators (i.e. developers, deployment hosts) will be responsible for pulling container images from registries, verifying their signatures, and then running them.
- Repository Owner: In some scenarios a container may be stored in a repository that is managed by the repository owner. The repository may be public or private and could also be air-gapped.

## Definitions:
- Root key: A self signed key used for the lowest designation of trust. Root keys can be created by developers, organizations, public/private CAs, and registry operators. The root key should be retrieved from a trusted source that can establish the authenticity of the creator's identity.
- Signing key: A signing key is used to generate artifact signatures. A signing key should be signed with a root key or one of its intermediaries. The certificate chain with a signing key can be used to verify which root it belongs to. While a root key can be used as a signing key, this is not recommended as it creates a large blast radius and increases the risk of compromising a root key. 
- Trust Store: The trust store defines the relationship between signing keys and artifacts that are used at validation time to determine whether to trust an artifact with a crytptographically valid signature. The trust store will relate a source (any source, specific registry, specific repository, or specific target) with a certificate (for root key, intermediate, or signing key) or key repository (for automated key distribtuion). It is not recommended to use a signing key as this will cause signature validation to fail if the signing key is rotated. 
- Trust Store Key Information: The key information configured in the trust store will be cryptographically verifiable and contain:
    - Public Key: Will be used to verify signed artifacts.
    - Signed Identity: Identity of creator of the signing key.
    - (Optional) Issuer: If the key is an intermediate this will describe the owner of the root.
    - Mechanism to update validity of signatures generated with the signing key.

## Requirements
- Signing an artifact MUST NOT require the publisher to perform additional actions with a registry/repository or registry operator. Uploading a signature MAY require additional actions.
- Retrieving a signature MUST NOT require the deployer to perform additional actions with a registry/repository or registry operator beyond those required to pull an unsigned artifact.
- Artifact integrity, source, and signature expiry MUST be verifiable from the signature AND NOT require additional calls to the registry/repository.
- Signature allow list/deny list MAY require additional calls that will be defined in a separate document.
- Moving an artifact from one repository to another SHOULD NOT invalidate the signature on the artifact.
- Publishers SHOULD be able to sign with keys stored on their local machines, secure tokens, Hardware Security Modules (HSMs), or cloud based Key Management Services.
- Publishers SHOULD be able to generate multiple signatures for a single artifact.
- Publisher admins MUST have a mechanism to revoke signatures to indicate they are no longer trusted. 
- Trust stores MUST be configurable by the deployer admin.
- Deployer admins MUST be able to configure trusted entities for individual repositories and targets.
- Deployers MUST be able to validate signatures on any version of an artifact including whether they have been revoked by the publisher.
- Signature validation MUST be enforceable in air-gapped environments. 

## Requirements that need further discussion
### Signing Key Expiry
In public key cryptography, a pair of private key a.k.a. signing key and public key a.k.a. verification key is generated, and later the public key is certified by a . The cerificate expiry time does not imply the expiry time of the key pair since the public key can be certified by a CA for multiple times. Moreover, a new key pair is always generated before generating a new certificate in the real world. Therefore, we can consider the key expiry is equivilant to certificate expiry.

### External Timestamp Server
Time Stamping Authorities (TSAs) defined by RFC3161 provide signed timestamp for a signature in order to prove that the signature was generated during the validity period of a certificate.

#### Scenarios
With TSA, signature can be considered valid even if the signing cerificate is expired. This technique is widely used by Authenticode with SignTool, NuGet, Adobe Acrobat, and many other industrial products.

In the world of artifacts, including container images, scenarios are
- Developers sign their artifacts or images with certificates. The certificates MAY have a configurable expiry time.
- Developers publish their artifacts or images and at a later date stop maintaining them. Content consumers SHOULD be able to verify signatures until they expire and use the artifacts or images
- Attackers with compromised keys try to sign artifacts or images with timestamps before the key compromise event.

#### Pros and Cons

There are many public TSA servers available on the Internet. The advantages of public timestamp servers are obvious:
    Public
    Free

However, those public TSAs also come with disadvantages:
    Require Internet access for signing
        It is not really a disadvantage since public TSAs are online services. Devices in the air-gapped environment SHALL access the Internet for timestamp signing services.
    Out of the control of the signer
        External dependency
        Availability is not assured. No SLA on signing.
            Not all TSAs are available in all regions.
            Some regions may have high latency.
        The certificates of TSAs MAY be revoked at any time without notices.
            The removal of trust of VeriSign broke .NET 5+ NuGet. Thus Microsoft has to disable the package verification with a new release to unblock customers.

Implications of not using a signed timestamp for a signature
    In the absense of additional timestamp signature, the signature is only considered valid till key expiry. This may limit the use of short lived keys.
    In case of key compromise it’s not possible to revoke signatures from a point in time where the time of compromise is known, as an attacker can create signatures with any signature time (by changing local time)

### Signature Expiry

Signatures can expire if

    The certificate of the signing key or the timestamp key (whichever is longer) expires
    The signed content indicates a expiry time

In this section, the #2 scenario is focused.
Scenarios

The main scenario for using a configurable signature expiry is for the publisher to indicate how long they plan to maintain the signature on the artifact for. For example if a publisher signs an artifact with an expiry for 6 months, they are indicating that if within 6 months they identfy a reason that deployers should no longer trust the artifact they will rescind the signature.

An added benefit is to defend against freeze attack. Since the signature expires in a short time although the signing key can live longer, the verifiers have to obtain the latest signature to verify, which implies that the verifies have access to the latest content.

    Freeze Attack Similar to a replay attack, a freeze attack works by providing metadata that is not current. However, in a freeze attack, the attacker freezes the information a client sees at the current point in time to prevent the client from seeing updates, rather than providing the client older versions than the client has already seen. As with replay attacks, the attacker’s goal is ultimately to compromise a client who has vulnerable versions of packages installed. A freeze attack may be used to prevent updates in addition to having an installed package be out of date.

### Transparent Root key auto rotation
Root keys form the basis of hierarchical trust systems, where intermediate and leaf keys chain back to a root key, and leaf keys are used to generate signatures. Consumers of signed artifacts associate a baseline level of trust with signatures that chain to roots they trust. In the scenario that a root key is no longer trusted, all signed artifacts chaining back to the root are no longer trusted. The conditions which render the root keys to be no longer trusted, are disclosed to the customer through some other channel (CVE, internal security audit, etc.). Safe rotation of keys require a overlap period where both new and old root keys are trusted. This is always not possible, such as when a root key is compromised.

#### Scenarios
1. A public root associated with a key a Publisher uses to sign artifacts is compromised. The access to root key is compromised, rather than the root key itself being stolen, allowing an attacker to create new intermediate and leaf trusted keys.
2. An internal root used by an organization is due to expire. A new root is created and rotated before the existing root expires.
3. An internal root used by an organization is not stored securely and the private key is lost. A new root is created to continue operations.
4. A cryptographic algorithm is deprecated, or a compliance standard requires a customer to upgrade their key strength. The customer wants to safely rotate a root key without rendering existing signatures invalid and disrupting operations.

### Rescinding Signature Validity
Artifact Publishers need mechanisms to indicate that a signature they generated is still trustworthy. 

#### Scenarios
- A Publisher vends a new version of an artifact, and wants to indicate the new version as trusted in addition to older versions already published.

#### Discussion Areas
- Code signing is a mechanism for Publishers to explicitly indicate trust which is implicitly trusted by Consumer given some conditions are satisfied (based on trust policy, signature being valid and unrevoked)
    - An explicit allowlist is not required
    - A denylist is required in addition to indicate which artifacts are no longer trusted

- Signature Allowlist explicitly specifies the list of trusted artifacts.
    - Pros
        - Anything not in the list is implicitly untrusted, no separate revocation mechanism is required
        - If signed allowlist are used, the artifacts themselves may not need to be signed.
    - Cons
        - The allowlist needs to be updated for every version of artifact being published
        - May need to maintain a large allowlist which may be an overhead to distribute
- Signature Denylist
    - For Consumers, denylist allows explicitly indicating that an artifact or dependency is untrusted
    - For Publishers, this allows communicating to Consumers that specific versions of artifact are untrusted.

- Centralized/local public/private lists
    - The list can be local to repository, requiring update to the list in each repository, and requires keeping track of all repositories where the artifact needs to be published/revoked.
    - The list can be centralized
        - Centralized deny list (e.g. CRL maintaned by public CAs) can be used. The endpoint infomation is included in the signature, and signature verification step checks against this list.
            - Customers can define network topology with restricted network access, where these endpoints may not be accesible for hosts where signature verification occurs.
            - These endpoints may not be available in air-gapped environments.
    - Public lists (e.g. transparency logs) may not be suitable for enterprise customers who don’t want their artifact updates to be disclosed publicly.


## Prototype Stages
1. Signature generation for each of the key storage scenarios. A succesful prototype should enable signing with keys on each of the following: local host, secure tokens, Hardware Security Modules (HSMs), and cloud based Key Management Services.
2. Trust store configuration and signature source validation in runtime environments. A succesful prototype should enable configuring a trust store in a runtime environment. The prototype should validate signatures from a trusted source and reject signatures from a source that is not listed. The prototype will not check for other aspects of signature validity (expiry/revocation).
3. Key rotation. A succesful prototype should meet all requirements from the key rotation document.
4. Signature/Key Expiration. A succesful prototype should meet all requirements from the signature/key expiry document.
5. Signature allowlist/denylist. A succesful prototype should meet all requirements from the signatur allowlist/denylist document.
