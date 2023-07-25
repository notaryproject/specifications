# Requirements

A collection of requirements and scenarios, framing the scope of Notary Project.

## TOC

<!-- no toc -->
- [Goals](#goals)
- [Non Goals](#non-goals)
- [Scenarios](./scenarios.md)
- [Threat Model](./threatmodel.md)
- [Key Stake Holders & Contributors](#key-stake-holders--contributors)
- [Definitions & Terms](./definitions-terms.md)
- [Contributing & Conversations](#contributing--conversations)

## Goals

Notary Project aims to address the learnings and limitations of [TUF-based implementation](https://github.com/notaryproject/notary), while establishing and prioritizing a set of goals and [scenarios](./scenarios.md) for new implementation (this repository).

1. Offline signature creation
1. Signatures attesting to authenticity and/or certification
1. Maintain the original artifact digest and collection of associated tags, supporting existing dev through deployment workflows
1. Multiple signatures per artifact, enabling the originating vendor signature, public registry certification and user/environment signatures
1. Native persistance within an [OCI distribution specification v1.1][oci-distribution] compliant registry
1. Artifact and signature copying within and across [OCI Image][oci-image] enabled, [OCI distribution specification][oci-distribution] based registries
1. Support multi-tenant registries enabling cloud providers and enterprises to support managed services at scale
1. Support private registries, where public content may be copied to, and new content originated within
1. Air-gapped environments, where the originating registry of content is not accessible
1. Key hierarchies and delegation
1. Key revocation, including private and air-gapped registries
1. Key acquisition must support users from hobbyists, open source projects to large software vendors
1. Usable workflows, enabled for adopters to easily create and consume Notary Project signatures

## Non Goals

1. Trust on first use
1. Implicit permissions on rotated keys
1. Compatibility with [TUF-based implementation](https://github.com/notaryproject/notary)

## Key Stake Holders & Contributors

As we identify the requirements and constraints, a number of key contributors will be asked to represent their requirements and constraints.

> Please submit PRs for companies, projects, products that you believe should be included:

- Registry Cloud Operators
  - [Azure Container Registry (acr)][acr] - Steve Lasker <steve.lasker@microsoft.com> ([@stevelasker](http://github.com/stevelasker))
  - [Amazon Elastic Container Registry (ecr)][ecr] - Omar Paul <omarpaul@amazon.com>
  - [Docker Hub][docker-hub] - Justin Cormack justin.cormack@docker.com
  - [Google Container Registry (gcr)][gcr]
  - [GitHub Package Registry (gpr)][gpr]
  - [Quay][quay] - Hank Donnay <hdonnay@redhat.com>
  - [IBM Cloud Container Registry (icr)][icr]
- Registry Vendors, Projects & Products
  - [Mirantis Secure Registry (msr, formerly Docker Trusted Registry)][msr]
  - [Harbor][harbor]
  - [JFrog Artifactory][jfrog]
- Controllers, Runtimes & Engines
  - [Mirantis Container Runtime (formerly Docker Engine – Enterprise)][mirantis-runtime]
  - [Mirantis Kubernetes Engine (mke, formerly Docker Enterprise/UCP)][mke]
- Artifact Types
  - [OCI & Docker Container Images][image-spec]
  - [Helm Charts][helm-registry]
  - [Singularity][singularity]
  - Operator Bundles
  
## Contributing & Conversations

Regular conversations for Notary Project occur on the [Cloud Native Computing Slack](https://app.slack.com/client/T08PSQ7BQ/CQUH8U287?) channel.

Weekly meetings occur each Monday.
Please see the [CNCF Calendar](https://www.cncf.io/community/calendar/) for details.

Meeting notes are captured on [hackmd.io](https://hackmd.io/_vrqBGAOSUC_VWvFzWruZw).

[acr]:                  https://aka.ms/acr/artifacts
[artifacts-repo]:       https://github.com/opencontainers/artifacts
[docker-hub]:           https://hub.docker.com/
[msr]:                  https://www.mirantis.com/software/mirantis-secure-registry
[mirantis-runtime]:     https://www.mirantis.com/software/container-runtime/
[mke]:                  https://www.mirantis.com/software/mirantis-kubernetes-engine/
[ecr]:                  https://aws.amazon.com/ecr/
[gcr]:                  https://cloud.google.com/container-registry/
[gpr]:                  https://github.com/features/package-registry
[harbor]:               https://goharbor.io/
[icr]:                  https://icr.io/
[helm-registry]:        https://v3.helm.sh/docs/topics/registries/
[image-spec]:           https://github.com/opencontainers/image-spec
[jfrog]:                https://jfrog.com/integration/docker-registry/
[oci-distribution]:     https://github.com/opencontainers/distribution-spec
[oci-image]:            https://github.com/opencontainers/image-spec
[oci-index]:            https://github.com/opencontainers/image-spec/blob/master/image-index.md
[oci-manifest]:         https://github.com/opencontainers/image-spec/blob/master/manifest.md
[oci-tob]:              https://github.com/opencontainers/tob
[singularity]:          https://github.com/sylabs/singularity
[quay]:                 https://quay.io/
