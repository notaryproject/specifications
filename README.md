# The Notary Project Overview

The Notary Project aims to provide enterprise-grade solutions and cross-industry standards for securing software supply chain. The following are current repositories, aka sub-projects, under the Notary Project umbrella in an alphabetical order:

| Repository                                                               | Description                                                                                                                                             |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------|
| [.github](https://github.com/notaryproject/.github)                      | The repository that contains the Notary Project governance documents                                                                                    |
| [meeting-notes](https://github.com/notaryproject/meeting-notes)          | The repository that contains archived meeting notes                                                                                                     |
| [notary](https://github.com/notaryproject/notatry)                       | The repository that is an implementation of TUF that runs next to a container registry and adds the ability to sign and verify content in the registry  |
| [notaryproject](https://github.com/notaryproject/notatryproject)         | The repository that contains the Notary Project requirements and specifications                                                                         |
| [notaryproject.dev](https://github.com/notaryproject/notatryproject.dev) | The repository that contains documents and styles for the Notary Project website                                                                        | 
| [notation](https://github.com/notaryproject/notation)                    | The repository that is an implementation of the Notary Project specification and provides a CLI tooling to sign and verify artifacts                                                                                 |
| [notation-go](https://github.com/notaryproject/notation-go)              | The repository that is an implementation of the Notary Project specification and provides the library supports of signing and verifying artifacts                    |
| [notation-core-go](https://github.com/notaryproject/notation-core-go)    | The repository that is an implementation of the Notary Project signature specification and provides the library supports                                                                             |
| [roadmap](https://github.com/notaryproject/roadmap)                      | The repository that contains roadmap issues for the Notary Project                                                                                      |
| [tuf](https://github.com/notaryproject/tuf)                              | The repository that implements the full TUF specification in a registry native way                                                                      |

## Project Status

The Notary Project is now in active development. The latest release announcement will be published on [the Notary Project website](https://notaryproject.dev/blog/). The Notary community uses the [project board](https://github.com/orgs/notaryproject/projects/10) for project planning and status tracking. You can also use GitHub milestones to track the progress:

- [Notary](https://github.com/notaryproject/notary/milestones)
- [Notation CLI](https://github.com/notaryproject/notation/milestones)
- [Notation library notation-go](https://github.com/notaryproject/notation-go/milestones)
- [Notation library notation-core-go](https://github.com/notaryproject/notation-core-go/milestones)

> Note: The original intention of [roadmap repository](https://github.com/notaryproject/roadmap) is to maintain the roadmap issues for the Notary Project, but we are now moving these issues to correspondent repositories. We will finally archive roadmap repository.

## Security

The Notary Project has had several public security audits:

- [August 7, 2018 by Cure53](https://github.com/notaryproject/notary/blob/master/docs/resources/cure53_tuf_notary_audit_2018_08_07.pdf)) covering `TUF` and `Notary` repositories
- [July 31, 2015 by NCC](https://github.com/notaryproject/notary/blob/master/docs/resources/ncc_docker_notary_audit_2015_07_31.pdf) covering `Notary` repository
- [Mar 21, 2023 by ADA Logics](https://github.com/notaryproject/notaryproject/blob/main/security/reports/fuzzing/ADA-fuzzing-audit-22-23.pdf) fuzzing audit covering `Notary`, `notation-go` and `notation-core-go` repositories

![Notary v2 scenarios](./media/notary-e2e-scenarios.svg)

## About this repository

![Notary v2 dependent projects](./media/oss-project-sequence.svg)

- [Requirements](./requirements/): The Notary Project goals, scenarios, and requirements are stored in this folder.
- [Specification](./specs/): The Notary Project specifications are stored in this folder. You can develop your own implementation based on the specifications. The Notary Project specifications now includes:
  - The Notary Project signature specification
  - The Notary Project signature envelope specifications: JWS and COSE
  - The signing and verifying specifications
  - The trust store and trust policy specification
  - The plugin specification
- [Threat Model](./threatmodel.md): General threat modeling for the Notary Project. We are also working on specific threat model for repositories Notation CLI `notation`, Notation libraries `notation-go` and `notation-core-go`.

## Community

You can reach the Notary Project community and developers via the following channels:

- Slack: Join the [Notary Project community channel](https://app.slack.com/client/T08PSQ7BQ/CQUH8U287/) for discussion and ask questions
- Twitter: [@NotaryProject](https://mobile.twitter.com/NotaryProject)
- Meetings: Join the [Community meetings](https://notaryproject.dev/community/#community-meetings)
  - Active meeting notes are captured at the [Notary Project meeting notes](https://hackmd.io/_vrqBGAOSUC_VWvFzWruZw?view)
  - Archived meeting notes are stored at the [meeting-notes repository](https://github.com/notaryproject/meeting-notes)