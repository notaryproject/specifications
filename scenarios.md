# Notary Signing - Scenarios

As containers and cloud native artifacts become the common unit of deployment, users want to know the artifacts in their environments are authentic and unmodified. 

These Notary v2 scenarios define end-to-end scenarios for signing artifacts in a generalized way, storing2 and moving them between OCI compliant registries, and validating them by various artifact hosts and tooling. 

By developing a generalized solution, artifact authors may develop their unique artifact types, allowing them to leverage Notary for signing and OCI Compliant registries for distribution.

## OCI Images & Artifacts

The [OCI TOB][oci-tob] has adopted [OCI Artifacts][artifacts-repo], generalizing container images as one of many types of artifacts that may be stored in a registry. Other artifact types currently include:

* [Helm Charts][helm-registry]
* [Singularity][singularity]
* Car firmware updates, deployed from OCI Artifact registries

## Goals

This document serves as the requirements and constraints of a generalized signing solution. It focuses on the scenarios and needs, and very specifically avoids any reference to other projects or implementations. As our working group forms a consensus on the requirements, the group will then transition to a spec.

## Non-Goals

- Notary v2 does not account for what the content represents or its lineage. Other efforts may attach additional content, and re-sign the super set of content to account for other scenarios. 

## Key Stake Holders & Contributors

As we identify the requirements and constraints, a number of key contributors will be asked to represent their requirements and constraints.

> Please add companies, projects, products that you believe should be included.

* Registry Cloud Operators
  * [Azure Container Registry (acr)][acr] - Steve Lasker <steve.lasker@microsoft.com> ([@stevelasker](http://github.com/stevelasker))
  * [Amazon Elastic Container Registry (ecr)][ecr] - Omar Paul <omarpaul@amazon.com>
  * [Docker Hub][docker-hub] - Justin Cormack justin.cormack@docker.com
  * [Google Container Registry (gcr)][gcr]
  * [GitHub Package Registry (gpr)][gpr]
  * [Quay][quay] - Joey Schorr jschorr@redhat.com
  * [IBM Cloud Container Registry (icr)][icr]
* Registry Vendors, Projects & Products
  * [Docker Trusted Registry][docker-dtr]
  * [Harbor][harbor]
  * [JFrog Artifactory][jfrog]
* Artifact Types
  * [OCI & Docker Container Images][image-spec]
  * [Helm Charts][helm-registry]
  * [Singularity][singularity]
  * Operator Bundles

## Scenarios

Notary v2 aims to solve the core issue of trusting content within, or across registries are what they claim to be. There are many elements of an end to end scenario that are not implemented by Notary v2, rather enabled becuase the content is verifiable.

### End to End Orchestrator Scenario

To put Notary v2 in context, the following scenario is outlined. 
![Notary e2e Scenarios](./media/notary-e2e-scenarios.png)


### Scenario #1: Local Build, Sign, Validate

Prior to committing any code, a developer can test the: "build, sign, validate scenario"

1. Locally build a container image using a non-registry specific `name:tag`, such as:  
  `$ docker build hello-world:1`
1. Locally sign `hello-world:1` 
1. Run the image in a local environment which was configured to only support signed artifacts. *(**note** - details on the usability is deferred from this requirement. This could be a local docker host, a local kind or minikube instance configured to only run signed images, or a completely different artifact type.)*  
  `$ docker run hello-world:1`

**Implications of this requirement:**

- The local developer has access to signing keys
- The local environment has a policy by which it states the set of keys it accepts.
- The signing and validation of artifacts does not require a registry.
- The signature used for validation may be hosted in a registry, or other accessible location.
- The lack of a registry name does not infer docker.io as the default registry. This does imply changes to either the docker client, the notary client or both.
- Signing is performed on the same artifacts that are expected to be uploaded to a registry so that verification of the signature can occur without additional transformation or computation. If the artifact is expected to be compressed, the signature will be performed on the compressed artifact rather than the uncompressed content.

### Scenario #2: Sign, Rename, Push, Validate

Once the developer has locally validated the build, sign, validate scenario, they will likely want to push the artifact to a registry used for deployment.

1. Locally build and sign an artifact, such as a container image
1. Rename the artifact to reflect the registry it will be pushed to:  
  `$ docker tag hello-world:1 myregistry.example.com/marketing/web:1`
  `$ docker push myregistry.example.com/marketing/web:1`
1. Deploy the artifact to a cluster that requires signatures:  
  `$ kubectl apply -f kubedeploy.yaml`
  > **Note:** the above example presumes `kubedeploy.yaml` references the myregistry.example.com/marketing/web:1 image

**Implications of this requirement:**

- Signatures can be verified based on the tag referenced, however the signature is not tied to a specific `repo:tag` name. The artifact can be renamed without invalidating the signature.
- This does not preclude tag locking scenarios, where a registry can provide users with the ability to lock a tag to a specific manifest. If a tag is updated with another signed artifact, the tag is considered signed and follows the signature validation rules of the new signature. For example, if a base image `fx:1.0` is signed, and rebuilt with a patched version, the updated `fx:1.0` is new content, and a valid scenario.
- If the tag for `fx:1.0` is redirected to another unsigned manifest, the signature validation of `fx:1.0` will fail as it's no longer signed. This is no different than signed binaries being updated or replaced on a users local computer.
- Notary v2 supports a pattern for signing any type of artifact, from OCI Images, Helm Charts, Singularity to yet unknown types.

### Scenario #3: Automate Build, Sign, Push, Deploy, Verify

A CI system is triggered by a git commit. The system can build the artifacts, sign them, and push to a registry. The production system can pull the artifacts, verify the signatures and run them.

1. A CI solution responds to a git commit notification
1. The CI system clones the git repo and builds the artifacts, with fully qualified names: **image**: `myregistry.example.com/sample/hello-world:aa31` and a **helm chart**: `myregistry.example.com/sample/charts/hello-world:aa31`
1. The CI system signs the artifact with locally available keys *(**Note:** key management deferred from this requirement)*
1. The artifact is pushed to a registry:  
  `$ docker push myregistry.example.com/sample/hello-world:aa31`
  `$ helm chart push myregistry.example.com/sample/charts/hello-world:aa31`
1. The artifacts are deployed by the orchestrator.
1. Upon request, the orchestrator verifies the signatures prior to deployment.

**Implications of this requirement:**

- Nothing additional from previous scenarios. It calls out a happy path from commit to deploy.

### Scenario 3.1: Required Signatures

Customers may require content to be signed. They may require specific signatures, or any signatures.

1. A deployment is requested. The host has been configured to require any signature, or specific signatures. 
1. The host accepts the request, and verifies it's configuration. If a signature is requried, but no specific signatures are specified, any signed content would be permitted. If specific signatures are required, the deployment will only succeed if the signature matches one of the specified signatures. 
1. The user may specify root signatures, thus any signatures that derive from the root would also considered valid.

### Scenario #4: Promote Artifacts Within a Registry, Using a Different repo

A CI/CD system promotes validated artifacts from a dev repository to production repositories.

1. A CI/CD solution responds to a git commit notification, cloning, building, signing and pushing the artifacts to a development repo within their registry.
1. As the CI/CD solution runs functional tests, determining the artifacts are ready for production, the artifacts are moved from one repo to another.  
  `$ docker tag myregistry.example.com/dev/alpha-team/web:1abc myregistry.example.com/prod/web:1abc`

**Implications of this requirement:**

- Renaming does not invalidate the signature
- Artifact movement, or copy, to a different repository does not invalidate the signature

### Scenario #5: Acquire Artifacts From Public Repositories, Copying to Private Registries for Deployment

As artifacts evolve from base images developers build upon, to products that customers run directly, they will need to run them from private registries which maintain and validate the signature provided by the software vendor.

1. ACME Rockets acquires networking monitoring software they run in their environment.
1. ACME Rockets copies the artifacts, renaming them to reflect their private registry, and pushes them to their registry.  
  `$ docker pull registry.wabbit-networks.com/netmonitor:1.0`
  `$ docker tag registry.wabbit-networks.com/netmonitor:1.0 registry.acme-rockets.com/infra/networking/wabbit-net-monitor:1.0`
  `$ docker push registry.acme-rockets.com/infra/networking/wabbit-net-monitor:1.0`
1. As ACME Rockets deploys the networking monitoring software, the signatures are validated to be signed by ACME Rockets.

**Implications of this requirement:**

- In addition to the above scenario where an artifact is promoted within the same registry, this scenario requires the signature to be available in the new registry. Whether the signature is moved, or referenced from its original location is a subject for design discussion. In this scenario, ACME Rockets could validate the signature against a wabbit-network signature server.

### Scenario #6: Validate Artifact Signatures Within Restricted Networks

ACME Rockets runs secure production environments, limiting all external network traffic. To assure the wabbit-networks network monitor software has valid signatures, they will need to trust a resource within their network to proxy the signature validations.

1. ACME Rockets acquires network monitoring software, copying it to their firewall protected production environment.
1. As part of the artifact copy, they will copy/proxy the signature validation to trusted resources within their network protected environment.

**Implications of this requirement:**

- In this scenario, the wabbit-networks signature must be validated within the ACME Rockets network. How this is done is open for design. However, the requirement states the signature must be validated without external access. When the artifact is copied to the private/network restricted registry, the signature may need to be copied, and is assumed to be trusted if available in the trusted server within the private network. How ACME Rockets would copy/proxy the signatures is part of the design and UX for a secure, but usable pattern.

### Scenario #7 Automated Deploy - Success

A CD solution initiates a deployment to a development environment. The development environment has been previously configured to only accept signed artifacts.

1. A CD solution is triggered to initiate a deployment.
1. The CD solution uses a deployment template to request an orchestrator to run specific images.
1. The orchestrator receives the request, validates the artifact is signed and the signature is valid *before* it can initiate the deployment.
1. The signature is considered valid, and the deployment is successful.

**Implications of this requirement:**

- The aspect of being a signed artifact is not germane to the CD solution. The CD solution simply requests a deployment. The enforcement of signature validation is a policy of the execution environment.

### Scenario #8 Automated Deploy - Failure

Same as #7, with the exception that the signature is deemed invalid.
1. A CD solution is triggered to initiate a deployment.
2. The CD solution uses a deployment template to request an orchestrator to run a specific image. In this case, the CD solution mistakenly attempted to deploy a dev signed artifact to a production environment.
3. The production orchestrator receives the request, validates the artifact is signed and recognizes the signature is **NOT** produced by a trusted key. As the signature is deemed invalid, the deployment fails.

**Implications of this requirement:**
- The coordination between a CD and operational environment for how a request and response is managed is not in-scope for artifact signing.
- An operational environment enforces a policy for which signatures it accepts. Although the artifact signature may be valid, it's NOT the signature required by this operational environment.

### Scenario #9: Multiple Signatures

Customers may require multiple signatures for the following scenarios:

- Validate the artifact is the same as what the vendor provided.
- Secondarily sign the artifact by the consuming company, attesting to its validity within their production environment.
- Signatures represent validations through different dev, staging, production environments.

1. A CI/CD solution builds, signs, pushes and deploys a collection of artifacts to a staging environment.
1. Once integrations tests are completed, the artifacts are signed with a production signature, copying them to a production registry or production set of repositories.
1. The Staging and Production container hosts validate the artifacts contain the signatures required for their environment.

**Implications of this requirement:**

- Multiple signatures, including signatures from multiple sources can be associated with a specific artifact.

## Open Discussions

- What is the relationship between a signature, an artifact and a registry?
- Can signature validation be dependent on an external entity?

[artifacts-repo]:   https://github.com/opencontainers/artifacts
[helm-registry]:    https://v3.helm.sh/docs/topics/registries/
[oci-tob]:          https://github.com/opencontainers/tob
[singularity]:      https://github.com/sylabs/singularity
[acr]:              https://aka.ms/acr/artifacts
[ecr]:              https://aws.amazon.com/ecr/
[docker-hub]:       https://hub.docker.com/
[docker-dtr]:       https://www.docker.com/products/image-registry
[image-spec]:       https://github.com/opencontainers/image-spec
[gcr]:              https://cloud.google.com/container-registry/
[gpr]:              https://github.com/features/package-registry
[quay]:             https://quay.io/
[harbor]:           https://goharbor.io/
[jfrog]:            https://jfrog.com/integration/docker-registry/
[icr]:              https://icr.io/
