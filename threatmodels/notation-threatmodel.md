# Notation Threat Model

## Overview

Notation is a comprehensive tool for generating and verifying signatures associated with software artifacts. Its purpose is to validate the integrity and authenticity, accomplishing this through the implementation of fine-grained policies.

The following diagram illustrates the architecture and components.

![Notation system overview](../media/notation-system.svg)

- **User**: The actor could be a signer or a verifier. A signer interacts with Notation CLI to sign artifacts. A verifier interacts with Notation CLI to verify artifacts against signatures.
- **Artifact Builder**: The actor who is responsible for producing software artifacts using build systems. Notation supports signing and verification of software artifacts including but are not limited to container images, helm charts, Software Bill of Materials (SBOMs). These artifacts can be stored either in remote registries or locally on disk using [OCI image layout](https://github.com/opencontainers/image-spec/blob/v1.0.0/image-layout.md).
- **Execution Environment**: The execution environment includes the host and **File System** where notation CLI will be installed and executed. Notation works with a shared responsibility model which means users/organizations are responsible for securing the notation execution environment. Following are the various directories and files used by the Notation:
  - `config.json` file is used to store various configurations such as credential store information, etc.
  - `trustpolicy.json` file is used to store trust policy related data.
  - `truststore` directory is used to store various trusted certificates used for signature verification.
  - `localkeys` directory is used to store test keys.
  - `plugins` directory is used to store various plugins.
  
  Notation uses credential stores to securely store the registry credentials.
- **Notation Plugin**: An external component/binary that can integrate as one of the steps in Notation’s workflow for signature generation or verification, see [plugin spec](https://github.com/notaryproject/notaryproject/blob/v1.0.0-rc.2/specs/plugin-extensibility.md) for details.
- **Registry**: An OCI-compliant registry that stores OCI artifacts, like container images, Helm charts or other OCI artifacts. Registries are outside Notation trust boundary.
- **KMS**: A Key Management System or a signing service that manages the the certificate along with private key that used for signing artifacts. KMS are outside Notation trust boundary.
- **OCSP Responder**: The [Online Certificate Status Protocol (OCSP)](https://www.rfc-editor.org/rfc/rfc6960) is an Internet protocol used for obtaining the revocation status of an X.509 digital certificate. Notation uses OCSP to check the certificate's revocation status.

## Notation sign artifacts using remote key

### Data flow

A signer uses Notation to sign the artifacts and store signatures either locally on the disk or in a registry.

The following diagram illustrates Notation signing artifacts stored in registries with keys stored in KMS:

![Notation signs artifacts stored in registries](../media/notation-sign-remote.svg)

The following diagram illustrates Notation signing artifacts as OCI image layout with keys stored in KMS:

![Notation signs artifacts as OCI image layout in the filesystem](../media/notation-sign-local.svg)

As per the [Notary signature specification](../specs/signature-specification.md), the signature payload is a JSON document which requires the [OCI descriptor](https://github.com/opencontainers/image-spec/blob/v1.0.0/descriptor.md) of the target artifact. For artifacts stored in a registry, the descriptor info can be retrieved from [image manifest](https://github.com/opencontainers/image-spec/blob/v1.1.0-rc2/manifest.md) stored in the registry. For artifacts stored as an [OCI Image layout](https://github.com/opencontainers/image-spec/blob/v1.0.0/image-layout.md), the descriptor info can be retrieved by inspecting the `index.json` file of OCI Image layout.

### Threats and Mitigation

|   Title                      | STRIDE threat type     | Threat Status | Priority | Entity          | Description   | Mitigation   |
| ---------------------------- | ---------------------- | ------------- | -------- | --------------- | ------------- | ------------ |
| Compromised signing key         | Information disclosure | Mitigated     | High     | Signer          | Signing keys are compromised, and this may lead to arbitrary artifacts being signed by attackers | Users store the signing keys in hardware security module (HSM) and rotate them periodically. In case a signing key is compromised, users must revoke associated certificates.  |
| Inaccessible Registry           | Denial of Service      | Mitigated     | High     | Registry        | Registry is not able to service incoming requests or perform up to spec, thus users are unable to publish any artifacts | Notation supports signing artifacts stored locally on disk |
| Serving tampered artifacts      | Tampering              | Mitigated     | High     | Registry        | Data stored in registries are tampered, this may lead to malicious artifacts being signed | Notation encourages user to sign artifacts using digest (and discourages signing using tag by emitting warnings), or Notation recommends signing artifacts stored in local file system, and stores signatures locally or in the registry |
| Disclosure of signature info    | Information disclosure | Mitigated     | High     | Registry        | Man-in-the-middle attack | Notation interacts with trusted registries via TLS or signs artifacts stored in local file system, and then stores signatures locally |
| Tampered files                  | Tampering              | Not Mitigated     | High     | File System     | Files read by Notation are tampered, this may lead to failure of notation sign operation | Notation must be installed on secure and trusted infrastructure(shared responsibility with users)|
| Disclosure of files             | Information disclosure | Mitigated     | High     | File System     | Files are accessed by attackers, this may lead to disclosure of notation config files and credential files | Notation must be installed on secure and trusted infrastructure(shared responsibility mode). Notation stores the credentials securely using credential store. Notation interacts with notation plugin to access signing keys via remote KMS or a HSM device |
| Malicious plugin                | Tampering              | Mitigated     | High     | Notation Plugin | A malicious plugin is installed, this may lead to arbitrary code being executed by attackers | Verify the integrity and authenticity of the plugin before using the plugin |
| Using weak crypto algorithms during the signing | Repudiation         | Mitigated     | High     | Notation Plugin | Bypass of signature verification | Notation restricts the signing algorithm to the following [set](https://github.com/notaryproject/notaryproject/blob/main/specs/signature-specification.md#algorithm-selection) |
| Compromised Notation dependencies | Tampering              | Mitigated  | High     | Notation       | The dependencies that built into Notation binary was compromised, this may lead to arbitrary code being executed | Notation keeps dependencies up-to-date and adds new dependency after careful consideration and only if it's absolutely required. Always use static build instead of dynamic linking |

## Notation verify artifacts against signatures

### Data flow

A verifier uses Notation to verify artifacts against signatures stored in the file system or in a registry to ensure the authenticity and integrity of the artifacts before deploying the artifacts.

The following diagram illustrates Notation verifying artifacts stored in a registry:

![Notation verifies artifacts stored in registries](../media/notation-verify-remote.svg)

The following diagram illustrates Notation verifying artifacts stored in a registry with a verification plugin:

![Notation verifies artifacts stored in registries using verification plugin](../media/notation-verify-plugin.svg)

The following diagram illustrates Notation verifying artifacts stored as OCI image layout in the file system:

![Notation verifies artifacts as OCI image layout in the filesystem](../media/notation-verify-local.svg)

The certificates trusted by the verifier are stored in Notation trust store in the file system. Users can download the certificates from KMS or request CA vendor for CA certificates.

### Threats and Mitigation

|   Title                                                        | STRIDE threat type     | Threat Status | Priority | Entity          | Description   | Mitigation   |
| -------------------------------------------------------------- | ---------------------- | ------------- | -------- | --------------- | ------------- | ------------ |
| Inaccessible Registry                                          | Denial of Service      | Mitigated     | High     | Registry        | Registry is not able to service incoming requests or perform up to spec, thus users are unable to verify artifacts | Notation supports verification of signatures stored locally on disk|
| Serving tampered artifacts                                     | Tampering              | Mitigated     | High     | Registry        | Data stored in registries are tampered, this may lead to malicious artifacts being consumed | Notation verifies the digest specified by the user and the digest store in the signature are the same. Notation does support verification using tag but its highly discouraged by emitting a warning |
| Disclosure of signature info                                   | Information disclosure | Mitigated     | High     | Registry        | Man-in-the-middle attack | Notation interacts with trusted registries via TLS by default or Notation signs artifacts stored in local file system, and then stores signatures locally |
| Tampered files                                                 | Tampering              | Not Mitigated     | High     | File System      | Files read by Notation are tampered, this may lead to failure of notation verify operation or verification bypassed | Notation must be installed on secure and trusted infrastructure(shared responsibility mode) |
| Disclosure of files                                            | Information disclosure | Mitigated     | High     | File System      | Files are accessed by attackers, this may lead to disclosure of notation config files and credential files | Notation must be installed on secure and trusted infrastructure(shared responsibility mode). Notation stores the credentials securely using credential store |
| Malicious plugin                                               | Tampering              | Mitigated     | High     | Notation Plugin | A malicious plugin is installed, this may lead to arbitrary code being executed by attackers | Verify plugin package before installation or plugin is installed by authorized users |
| Compromised trust policies by the weakest CA                   | Tampering              | Mitigated     | High     | Notation        | An attacker can make the signature verification for any signatures succeed by exploiting a vulnerability in the weakest CA’s infrastructure that exists in the trust store | Notation supports scoping trust policies per a registry. That means, users can write their policies to scope a CA to only to the registry that uses that CA, hence not affecting signatures of other registries signed by other CAs |
| Malicious signature faking to be signed by a signing authority | Tampering      | Mitigated | High   | Notation     | Unlike notary.x509 signing scheme, trusted timestamps are not checked against RFC#3161 TSA servers for notary.x509.signingAuthority signing scheme. An attacker can use this and bypass trusted timestamp checks by crafting a signature that uses notary.x509 keys but with signingAuthority as the signing scheme. | To prevent this threat, notary.x509.signingAuthority signing scheme requires trusted roots to be present in a trust store type called signingAuthority as opposed to CA trust store type for notary.x509 signing scheme |
| Inaccessible OCSP Responder                                    | Denial of Service      | Not Mitigated | High     | OCSP Responder  | OCSP Responder is not able to service incoming requests or perform up to spec, thus users are unable to validate certificate revocation status | It cannot be mitigated, since revocation status should be retrieved from OCSP responder, which requires network access. Notation verification should fail if revocation check is configured as `enforced` and OCSP responder is inaccessible. Users can configure trust policy to log or skip revocation check if OCSP responder is not reliable. |
| Compromised Notation dependencies                              | Tampering              | Mitigated | High     | Notation       | The dependencies that built into Notation binary was compromised, this may lead to arbitrary code being executed | Notation keeps dependencies up-to-date and adds new dependency after careful consideration and only if it's absolutely required. Always use static build instead of dynamic linking |
| Rollback Attack                              | Tampering              | Mitigated | High     | Notation       | Attacker can exploit a compromised repository to return outdated vulnerable artifacts | Signer can employ short signature expiration periods (and periodically re-sign artifacts) or revoke outdated vulnerable artifacts |