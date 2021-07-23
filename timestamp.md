# Timestamping

Time Stamping Authorities (TSAs) defined by [RFC3161](https://tools.ietf.org/html/rfc3161) provide signed timestamp for a signature in order to prove that the signature was generated during the validity period of a certificate.


## Scenarios

With TSA, signature can be considered valid even if the signing certificate is expired. This technique is widely used by [Authenticode](https://docs.microsoft.com/windows/win32/seccrypto/time-stamping-authenticode-signatures) with [SignTool](https://docs.microsoft.com/windows/win32/seccrypto/signtool), [NuGet](https://docs.microsoft.com/nuget/create-packages/sign-a-package), [Adobe Acrobat](https://helpx.adobe.com/acrobat/using/certificate-based-signatures.html#add_a_timestamp_to_certificate_based_signatures), and many other industrial products.

In the world of artifacts, including container images, scenarios are
- Developers sign their artifacts or images with certificates.
    - The certificates MAY have a configurable expiry time.
- Developers publish their artifacts or images and at a later date stop maintaining them.
    - Content consumers SHOULD be able to verify signatures until they expire and use the artifacts or images.
- Attackers with compromised keys try to sign artifacts or images with timestamps before the key compromise event.


## Discussions

There are many public TSA servers [available](https://gist.github.com/Manouchehri/fd754e402d98430243455713efada710) on the Internet. The advantages of public timestamp servers are obvious that they are **public** and **free**.

However, those public TSAs also come with disadvantages:
- Require the Internet access for signing.
    - It is not really a disadvantage since public TSAs are online services. Devices in the air-gapped environment SHALL access the Internet for timestamp signing services.
- Out of the control of the signer.
    - External dependency.
    - Availability is not assured. No SLA on signing.
        - Not all TSAs are available in all regions.
        - Some regions may have high latency.
    - The certificates of TSAs MAY be revoked at any time without notices.
        - [The removal of trust of VeriSign broke .NET 5+ NuGet.](https://github.com/dotnet/announcements/issues/180) Thus Microsoft has to disable the package verification with a new release to unblock customers.

Implications of not using a signed timestamp for a signature:
- In the absence of additional timestamp signature, the signature is only considered valid till key expiry. This may limit the use of short lived keys.
- In case of key compromise it's not possible to revoke signatures from a point in time where the time of compromise is known, as an attacker can create signatures with any signature time (by changing local time).


## Requirements and Recommendation

The following requirements and recommendations use TSAs as a method to prolong the validity of the original artifact signature. However, it does not prevent attackers signing signature and declaring arbitrary signing date in a key compromise event.

1. An artifact signature SHOULD be time stamped with a timestamp signature.
2. When time stamping an artifact signature, the following process SHOULD be applied:
    1. The artifact signature SHOULD be hashed and sent to TSA for time stamping.
    2. After timestamp signature retrieval, the artifact signature and the timestamp signature SHOULD be signed again by the artifact signer as a countersignature.
3. Clients MUST verify the artifact signature first.
4. If the artifact signature is expired and the timestamp signature with the countersignature exist, clients MAY verify those signatures and use the timestamp to verify the artifact signature again.
5. Timestamp signers MUST have a valid certificate with the `id-kp-timeStamping` purpose
6. Publishers MAY publish a list of TSAs they will use
   - The TSA list MUST NOT be defined in the signature
