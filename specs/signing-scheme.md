# Signing Scheme

Signatures are primarily used to provide a consumer of an artifact the following guarantees - integrity (the signed artifact was not tampered with after signing it), and authenticity (the artifact was indeed signed by the entity who claims to have signed it).
X.509 PKI based identities are commonly used to for signing artifacts.
It has well established standards for issuing and representing identities (CAs and certificates) which sign artifacts, mechanism to establish authenticity using trust stores, and support for managing key lifetimes and rotation, and revocation of identities.
There is interest in the community to support other ways to establish integrity and authenticity, based on other systems and techniques ([TUF-based implementation](https://github.com/notaryproject/notary), ledger based).
These approaches also can provide different feature sets than ones traditionally provided by standard X.509 PKI based signing.
E.g. TUF is geared for software update systems and provides a freshness guarantee.

Notary Project will initially support X.509 PKI identity based signing, but provide the flexibility for additional systems through an abstraction called Signing Scheme.
The Signing Scheme covers aspects of signature generation and verification, and formalizes the feature set (guarantees) provided by the signature produced using a signing scheme.
Generally it covers the following aspects, but can be extended to other aspects as required by newer systems.

*Signature creation*

1. Supported entities that can generate signatures - typically an end user generates the signature, other models can be supported using signature scheme.
1. Representation of identities - e.g. X.509 certificates are used to represent end users and entities which verify the end user’s authenticity (CAs).
1. Signature schema - Defines the signed and unsigned attributes to be included in a signature envelope, and which attributes are required, optional and critical.

*Signature verification*

1. Mechanism to establish trust in end users and other entities (like CAs).
1. Set of guarantees available to the verifier apart from integrity and authenticity.

## Signing Scheme

Notary Project currently defines the following Signing Schemes.

`notary.x509` - Defines a signing scheme that uses the traditional signing workflow in which an end user generates signatures using X.509 certificates.

`notary.x509.signingAuthority` - Defines a signing scheme in which a `signing authority` generates signatures on behalf of an end user (the signature requestor) using X.509 certificates.
A trusted signing authority is defined as a third party service that is trusted both by the end user (the signature requestor) and verifying entity to generate signatures.
Authorities trusted by users should have mechanisms for validating, logging, monitoring, and auditing access to their systems and publish incidence response plans.
A certificate authority (CA) will need to demonstrate only validated entities were issued an X.509 certificate that chains to their root, and a time stamp authority (TSA) will need to demonstrate the time-stamp signing keys were only used within their service.
Similarly, a signing authority (SA) will need to demonstrate signing keys were only used within their service and only validated entities were allowed to generate signatures using the service.

* A signature envelope can only specify one Signing Scheme
* When Notary Project supports an additional Signing Scheme
  * Existing signed artifacts MUST be re-signed if they need to be verified using the new signing scheme defined verification process.
  * Existing clients used by verifying entity MUST be updated to newer versions that support verifying signatures that use the new signing scheme, otherwise the signatures with newer signing schemes which are unknown to existing clients will fail signature verification. Signatures that use older signing schemes which are known to existing clients will continue to verify correctly.
  * The language of the [Notary Project verification policy](./trust-store-trust-policy.md) in *trustpolicy.json* MAY have breaking changes to support newer concepts/configuration elements introduced by the new signing scheme.
  The breaking changes are addressed by introducing new major version in the versioned *trustpolicy.json* .

## Signature Creation

The signature envelope contains `Signing Scheme` as an required and critical attribute.
This attribute dictates the rest of signature schema - the set of signed and unsigned attributes to be included in the signature.

Both `notary.x509` and `notary.x509.signingAuthority` signing schemes use the similar signature schema (set of signed and unsigned attributes) with the following differences.

* `notary.x509` MUST use a countersignature from trusted source to determine authentic signing time (timestamp).
This is supported through the Timestamp signature unsigned attribute in the signature envelope. Currently Notary Project uses a [RFC3161](ietf-rfc3161) compliant TSA signature for this purpose.

* `notary.x509.signingAuthority` MUST use a timestamp attribute that is generated by a signing service as part of the original signature itself to determine authenticated signing time (timestamp).
This is supported through the *Authentic Signing time* attribute in the signature envelope.

## Signature Verification

Signature verification requires that a `Signing Scheme` attribute is present in the signature and it’s treated as critical i.e. the attribute MUST be understood and processed by the verifier.
For the JWS signature format, the attribute name is `io.cncf.notary.signingScheme`, with supported values `notary.x509` and `notary.x509.signingAuthority`.
Any other value will fail signature verification i.e when Notary Project supports an additional Signing Scheme, clients (like *Notation CLI*) MUST be updated to a version that supports the new signing scheme.

### Trust Stores

Each *Signing Scheme* defines the set of trust store types (e.g. CA) that it uses for signature verification.

`notary.x509`

* Uses trusts store types Certificate Authority (CA) and Timestamping Authority (TSA) during signature verification.
The signature is verified against the trust store of type CA, and the *Timestamp signature* is verified against the trust store of type TSA (to determine the signing time).
* For signature verification to be successful
  * The verifying entity’s trust store MUST contain the trusted root certificates under named trust stores of type CA (`{CONFIG}/notation/truststore/x509/ca`) and TSA(`{CONFIG}/notation/truststore/x509/tsa`)
  * The named trust stores MUST be specified in *trustpolicy.json*. E.g. *trustPolicy.trustStores* with value of `ca:acme-rockets,tsa:acme-tsa`.

`notary.x509.signingAuthority`

* Uses trusts store type Signing Authority during signature verification.
The signature is verified against the trust store of type Signing Authority.
The signing time is determined using the *Authentic Signing time* attribute in the signature envelope, and does not rely on a separate TSA generated Timestamp signature.
* For signature verification to be successful
  * The verifying entity’s trust store MUST contain the trusted root certificates under named trust stores of type signingAuthority (`{CONFIG}/notation/truststore/x509/signingAuthority`).
  * The named trust stores MUST be specified in *trustpolicy.json*. E.g. *trustPolicy.trustStores* with value of `signingAuthority:foobar` .

## FAQ

*Q:* What is the relationship of Signing Scheme with Signature Envelope format?

*A:* Signing Scheme aims to be agnostic of the Signature Envelope format.
A given signing scheme can be implemented through any signature envelope format (such as JWS or COSE) as long as it can support the required signature schema used by the signing scheme.

*Q:* Why is the trust store used for Signing Authority (`x509/signingAuthority`) distinct from trust store for Certificate Authority (`x509/ca`), why can’t they share the same trust store?

*A:* Signing Authority is a different type of trusted entity as compared to Certificate Authority (CA) or Timestamping Authority (TSA).
A CA is trusted for verifying the identity of a signing entity (end user) and issuing it a certificate, whereas a TSA is trusted to generate authentic timestamp.
In contrast, an SA is trusted to generate signatures on behalf of an end user (signature requestor) and also to generate authentic timestamp as part of the signature.
If we use a shared trust store for CA and SA, a verifying entity does not have the ability to differentiate between CA and SA when the verifying entity configures trusted roots in the trust store.
This has implication such as an end user with CA issued certificate can masquerade themselves as an SA by generating a signature with signing scheme `notary.x509.signingAuthority`.

[ietf-rfc3161]: https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2
