# Notary Project Specifications

This repositories is in active maintenance and contains specifications shared across repositories under Notary Project as well as used by other open source projects and/or vendor tools that want to interoperate with Notary Project tooling.

Please see the Notary Project [README](https://github.com/notaryproject/.github/blob/main/README.md) file to learn about overall Notary Project.

In this README

- [Folder Structure](#folder-structure)
- [Requirements](#requirements)
- [Security Documents](#security-documents)
- [Specifications](#specifications)
- [Threat Models](#threat-models)
- [Community](#community)

## Folder Structure

| Folder Name   | Description  |
| --------------| -------------|
| [media](./media/)    | Media files referenced by documents in this repository |
| [requirements](./requirements/) | Requirements for Notary Project |
| [security](./security/) | Notary Project security related documents and reports |
| [specs](./specs/) | Notary Project specifications |
| [status-updates](./status-updates/) | This folder is not in active maintenance and contains status updates report for Notary Project |
| [threatmodels](./threatmodels/) | Threat models for repositories under Notary Project |

## Requirements

| File Name     | Description |
| -------- | ----------- |
| [definitions-terms.md](./requirements/definitions-terms.md) | A collection of definitions and terms used within this repository |
| [key-revocation.md](./requirements/key-revocation.md) | Requirements and proposals for key revocation |
| [keymanagementrequirements.md](./requirements/keymanagementrequirements.md) | Requirements for key management |
| [requirements.md](./requirements/requirements.md) | A collection of requirements and scenarios for Notary Project |
| [scenarios.md](./requirements/scenarios.md) | Notary Project signing scenarios |
| [verification-by-reference.md](./requirements/verification-by-reference.md) | Requirement of verification by reference |

## Security Documents

| File Name     | Description |
| -------- | ----------- |
| [ADA-notation-security-audit-23.pdf](./security/reports/audit/ADA-notation-security-audit-23.pdf) | Security audit report in 2023 covering [notation](https://github.com/notaryproject/notation), [notation-go](https://github.com/notaryproject/notation-go), and [notation-core-go](https://github.com/notaryproject/notation-core-go) repositories |
| [ADA-fuzzing-audit-22-23.pdf](./security/reports/fuzzing/ADA-fuzzing-audit-22-23.pdf) | Fuzz testing audit in 2023 covering [notary](https://github.com/notaryproject/notaty), [notation-go](https://github.com/notaryproject/notation-go), and [notation-core-go](https://github.com/notaryproject/notation-core-go) repositories |

## Specifications

| File Name     | Description |
| -------- | ----------- |
| [plugin-extensibility.md](./specs/plugin-extensibility.md) | Notation Plugin specification |
| [signature-envelope-cose.md](./specs/signature-envelope-cose.md) | Notary Project OCI COSE signature envelope |
| [signature-envelope-jws.md](./specs/signature-envelope-jws.md) | Notary Project OCI JWS signature envelope |
| [signature-specification.md](./specs/signature-specification.md) | Notary Project OCI signature specification |
| [signing-and-verification-workflow.md](./specs/signing-and-verification-workflow.md) | Notary Project OCI signing and verification workflow |
| [signing-scheme.md](./specs/signing-scheme.md) | Notary Project signing scheme|
| [trust-store-trust-policy.md](./specs/trust-store-trust-policy.md) | Notation Trust Store and Trust Policy  |


## Threat Models

| File Name     | Description |
| -------- | ----------- |
| [notation-threatmodel.md](./threatmodels/notation-threatmodel.md) | Threat models for [Notation CLI](https://github.com/notaryproject/notation) |

## Community

If you have any questions about Notary Project or contributing, do not hesitate to file an issue on relevant repository or contact the Notary Project maintainers and community members via the following channels:
- Join the [Notary Project community slack channel](https://app.slack.com/client/T08PSQ7BQ/CQUH8U287/)
- Join the [Community meetings](https://notaryproject.dev/community/#community-meetings)