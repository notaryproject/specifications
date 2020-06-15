# Notary Key Management Scenarios
In order to allow for end to end signing of artifacts, Notary v2 will require the use of cryptographic keys. These scenarios define how these keys will be generated, distributed, and trusted in Notary v2.

## Scenarios

### Scenario #1: A developer signs a package for the first time
The first time a developer signs a package, they will need to set up a signing key and their local environment.
  1. Generate a developer signing key.
  1. Create a local repository that delegates to the developer key notary-tool --create-local-repository, notary-tool --delegate developer-key.pub

### Scenario #2: A developer is given permission to upload to a registry (for the first time)
The first time a developer uploads to a registry, they need to establish that they are a trusted developer.
  1. Add the developer key to delegating metadata on the registry. The delegating metadata may be top-level targets metadata, an organization-hosted targets metadata file (see TAP 13 for how organization-specific targets metadata can be used), or a different targets metadata file.

### Scenario #3: A developer updates their signing key
In the event of a compromised or lost key, a developer may need to switch to a new signing key.
  1. The developer will generate a new signing key.
  1. The new developer key will need to be added to the delegating targets metadata.
  1. The previous developer key should no longer be trusted.
