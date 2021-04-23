# Overview
Key management for container signing can be broadly categorized into three general use cases:
- Key Setup/Signing (Key managment and integrations with client tooling for generating signatures.)
- Deployment configurations (Configuring trust policies and which keys to trust for artifacts.)
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
- Trust Policy: The trust policy defines whether to enable signature validation and which checks need to be run. An example trust policy:
```
            {
                "trustPolicy": {
                    "signatureCheck": true,
                    "optionalChecks": {
                        "signatureExpiry": true,
                        "signatureRescinded": false
                    },
                    "trustStore": [
                        "truststore.json",
                        {
                            "trustedRoots": [
                                {
                                    "root": "test.crt"
                                }
                            ]
                        }
                    ],
                    "trustedArtifacts": []
                }
            }
```
- Trust Store: The trust store defines the relationship between signing keys and artifacts that are used at validation time to determine whether to trust an artifact with a crytptographically valid signature. The trust store will relate a scope (any source, specific registry, specific repository, or specific target) with a certificate (for root key, intermediate, or signing key) or key repository (for automated key distribtuion). It is not recommended to use a signing key as this will cause signature validation to fail if the signing key is rotated. An example trustStore where the first root is trusted for artifacts from any registry and the second root is only trusted for artifacts from "registry.wabbit-networks.io" :
```
            {
                "trustedRoots": [
                    {
                        "root": "-----BEGIN CERTIFICATE-----
                            MIIDQTCCAimgAwIBAgITBmyfz5m/jAo54vB4ikPmljZbyjANBgkqhkiG9w0BAQsF
                            ADA5MQswCQYDVQQGEwJVUzEPMA0GA1UEChMGQW1hem9uMRkwFwYDVQQDExBBbWF6
                            b24gUm9vdCBDQSAxMB4XDTE1MDUyNjAwMDAwMFoXDTM4MDExNzAwMDAwMFowOTEL
                            MAkGA1UEBhMCVVMxDzANBgNVBAoTBkFtYXpvbjEZMBcGA1UEAxMQQW1hem9uIFJv
                            b3QgQ0EgMTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALJ4gHHKeNXj
                            ca9HgFB0fW7Y14h29Jlo91ghYPl0hAEvrAIthtOgQ3pOsqTQNroBvo3bSMgHFzZM
                            9O6II8c+6zf1tRn4SWiw3te5djgdYZ6k/oI2peVKVuRF4fn9tBb6dNqcmzU5L/qw
                            IFAGbHrQgLKm+a/sRxmPUDgH3KKHOVj4utWp+UhnMJbulHheb4mjUcAwhmahRWa6
                            VOujw5H5SNz/0egwLX0tdHA114gk957EWW67c4cX8jJGKLhD+rcdqsq08p8kDi1L
                            93FcXmn/6pUCyziKrlA4b9v7LWIbxcceVOF34GfID5yHI9Y/QCB/IIDEgEw+OyQm
                            jgSubJrIqg0CAwEAAaNCMEAwDwYDVR0TAQH/BAUwAwEB/zAOBgNVHQ8BAf8EBAMC
                            AYYwHQYDVR0OBBYEFIQYzIU07LwMlJQuCFmcx7IQTgoIMA0GCSqGSIb3DQEBCwUA
                            A4IBAQCY8jdaQZChGsV2USggNiMOruYou6r4lK5IpDB/G/wkjUu0yKGX9rbxenDI
                            U5PMCCjjmCXPI6T53iHTfIUJrU6adTrCC2qJeHZERxhlbI1Bjjt/msv0tadQ1wUs
                            N+gDS63pYaACbvXy8MWy7Vu33PqUXHeeE6V/Uq2V8viTO96LXFvKWlJbYK8U90vv
                            o/ufQJVtMVT8QtPHRh8jrdkPSHCa2XV4cdFyQzR1bldZwgJcJmApzyMZFo6IQ6XU
                            5MsI+yMRQ+hDKXJioaldXgjUkK642M4UwtBV8ob2xJNDd2ZhwLnoQdeXeGADbkpy
                            rqXRfboQnoZsG4q5WTP468SQvvG5
                            -----END CERTIFICATE-----"
                    },
                    {
                        "root": "-----BEGIN CERTIFICATE-----
                            MIIFQTCCAymgAwIBAgITBmyf0pY1hp8KD+WGePhbJruKNzANBgkqhkiG9w0BAQwF
                            ADA5MQswCQYDVQQGEwJVUzEPMA0GA1UEChMGQW1hem9uMRkwFwYDVQQDExBBbWF6
                            b24gUm9vdCBDQSAyMB4XDTE1MDUyNjAwMDAwMFoXDTQwMDUyNjAwMDAwMFowOTEL
                            MAkGA1UEBhMCVVMxDzANBgNVBAoTBkFtYXpvbjEZMBcGA1UEAxMQQW1hem9uIFJv
                            b3QgQ0EgMjCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAK2Wny2cSkxK
                            gXlRmeyKy2tgURO8TW0G/LAIjd0ZEGrHJgw12MBvIITplLGbhQPDW9tK6Mj4kHbZ
                            W0/jTOgGNk3Mmqw9DJArktQGGWCsN0R5hYGCrVo34A3MnaZMUnbqQ523BNFQ9lXg
                            1dKmSYXpN+nKfq5clU1Imj+uIFptiJXZNLhSGkOQsL9sBbm2eLfq0OQ6PBJTYv9K
                            8nu+NQWpEjTj82R0Yiw9AElaKP4yRLuH3WUnAnE72kr3H9rN9yFVkE8P7K6C4Z9r
                            2UXTu/Bfh+08LDmG2j/e7HJV63mjrdvdfLC6HM783k81ds8P+HgfajZRRidhW+me
                            z/CiVX18JYpvL7TFz4QuK/0NURBs+18bvBt+xa47mAExkv8LV/SasrlX6avvDXbR
                            8O70zoan4G7ptGmh32n2M8ZpLpcTnqWHsFcQgTfJU7O7f/aS0ZzQGPSSbtqDT6Zj
                            mUyl+17vIWR6IF9sZIUVyzfpYgwLKhbcAS4y2j5L9Z469hdAlO+ekQiG+r5jqFoz
                            7Mt0Q5X5bGlSNscpb/xVA1wf+5+9R+vnSUeVC06JIglJ4PVhHvG/LopyboBZ/1c6
                            +XUyo05f7O0oYtlNc/LMgRdg7c3r3NunysV+Ar3yVAhU/bQtCSwXVEqY0VThUWcI
                            0u1ufm8/0i2BWSlmy5A5lREedCf+3euvAgMBAAGjQjBAMA8GA1UdEwEB/wQFMAMB
                            Af8wDgYDVR0PAQH/BAQDAgGGMB0GA1UdDgQWBBSwDPBMMPQFWAJI/TPlUq9LhONm
                            UjANBgkqhkiG9w0BAQwFAAOCAgEAqqiAjw54o+Ci1M3m9Zh6O+oAA7CXDpO8Wqj2
                            LIxyh6mx/H9z/WNxeKWHWc8w4Q0QshNabYL1auaAn6AFC2jkR2vHat+2/XcycuUY
                            +gn0oJMsXdKMdYV2ZZAMA3m3MSNjrXiDCYZohMr/+c8mmpJ5581LxedhpxfL86kS
                            k5Nrp+gvU5LEYFiwzAJRGFuFjWJZY7attN6a+yb3ACfAXVU3dJnJUH/jWS5E4ywl
                            7uxMMne0nxrpS10gxdr9HIcWxkPo1LsmmkVwXqkLN1PiRnsn/eBG8om3zEK2yygm
                            btmlyTrIQRNg91CMFa6ybRoVGld45pIq2WWQgj9sAq+uEjonljYE1x2igGOpm/Hl
                            urR8FLBOybEfdF849lHqm/osohHUqS0nGkWxr7JOcQ3AWEbWaQbLU8uz/mtBzUF+
                            fUwPfHJ5elnNXkoOrJupmHN5fLT0zLm4BwyydFy4x2+IoZCn9Kr5v2c69BoVYh63
                            n749sSmvZ6ES8lgQGVMDMBu4Gon2nL2XA46jCfMdiyHxtN/kHNGfZQIG6lzWE7OE
                            76KlXIx3KadowGuuQNKotOrN8I1LOJwZmhsoVLiJkO/KdYE+HvJkJMcYr07/R54H
                            9jVlpNMKVv/1F2Rs76giJUmTtt8AF9pYfl3uxRuw0dFfIRDH+fO6AgonB8Xx1sfT
                            4PsJYGw=
                            -----END CERTIFICATE-----",
                        "scope": "registry.wabbit-networks.io"
                    }
                ]
            }
```
- Trust Store Key Information: The key information configured in the trust store will be cryptographically verifiable and contain:
    - Public Key: Will be used to verify signed artifacts.
    - Signed Identity: Identity of creator of the signing key.
    - (Optional) Issuer: If the key is an intermediate this will describe the owner of the root.
    - (Optional) Mechanism to check whether signatures generated with the signing key have been rescinded.

## Requirements
- Signing an artifact MUST NOT require the publisher to perform additional actions with a registry/repository or registry operator. Uploading a signature MAY require additional actions.
- Retrieving a signature MUST NOT require the deployer to perform additional actions with a registry/repository or registry operator beyond those required to pull an unsigned artifact.
- Artifact integrity, source, and signature expiry MUST be verifiable from the signature AND NOT require additional calls to the registry/repository.
- Signature allow list/deny list MAY require additional calls that will be defined in a separate document.
- Copying an artifact from one repository to another SHOULD NOT invalidate the signature on the artifact.
- Publishers SHOULD be able to sign with keys stored on their local machines, secure tokens, Hardware Security Modules (HSMs), or cloud based Key Management Services.
- Publishers SHOULD be able to generate multiple signatures for a single artifact.
- Publisher admins MUST have a mechanism to revoke signatures to indicate they are no longer trusted. 
- Trust stores MUST be configurable by the deployer admin.
- Deployer admins MUST be able to configure trusted entities for individual repositories and targets.
- Deployers MUST be able to validate signatures on any version of an artifact including whether they have been revoked by the publisher.
- Signature validation MUST be enforceable in air-gapped environments. 

## Requirements that need further discussion
### Signing Key Expiry - https://hackmd.io/n82ZTBv3TK2Y4rujlnW3ng
### External Timestamp Server - https://hackmd.io/WvoBFNg2TR-ooTe14a6pfw
### Signature Expiry - https://hackmd.io/pT4IicsQRlGhR9szlrIUWQ
### Trust Policy Management and Trust Store Updates - Trust Policy Management and Trust Store Updates
### Rescinding Signature Validity - https://hackmd.io/TnX8l31CQnGPRujZ_gLGJA
### Multiple Signatures 


## Prototype Stages
1. Signature generation for each of the key storage scenarios. A succesful prototype should enable signing with keys on each of the following: local host, secure tokens, Hardware Security Modules (HSMs), and cloud based Key Management Services.
2. Trust store configuration and signature source validation in runtime environments. A succesful prototype should enable configuring a trust store in a runtime environment. The prototype should validate signatures from a trusted source and reject signatures from a source that is not listed. The prototype will not check for other aspects of signature validity (expiry/revocation).
3. Key rotation. A succesful prototype should meet all requirements from the key rotation document.
4. Signature/Key Expiration. A succesful prototype should meet all requirements from the signature/key expiry document.
5. Signature allowlist/denylist. A succesful prototype should meet all requirements from the signatur allowlist/denylist document.
