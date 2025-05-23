# Notation Extensibility for Signing and Verification

Keys and associated certificates used for signing artifacts using Notation could be available to users through varied solutions that provide secure key generation, storage and cryptographic operations. Some are well established with standards like [PIV](https://csrc.nist.gov/projects/piv/piv-standards-and-supporting-documentation) and [PKCS #11](https://docs.oasis-open.org/pkcs11/pkcs11-base/v3.0/os/pkcs11-base-v3.0-os.html) implemented by hardware tokens, smart cards. More recent options which use varied authentication and API protocols are remote key management services and signing services by third party vendors and cloud service providers. Notation will support a few built-in integrations with standard providers, and will provide plugin interfaces for users, and vendors to implement their own integrations with the solutions they use. This allows a plugin publisher to implement, test, release and patch their solutions independent of Notation’s development and release cycle. This document provides specification for the plugin model, and interfaces to implement. This specification aims to work both for [existing](./signature-specification.md) and future signature formats adopted by Notary Project.

## Terminology

* **Plugin Publisher** - A user, organization, open source project or 3rd party vendor that creates a Notation plugin for internal or public distribution.
* **Plugin** - A component external to Notation that can integrate as one of the steps in Notation’s workflow for signature generation or verification. A Notation plugin can be distributed as a single executable file, or an archive file (`zip` or `tar.gz`).
* **Default provider** - Signing and verification mechanisms built into Notation itself to provide default experience without requiring to install/configure additional plugins. ***[We are yet to define what is included in the default experience]***.

## Plugin mechanism

### Requirements

* A plugin publisher MUST be able to distribute and patch a Notation plugin independently of Notation’s release cycle.
* The plugin mechanism MUST work across commonly used OS platforms like Linux, Windows and macOS.
* The plugin interface contract MUST be versioned. This version is different from Notation's release version. It will be used to accommodate for additional capabilities and contract changes post initial release of Notation.
* A plugin MAY implement a subset of capabilities (features) available in plugin contract. E.g A plugin may implement signing feature, but not verification.
* Notation and plugins MAY be updated independently in an environment.
  * Notation MUST work with a plugin that implements a matching or lower minor version of the plugin contract. Notation SHALL NOT support using a plugin with higher version of plugin contract.
  * A plugin MUST support a single plugin contract version, per major version.

Notation will invoke plugins as executable, pass parameters using command line arguments, and use standard IO streams to pass request/response payloads. This mechanism is used as Go language (used to develop [Notation Go library](https://github.com/notaryproject/notation-go)) does not have a [in built support](https://github.com/golang/go/issues/19282) to load and execute plugins that works across OS platforms. Other mechanisms like gRPC require every plugin to be implemented as a service/daemon.

### Package and release a plugin

Notation supports the installation of a plugin from an `https` URL or from a file in the filesystem. To support the plugin installation and management using `notation plugin` commands, the plugin publisher must adhere to the following conventions:

* Plugin executable file MUST follow the naming convention `notation-{plugin-name}`. On `windows` OS, the file extension `.exe` is REQUIRED.
* The plugin distribution format MUST be either .zip, .tar.gz, or a single plugin executable file. If the release format is _single plugin executable file_, it is highly recommended to compress it into an archive for installation efficiency consideration.
* If the archive format is `.zip` or `.tar.gz`, there MUST be one and only one plugin executable file within each archive.
* Plugin publisher MUST provide SHA256 checksum for each released archive or _single plugin executable file_ if they want to enable users to install a plugin from an `https` URL.
* The plugin archive MAY contain License files. Also, it's recommended to include licenses in the archive.
* Currently, Notation facilitates the installation of plugin executable files and archives, with a size limit of less than 256 MiB. Consequently, the plugin executable file and archive size MUST be less than 256 MiB. Note: Although users have the option to install plugins larger than 256 MiB, they will be unable to utilize the `notation plugin install` command in such cases.

For example, an archive of a Notation plugin `helloworld` for Linux AMD64 machine `notation-helloworld_1.0.1_linux_amd64.tar.gz` includes these files:

```
notation-helloworld_1.0.1_linux_amd64.tar.gz
├── notation-helloworld (required)
├── LICENSE (optional)
└── Dependencies (required if present)
```

### Plugin lifecycle management

#### Installation

Plugin publisher will provide instructions to download and install the plugin. Plugins intended for public distribution should also include instructions for users to verify the authenticity of the plugin.

##### Plugin installation methods

Notation offers [plugin management](https://github.com/notaryproject/notation/blob/v1.1.0/specs/commandline/plugin.md) commands that allow users to install plugins from various sources, including `https` URL and filesystem.

##### Plugin installation path

To enumerate all available plugins the `PLUGIN_DIRECTORY` is scanned based on per OS:
| OS      | PLUGIN_DIRECTORY                                     |
| ------- | ---------------------------------------------------- |
| Unix    | `$XDG_CONFIG_HOME/notation/plugins`                  |
| Windows | `%AppData%/notation/plugins`                         |
| Darwin  | `$HOME/Library/Application Support/notation/plugins` |

Each plugin executable must be located under `$PLUGIN_DIRECTORY/{plugin-name}` directory, with executable named as  `notation-{plugin-name}`.

Any directory found inside `$PLUGIN_DIRECTORY` is considered potential plugin "candidates". Anything found which is not a directory is ignored and is not considered as a plugin candidate.

To be considered a valid plugin a candidate must pass each of these "plugin candidate tests":

* The directory must contain an executable named `notation-{plugin-name}`.
* The executable MUST be a regular file, symlinks are not supported. Implementation MUST validate that the executable is a regular file, before executing it, and fail it it does not meet this condition.
* On Windows, executables must have a `.exe` suffix.
* Must, where relevant, have appropriate OS "execute" permissions (e.g. Unix x bit set) for the current user.
* Must actually be executed successfully and when executed with the subcommand `get-plugin-metadata` must produce a valid JSON metadata (and nothing else) on its standard output (schema to be discussed later).

### Commands

Notation provides various commands, including `install`, `list`, and `uninstall`, to manage plugin lifecycle. For more information, please refer to the [`notation plugin` CLI specification](https://github.com/notaryproject/notation/blob/v1.1.0/specs/commandline/plugin.md).

### Using a plugin for signing

* To use a plugin for signing, the user associates the plugin as part of registering a signing key. E.g.
  * `notation key add --name "mysigningkey" --id "keyid" --plugin "com.example.nv2plugin"`
  * In the example, the command registers a signing key in `/notation/config.json`, where `mysigningkey` is a friendly key name to refer during signing operation from the CLI, `id` is an key identifier known to plugin that is used for signing, and the value of `plugin` specifies it's using a plugin located at `~/notation/plugins/com.example.nv2plugin/notation-com.example.nv2plugin`.
  
```jsonc
{
  "signingKeys": {  
    "default": "",  
    "keys": [
        {  
          "name": "mysigningkey",  
          "id" : "keyid",
          "plugin": "com.example.nv2plugin",
          // Optional configuration as required by the plugin
          "pluginConfig" : {   
            "key1" : "value1",
            "key 2" : "value 2"
          }
        }
    ]  
  }
}
```

### Plugin configuration

Plugins may require additional configuration to work correctly, that could be set out of band, or provided by notation when it invokes a plugin. To support plugin configuration through notation, the key configuration provides an optional `pluginConfig` map of of key value pairs, that is passed as-is by notation to the plugin.

* To use this feature, plugin authors MUST define and document the set of plugin configuration keys-values, which is set by users when they associate a plugin with signing key.
* Plugin authors SHOULD NOT use plugin configuration to store sensitive configuration in plaintext, such as authentication keys used by the plugin etc.

For the previous example, plugin config can be set using command line arguments as part of registering a key with the `notation key add` command. E.g.

* `notation key add "mysigningkey" --id "keyid" --plugin "com.example.nv2plugin" --plugin-config key1=value1 --plugin-config "key 2"="value 2"`

Plugin config can be also set/overriden during signing with the `notation sign` command. Following example overrides value for `key 2` already set in `config.json` by previous command.

* `notation sign $IMAGE --key "mysigningkey" --plugin-config "key 2"=newValue2`

### Plugin contract

* Notation will invoke the plugin executable for each command (e.g. sign, verify), pass inputs through `stdin` and get output through `stdout` and `stderr`. Currently, a size limit of less than 64 MiB is applied to each output channel for preventing out-of-memory issues of potential plugin malfunctioning.
* The command will be passed as the first argument to the plugin e.g. `notation-{plugin-name} <command>`. A JSON request is passed using `stdin`. The plugin is expected to return a JSON response through `stdout` with a `0` exit code for successful response, and a non-zero exit code with a JSON error response in `stderr` for error response. Each command defines its request, response and error contract. To avoid any additional content like debug or info level logging from dependencies and inbuilt libraries, the plugin implementation should redirect any output to `stdout` on initialization, and only send the JSON response away from `stdout` when the command execution completes. E.g. For golang, set [`os.Stdout`](https://pkg.go.dev/os#pkg-variables) to point to a log file.
* Every request JSON will contain a `contractVersion` top level attribute whose value will indicate the plugin contract version. Contract version is revised when there are changes to command request/response, new plugin commands are introduced, and supported through Notation.
* To maintain forward compatibility plugin implementors MUST ignore unrecognized attributes in command request which are introduced in minor version updates of the plugin contract.
* For an error response, every command returns a non-zero exit code 1, with an OPTIONAL JSON error response in `stderr`. It is recommended to return an error response to help user troubleshoot the error.  There is no need to send different exit codes for different error conditions, as Notation (which will call the plugin and parse the response) will use `error-code` in the error response to interpret different error conditions if needed. Notation will attempt to parse the error response in `stderr` when exit code is 1, else treat it as a general error for any other non-zero exit codes.

```jsonc
{
  // Each command defines expected error codes
  "errorCode" : "<error code>",  
  // Plugin defined error message.
  "errorMessage" : "User friendly error message",

  // Optional plugin defined additional key value pairs related to the error.
  "errorMetadata" : {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

### Plugin metadata

* Every plugin MUST implement a metadata discovery command called `get-plugin-metadata`.
* Notation will invoke this command as part of signing and verification workflows to discover the capabilities of the plugin and subsequently invoke commands corresponding to the capabilities. Notation will invoke the `get-plugin-metadata` command every time a signing or verification workflow is invoked to discover metadata. Notation will not cache the response of `get-plugin-metadata` command as it involves invalidating cache when a plugin is updated, detecting a plugin update can be non trivial. The `get-plugin-metadata` command is expected to return only static data, and plugin implementors should avoid making remote calls in this command.

***get-plugin-metadata***

*Request*

```jsonc
{
  // Optional plugin configuration, map of string-string
  "pluginConfig" : { }
}
```

*Response*
All response attributes are required.

```jsonc
{
  // Plugin name that matches its install dir
  // e.g. "com.example.nv2plugin".
  "name" : "<plugin name>",  
  // Plugin description.
  "description" : "description>",
  // Plugin publisher controlled version.
  "version" : "<version>",
  // Plugin webpage for support or documentation.
  "url" : "<URL>",
  
  // List of contract versions supported by the plugin, one per major version
  "supportedContractVersions" : [ ],
  
  // List of one or more capabilities supported by the plugin.
  // See the 'capabilities' section below for valid values and details.
  "capabilities" : [
    
  ]
}
```

*plugin-name* - Plugin name uses reverse domain name notation to avoid plugin name collisions.

*supported-contract-versions* - The list of contract versions supported by the plugin. Currently this list must include only one version, per major version. Post initial release, Notation may add new features through plugins, in the form of new commands (e.g. tsa-sign for timestamping), or additional request and response parameters. Notation will publish updates to plugin interface along with appropriate contract version update. Backwards compatible changes (changes for which older version of plugin continue to work with versions of Notation using newer contract version) like new optional parameters on existing contracts, and new commands will be supported through minor version contract updates, breaking changes through major version updates. To maintain forward compatibility plugin implementors MUST ignore unrecognized attributes in command request which are introduced in minor version updates of the plugin contract. Plugin `get-plugin-metadata` command returns the contract version a plugin supports. Notation will evaluate the minimum plugin version required to satisfy a user's request, and reject the request if the plugin does not support the required version.

*capabilities* - A non-empty list of features supported by the plugin. Each capability requires the plugin to implement specific commands. Implementations of Notary Project spec can evaluate the required capability of a user's request and may reject the request if the plugin does not support it.

Valid capabilities and required commands:
  - `SIGNATURE_GENERATOR.RAW`: Generates a raw signature for OCI or blob payloads.
    - Required commands: `describe-key`, `generate-signature`
  - `SIGNATURE_GENERATOR.ENVELOPE`: Generates a complete signature envelope for OCI payloads.
    - Required command: `generate-envelope`
  - `SIGNATURE_GENERATOR.ENVELOPE_FOR_BLOB`: Generates a complete signature envelope for blob payloads.
    - Required commands: `describe-key`, `generate-envelope`
  - `SIGNATURE_VERIFIER.TRUSTED_IDENTITY`: Performs trusted identity verification.
    - Required command: `verify-signature`
  - `SIGNATURE_VERIFIER.REVOCATION_CHECK`: Performs revocation check verification.
    - Required command: `verify-signature`

## Signing interfaces

### Requirements

* The interface MUST NOT be tied to a specific signature envelope format.

Notation will support plugins to be developed against the following interfaces - *Signature Generator*, and *Signature Envelope Generator*. These interfaces target abstraction levels that satisfy most plugin integration scenarios.

### Signature Generator

This interface targets plugins that integrate with providers of basic cryptographic operations (e.g., Local PIV/PKCS#11 hardware tokens, remote KMS, or key vault services). Plugins that target this interface will only generate a raw signature given a payload to sign. Plugin developers who implement the Notary Project specification are responsible for packaging this signature into a signature envelope along with a signature manifest (OCI) or a signature file (blob). Plugin developers are also responsible for generating the TSA signature if indicated by the user. The plugin does not need to be signature envelope format aware and will continue to work if Notary Project adopts additional signature formats.

#### Signing workflow using plugin

**OCI signing:**
1. Given a user request to sign an OCI artifact, with signing key `keyName` (the friendly key name)
2. Pull the image manifest using the `oci-artifact` URL, and construct a descriptor
3. Append any user provided metadata and metadata defined in [Notary Project signature specification](signature-specification.md) as descriptor annotations.
4. Determine if the registered key uses a plugin
5. Execute the plugin with `get-plugin-metadata` command
    1. If plugin supports capability `SIGNATURE_GENERATOR.RAW`
        1. Execute the plugin with `describe-key` command, set `request.keyId` and the optional `request.pluginConfig` to corresponding values associated with signing key `keyName` in `config.json`.
        2. Generate the payload to be signed for [JWS](./signature-specification.md#supported-signature-envelopes) envelope format.
           1. Create the JWS protected headers collection and set `alg` to value corresponding to `describe-key.response.keySpec` as per [signature algorithm selection](./signature-specification.md#algorithm-selection).
           2. Create the Notary Project signature Payload (JWS Payload) as defined [here](./signature-specification.md#payload).
           3. The *payload to sign* is then created as - `ASCII(BASE64URL(UTF8(ProtectedHeaders)) ‘.’ BASE64URL(JWSPayload))`
        3. Execute the plugin with `generate-signature` command.
           1. Set `request.keyId` and the optional `request.pluginConfig` to corresponding values associated with signing key `keyName` in `config.json`.
           2. Set `request.payload` as base64 encoded *payload to sign* (the JWS *payload to sign* is double encoded, this is a shortcoming of using plugin contract with JSON encoding).
           3. Set `keySpec` to value returned by `describe-key` command in `response.keySpec`, and `hashAlgorithm` to hash algorithm corresponding to the key spec, as per [signature algorithm selection](./signature-specification.md#algorithm-selection). The algorithm specified in `hashAlgorithm` MUST be used by the plugin to hash the payload (`request.payload`) as part of signature generation.
        4. Validate the generated signature, return an error if any of the checks fails.
           1. Check if `response.signingAlgorithm` is one of [supported signing algorithms](./signature-specification.md#algorithm-selection).
           2. Check that the plugin did not modify `request.payload` before generating the signature, and the signature is valid for the given payload. Verify the hash of the `request.payload` against `response.signature`, using the public key of signing certificate (leaf certificate) in `response.certificateChain` along with the `response.signingAlgorithm`. This step does not include certificate chain validation (certificate chain leads to a trusted root configured in Notation's Trust Store), or revocation check.
           3. Check that the `response.certificateChain` conforms to [Certificate Requirements](./signature-specification.md#certificate-requirements).
        5. Assemble the signature envelope using `response.signature`, `response.signingAlgorithm` and `response.certificateChain`. If the signing scheme is [`notary.x509`](./signing-scheme.md/#notaryx509), implementations SHOULD also request and include TSA timestamp countersignature in this step. The `certReq` field in the [timestamping request](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.1) MUST be set to true.
        6. Generate a signature manifest for the given signature envelope.
    2. Else if, plugin supports capability `SIGNATURE_GENERATOR.ENVELOPE` *(covered in next section)*
    3. Return an error

**Blob signing:**
1. Given a user request to sign a blob, with signing key `keyName`
2. Determine if the registered key uses a plugin
3. Execute the plugin with the `get-plugin-metadata` command and check its capabilities:
    1. **If the plugin supports `SIGNATURE_GENERATOR.RAW`:**
        1. Execute the plugin with the `describe-key` command, setting `request.keyId` and the optional `request.pluginConfig` to the values associated with signing key `keyName` in `config.json`.
        2. Generate the digest of the blob using the **hash algorithm** specified in the `keySpec` from the `describe-key` response. Construct the payload to be signed using this digest, as described in the [signature specification](signature-specification.md#payload).
        3. Follow the same signing steps as in the OCI signing workflow above (i.e., generate the signature, validate it, and assemble the signature envelope).
        4. Generate the signature file for the resulting signature envelope.
    2. **Else if** plugin supports capability `SIGNATURE_GENERATOR.ENVELOPE_FOR_BLOB` *(covered in next section)*
    3. **Else** return an error indicating that the plugin does not support the required signing capability.

> Note: The plugin does not need to be aware of the signature envelope format or whether it is signing an OCI artifact or a blob; it only receives the payload to sign.

#### describe-key

This command is used to get metadata for a given key. The command is required for `SIGNATURE_GENERATOR.RAW` and `SIGNATURE_GENERATOR.ENVELOPE_FOR_BLOB` capabilities.

*Request*

```jsonc
{
  "contractVersion" : "<major-version.minor-version>",

  // Key id associated with signing key (keyName)
  // in config.json /signingKeys/keys
  "keyId": "<key id>",

  // Optional plugin configuration, map of string-string
  "pluginConfig" : { }
}
```

*keyId* : Required field that has the key identifier (`keyId`) associated with signing key `keyName` in `config.json`.

*pluginConfig* : Optional field for plugin configuration. For details, see [Plugin Configuration](#plugin-configuration) section.

*Response*

```jsonc
{
   // The same key id as passed in the request.
  "keyId" : "<key id>",
  "keySpec" : "<key type and size>"
}
```

*keySpec* : One of following [supported key types](./signature-specification.md#algorithm-selection) - `RSA-2048`, `RSA-3072`, `RSA-4096`, `EC-256`, `EC-384`, `EC-521`.

NOTE: This command can also be used as part of `notation key describe {key-name}` which will include the following output

* Summary of key definition from `config.json`
* If the key has an associated plugin
  * Output of plugin `discover` command
  * Output of `describe-key` command for the specific key

#### generate-signature

This command is used to generate the raw signature for a given payload.

*Request*

```jsonc
{
  "contractVersion" : "<major-version.minor-version>",

  // Key id associated with signing key (keyName)
  // in config.json /signingKeys/keys
  "keyId": "<key id>",

  // Optional plugin configuration, map of string-string
  "pluginConfig" : { },

  // The key spec for the given key id
  "keySpec" : "<key type and size>",

  // Hash algorithm associated with the key spec, plugin must 
  // hash the payload using this hash algorithm
  "hashAlgorithm" : "SHA-256" | "SHA-384" | "SHA-512",

  // Payload to sign, this is base64 encoded
  "payload" : "<base64 encoded payload to be signed>",

}
```

*keyId* : Required field that has the key identifier (`keyId`) associated with signing key `keyName` in `config.json`.

*pluginConfig* : Optional field for plugin configuration. For details, see [Plugin Configuration section](#plugin-configuration).

*keySpec* : Required field that has one of following [supported key types](./signature-specification.md#algorithm-selection) - `RSA-2048`, `RSA-3072`, `RSA-4096`, `EC-256`, `EC-384`, `EC-521`. Specifies the key type and size for the key.

*hashAlgorithm* : Required field that specifies Hash algorithm corresponding to the signature algorithm determined by `keySpec` for the key.

*payload* : Required field that contains base64 encoded payload to be signed. For JWS, the *payload to sign* is base64 encoded (a second time) before sending to the plugin.

*Response*

All response attributes are required.

```jsonc
{    
  // The same key id as passed in the request.
  "keyId" : "<key id>",
  "signature" : "<Base64 encoded signature>",
  "signingAlgorithm" : "<signing algorithm>",
  "certificateChain": ["Base64(DER(leafCert))","Base64(DER(intermediateCACert))","Base64(DER(rootCert))"]
}
```

*signingAlgorithm* : One of following [supported signing algorithms](./signature-specification.md#algorithm-selection), Notation uses this validate the signature, and to set the appropriate attribute in signature envelope (e.g. JWS `alg`). Supported values are

* `RSASSA-PSS-SHA-256`: RSASSA-PSS with SHA-256
* `RSASSA-PSS-SHA-384`: RSASSA-PSS with SHA-384
* `RSASSA-PSS-SHA-512`: RSASSA-PSS with SHA-512
* `ECDSA-SHA-256`: ECDSA on secp256r1 with SHA-256
* `ECDSA-SHA-384`: ECDSA on secp384r1 with SHA-384
* `ECDSA-SHA-512`: ECDSA on secp521r1 with SHA-512

*certificateChain* : Ordered list of certificates starting with leaf certificate and ending with root certificate.

#### Error codes for describe-key and generate-signature

* VALIDATION_ERROR - Any of the required request fields was empty, or a value was malformed/invalid. Includes condition where the key referenced by `keyId` was not found.
* UNSUPPORTED_CONTRACT_VERSION - The contract version used in the request is unsupported.
* ACCESS_DENIED - Authentication/authorization error to use given key.
* TIMEOUT - The operation to generate signature timed out and can be retried by Notation.
* THROTTLED - The operation to generate signature was throttles and can be retried by Notation.
* ERROR - Any general error that does not fall into previous error categories.

### Signature Envelope Generator

This interface targets plugins that, in addition to signature generation, want to generate the complete signature envelope. This interface allows plugins to have full control over the generated signature envelope, and can append additional signed and unsigned metadata, including timestamp countersignatures. The plugin must be signature envelope format aware, and implement new formats when Notary Project adopts new formats.

#### Signing workflow using plugin

**OCI signing:**
1. Given a user request to sign an OCI artifact, with `keyName` (friendly key name)
2. Pull the image manifest using the `image` URL, and construct a descriptor
3. Append any user provided metadata and metadata defined in [Notary Project signature specification](signature-specification.md) as descriptor annotations.
4. Determine if the registered key uses a plugin
5. Execute the plugin with `get-plugin-metadata` command
    1. If plugin supports capability `SIGNATURE_GENERATOR.ENVELOPE`
        1. Execute the plugin with `generate-envelope` command. Set `request.keyId` and the optional `request.pluginConfig` to corresponding values associated with signing key `keyName` in `config.json`. Set `request.payload` to base64 encoded [Notary Project signature Payload](./signature-specification.md#payload), `request.payloadType` to `application/vnd.cncf.notary.payload.v1+json` and `request.signatureEnvelopeType` to a pre-defined type (`application/jose+json` for JWS).
        2. If plugin supports timestamping under signing scheme [`notary.x509`](./signing-scheme.md/#notaryx509), the plugin SHOULD request and include TSA timestamp countersignature in the signature envelope at this step. The timestamp countersignature MUST be [RFC 3161](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.2) compliant. The `certReq` field in the [timestamping request](https://datatracker.ietf.org/doc/html/rfc3161#section-2.4.1) MUST be set to true.
        3. `response.signatureEnvelope` contains the base64 encoded signature envelope, value of `response.signatureEnvelopeType` MUST match `request.signatureEnvelopeType`.
        4. Validate the generated signature envelope, return an error if any of the checks fails.
           1. Check if `response.signatureEnvelopeType` is a supported envelope type and `response.signatureEnvelope`'s format matches `response.signatureEnvelopeType`.
           2. Check if the signing algorithm in the signature envelope is one of [supported signing algorithms](./signature-specification.md#algorithm-selection).
           3. Check that the [`targetArtifact` descriptor](./signature-specification.md#payload) in JWSPayload in `response.signatureEnvelope` matches `request.payload`. Plugins MAY append additional annotations but MUST NOT replace/override existing descriptor attributes and annotations.
           4. Check that `response.signatureEnvelope` can be verified using the public key and signing algorithm specified in the signing certificate, which is embedded as part of certificate chain in `response.signatureEnvelope`. This step does not include certificate chain validation (certificate chain leads to a trusted root configured in Notation), or revocation check.
           5. Check that the certificate chain in `response.signatureEnvelope` and [timestamp countersignature](./signature-specification.md#unsigned-attributes) confirm to [certificate requirements](./signature-specification.md#certificate-requirements).
        5. Generate a signature manifest for the given signature envelope, and append `response.annotations` to manifest annotations.
    2. Else if plugin supports capability `SIGNATURE_GENERATOR.RAW` *(covered in previous section)*
    3. Return an error

**Blob signing:**
1. Given a user request to sign a blob, with signing key `keyName` (the friendly key name)
2. Determine if the registered key uses a plugin
3. Execute the plugin with the `get-plugin-metadata` command and check its capabilities:
    1. **If the plugin supports `SIGNATURE_GENERATOR.ENVELOPE_FOR_BLOB`:**
        1. Execute the plugin with the `describe-key` command, setting `request.keyId` and the optional `request.pluginConfig` to the values associated with signing key `keyName` in `config.json`.
        2. Generate the digest of the blob using the **hash algorithm** specified in the `keySpec` from the `describe-key` response. Construct the payload to be signed using this digest, as described in the [signature specification](signature-specification.md#payload).
        3. Follow the same envelope generation steps as in the OCI signing workflow above (i.e., generate the envelope and validate it).
        4. Generate the signature file for the resulting signature envelope.
    3. **Else** return an error indicating that the plugin does not support the required signing capability.

> Note: The plugin does not need to be aware of whether it is signing an OCI artifact or a blob; however, the plugin must provide an additional `describe-key` command for `SIGNATURE_GENERATOR.ENVELOPE_FOR_BLOB` capability compared to `SIGNATURE_GENERATOR.ENVELOPE` capability.

#### generate-envelope

This command is used to generate the complete signature envelope for a given payload.

*Request*
All request attributes are required.

```jsonc
{
  "contractVersion" : "<major-version.minor-version>",
  
  // Key id associated with signing key (keyName)
  // in config.json /signingKeys/keys
  "keyId": "<key id>",

  // Optional plugin configuration, map of string-string
  "pluginConfig" : { },

  // Optional signature expiry duration in seconds, number
  "expiryDurationInSeconds" : 1234567,

  "payload" : "<Base64 encoded payload to be signed>",
  
  // The type of payload - currently a descriptor
  "payloadType" : "application/vnd.cncf.notary.payload.v1+json",
  
  // The expected response signature envelope
  "signatureEnvelopeType" : "application/jose+json"
}
```

*keyId* : Required field that has the key identifier (`keyId`) associated with signing key `keyName` in `config.json`.

*pluginConfig* : Optional field for plugin configuration. For details, see [Plugin Configuration section](#plugin-configuration).

*expiryDurationInSeconds* : Optional field which contains signature expiry duration(in seconds) as integer number.

*signatureEnvelopeType* - defines the type of signature envelope expected from the plugin. As  Notation clients need to be updated in order to parse and verify new signature formats, the default signature format can only be changed with new major version releases of Notation. Users however can opt into using an updated signature envelope format supported by Notation, by passing an optional parameter.
e.g. `notation sign $IMAGE --key {key-name} --signature-format {some-new-type}`

*Response*
All response attributes are required.

```jsonc
{
   "signatureEnvelope": "<Base64 encoded signature envelope>",
   "signatureEnvelopeType" : "application/jose+json",
   
  // Annotations to be appended to Signature Manifest annotations
  "annotations" : {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

#### Error codes for generate-envelope

* VALIDATION_ERROR - Any of the required request fields was empty, or a value was malformed/invalid. Includes condition where the key referenced by `keyId` was not found, payload type or signature envelope type in the request is unsupported by the plugin.
* UNSUPPORTED_CONTRACT_VERSION - The contract version used in the request is unsupported.
* ACCESS_DENIED - Authentication/authorization error to use given key.
* TIMEOUT - The operation to generate signature timed out and can be retried by Notation.
* THROTTLED - The operation to generate signature was throttles and can be retried by Notation.
* ERROR - Any general error that does not fall into previous error categories.

## Verification extensibility

### Requirements

* The interface MUST NOT be tied to a specific signature envelope format.
* The interface MUST NOT allow a plugin to customize the complete [signature verification workflow](./trust-store-trust-policy.md#signature-verification). Certain steps like identifying the applicable trust policy for the artifact, signature envelope schema validation, integrity checks, certificate chain validation against trust store will be performed by Notation, and cannot be customized by a plugin.
* The interface MUST allow plugins to customize verification logic for specific supported steps
  * Trusted Identity validation
  * Revocation check validation
* The interface MAY be extended in future to support additional steps which can be customized through a plugin.
* The interface MUST be agnostic of sequencing of steps in signature verification workflow as implemented in Notation or other implementations.
* The interface MUST allow processing of [extended attributes](./signature-specification.md#extended-attributes) that are not part of the Notary Project signature [standard attributes](./signature-specification.md#standard-attributes).

### Guidelines for Verification plugin publishers

* Usage of extended signed attributes which are marked critical in signature will have implications on portability of the signature.
The environment where verification occurs will require dependencies on either a compatible verification plugin in addition to Notation, or a compliant verification tool that understands the extended signed attributes.
Therefore, signatures intended for public distribution which require broad signature portability SHOULD avoid extended signed attributes which are marked critical.

* Signatures which require a plugin for verification may be distributed privately (e.g. within an organization) or publicly (e.g. via a public registry).
If the plugin publisher wants their plugin used publicly they SHOULD publish specifications for the verification logic the plugin performs and test vectors.
This allows implementations of the [Notary Project signature specification](./signature-specification.md) to perform the same logic themselves, if they choose to.

### Signature Verifier

#### Verification workflow using plugin

1. If signature envelope contains *Verification Plugin* attribute, check if the plugin with the given name exists, else fail signature verification.
    1. For the resolved plugin, call the `get-plugin-metadata` plugin command to get plugin version and capabilities.
    2. If signature envelope contains *Verification plugin minimum version* attribute.
        1. Validate that *Verification plugin minimum version* and plugin version are in SemVer format
        2. Validate that plugin version is greater than or equal to *Verification plugin minimum version*
        3. Fail signature verification if these validations fail
    3. Validate if plugin capabilities contains any `SIGNATURE_VERIFIER` capabilities
        1. Fail signature verification if a matching plugin with `SIGNATURE_VERIFIER` capability cannot be found. Include *Verification Plugin Name* attribute as a hint in error response.
2. Complete steps *Identify applicable trust policy* and *Proceed based on signature verification level* from [signature verification workflow](./trust-store-trust-policy.md#steps).
3. Complete steps *Validate Integrity, Validate Expiry* and *Validate Trust Store* from [signature verification workflow](./trust-store-trust-policy.md#steps).
4. Based on the signature verification level, each validation may be enforced, logged or skipped.
5. Populate `verify-signature` request. NOTE: The processing order of remaining known attributes does not matter as long as they are processed before the end of signature verification workflow.
    1. Set `request.signature.criticalAttributes` to the set of [standard Notary Project signature attributes](./signature-specification.md#standard-attributes) that are marked critical, from the signature envelope.
    2. Set `request.signature.criticalAttributes.extendedAttributes` to extended attributes that Notation does not recognize. Notation only supports JSON primitive types for critical extended attributes. If a signature producer required complex types, it MUST set the value to a primitive type by encoding it (e.g. Base64), and MUST be decoded by the plugin which processed the attribute.
    3. Set `request.signature.unprocessedAttributes` to set of critical attribute names that are marked critical, but unknown to Notation, and therefore cannot be processed by Notation.
    4. Set `request.signature.certificateChain` to ordered array of certificates from signing envelope.
    5. Set `request.trustPolicy.trustedIdentities` to corresponding values from user configured trust policy that is applicable for given artifact and was identified in step 2.
    6. Set `request.trustPolicy.signatureVerification` to the set of verification checks that are supported by the plugin, and are required to be performed as per step 2. Notation may not populate some checks in this array, if it determined them as “skipped” as per user’s trust policy.
6. Process `verify-signature.response`
    1. Validate `response.verificationResults` map keys match the set of verifications requested in  `request.trustPolicy.signatureVerification` . Fail signature verification if they don't match.
    2. For each element in `response.verificationResults` map, process each result as follows
        1. If `verificationResult.success` is true, proceed to next element.
        2. Else if `verificationResult.success` is `false`, check if the failure should be enforced or logged based on value of `signatureVerification` level in Trust Policy. Proceed to next element if failure is logged, else fail signature verification with `verificationResult.reason`.
    3. Validate values in `response.processedAttributes` match the set of values in `request.signature.unprocessedAttributes`. If they don't, fail signature verification, as the plugin did not process a critical attribute that is unknown to the caller.
7. Perform any remaining steps in [signature verification workflow](./trust-store-trust-policy.md#signature-verification) which are not covered by plugin's verification capabilities. These steps MUST process any remaining critical attributes. Fail signature verification if any critical attributes are unprocessed at the end of this step.

#### *verify-signature* command

*Request*

```jsonc
{
  "contractVersion" : "<major-version.minor-version>",
  
   "signature" : {
    // Array of all the Notary Project defined critical attributes and their values
    // in the signature envelope. Agnostic of header names and value serialization
    // in specific envelope formats like JWS or COSE.
    "criticalAttributes" : 
    { 
       "contentType" : "application/vnd.cncf.notary.payload.v1+json",
       // One of notary.x509 or notary.x509.signingAuthority
       "signingScheme" : "notary.x509" | "notary.x509.signingAuthority",
       // Value is always RFC 3339 formatted date time string
       "expiry": "2022-10-06T07:01:20Z", 
       // if signingScheme is notary.x509.signingAuthority
       "authenticSigningTime": "2022-04-06T07:01:20Z",
       // Name of the verification plugin
       "verificationPlugin": "com.example.nv2plugin",
       // Optional plugin minimum version, if present.
       "verificationPluginMinVersion": "1.0.0",
       // Map of extended attributes
       // Extended attributes values support JSON primitive values string, number and null
       "extendedAttributes" : {
           // Map of extended attributes
          // Extended attributes values support JSON primitive values string, number and null.
           "name" : "<primitive-value>" 
       }
    },
   
    // Array of names of critical attributes that plugin 
    // caller does not understand and does not intend to process.
    "unprocessedAttributes" : [ ],

    // Certificate chain from signature envelope.
    "certificateChain" : [ ]
  },

  "trustPolicy" : {
    // Array of trusted identities as specified in 
    // trust policy.
    "trustedIdentities": [],
    // Array of verification checks to be performed by the plugin
    "signatureVerification" : [ 
        "SIGNATURE_VERIFIER.TRUSTED_IDENTITY",
        "SIGNATURE_VERIFIER.REVOCATION_CHECK"
        ]
  },
  
  // Optional plugin configuration, map of string-string
  "pluginConfig" : { }
}
```

The request can be extended in future by including other elements of signature envelope and trust policy as required to customize additional steps in verification.

*Response*
All response attributes are required.

```jsonc
{
  "verificationResults" : {
        // Map of results. Key must match the set of verification capabilities 
        // in verify-signature request's trustPolicy.signatureVerification attribute.
        "SIGNATURE_VERIFIER.TRUSTED_IDENTITY" :
        {
            // The result of a verification check
            "success" : true | false,
            // Reason for check being successful or not,
            // required if value of success attribute is false.
            "reason" : ""
        },
        "SIGNATURE_VERIFIER.REVOCATION_CHECK" :
        {
            // The result of a verification check
            "success" : true | false,
            // Reason for check being successful or not,
            // required if value of success attribute is false.
            "reason" : ""
        }
  },
  // Array of strings containing critical attributes processed by the plugin.
  "processedAttributes" : [ 
  ]
}
```

*verificationResults* : Verifications performed by the plugin. This is a map where the keys MUST match set of verify capabilities in `verify-signature` request's `trustPolicy.signatureVerification` attribute, and values are objects with following attributes

* *success* (required): The `boolean` verification result.
* *reason* (optional): Reason associated with verification being successful or not, REQUIRED if value of success field is `false`.

*processedAttributes* (required): Array of strings containing critical attributes processed by the plugin. Values must be one or more of attribute names in `verify-signature` request's `signature.unprocessedAttributes`.

#### Error codes for *verify-signature*

* VALIDATION_ERROR - Any of the required request fields was empty, or a value was malformed/invalid.
* UNSUPPORTED_CONTRACT_VERSION - The contract version used in the request is unsupported.
* ACCESS_DENIED - Authentication/authorization during signature verification, this may be due to external calls made by the plugin.
* TIMEOUT - The verification process timed out and can be retried by Notation.
* THROTTLED - The verification process was throttled and can be retried by Notation.
* ERROR - Any general error that does not fall into previous error categories.

## FAQ

**Q: Will Notation generate timestamp countersignature for Signature Envelope Generator plugin or its responsibility of plugin publisher?**

**A :** The Signature Envelope Generator plugin has full control over the generated signature envelope including any timestamp countersignature. Notation will NOT add/update/delete any timestamp countersignature in an envelope that's generated by a Signature Envelope Generator plugin.