# Notation Extensibility for Signing and Verification

Keys and associated certificates used for signing artifacts using Notary could be available to users through varied solutions that provide secure key generation, storage and cryptographic operations. Some are well established with standards like [PIV](https://csrc.nist.gov/projects/piv/piv-standards-and-supporting-documentation) and [PKCS #11](https://docs.oasis-open.org/pkcs11/pkcs11-base/v3.0/os/pkcs11-base-v3.0-os.html) implemented by hardware tokens, smart cards. More recent options which use varied authentication and API protocols are remote key management services and signing services by third party vendors and cloud service providers. Notation will support a few built-in integrations with standard providers, and will provide plugin interfaces for users, and vendors to implement their own integrations with the solutions they use. This allows a plugin publisher to implement, test, release and patch their solutions independent of Notation’s development and release cycle. This document provides specification for the plugin model, and interfaces to implement. This specification aims to work both for [existing](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md) and future signature formats adopted by Notary.

## Terminology

* **Plugin Publisher** - A user, organization, open source project or 3rd party vendor that creates a Notation plugin for internal or public distribution.
* **Plugin** - A component external to Notation that can integrate as one of the steps in  Notation’s workflow for signature generation or verification.
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

Notation will invoke plugins as executable, pass parameters using command line arguments, and use standard IO streams to pass request/response payloads. This mechanism is used as Go language (used to develop [Notation library](https://github.com/notaryproject/notation-go-lib)) does not have a [in built support](https://github.com/golang/go/issues/19282) to load and execute plugins that works across OS platforms. Other mechanisms like gRPC require every plugin to be implemented as a service/daemon.

### Plugin lifecycle management

#### Installation

Plugin publisher will provide instructions to download and install the plugin. Plugins intended for public distribution should also include instructions for users to verify the authenticity of the plugin.

**Open Item** : [Plugin install paths](https://github.com/notaryproject/notation/issues/167)

To enumerate all available plugins the following paths are scanned:
* Unix-like OSes:
  * `$HOME/notation/plugins`
* On Windows:
  * `%USERPROFILE%\notation\plugins`

Each plugin executable and dependencies are installed under directory `~/notation/plugins/{plugin-name}` with an executable under that directory `~/notation/plugins/{plugin-name}/notation-{plugin-name}`.

Any directory found inside `~/notation/plugins` is considered potential plugin "candidates". Anything found which is not a directory is ignored and is not considered as a plugin candidate.

To be considered a valid plugin a candidate must pass each of these "plugin candidate tests":

* The directory must contain an executable named `notation-{plugin-name}`.
* The executable MUST be a regular file, symlinks are not supported. Implementation MUST validate that the executable is a regular file, before executing it, and fail it it does not meet this condition.
* On Windows, executables must have a `.exe` suffix.
* Must, where relevant, have appropriate OS "execute" permissions (e.g. Unix x bit set) for the current user.
* Must actually be executed successfully and when executed with the subcommand `get-plugin-metadata` must produce a valid JSON metadata (and nothing else) on its standard output (schema to be discussed later).

#### Commands

* List

`notation plugin list`

List all valid plugins.

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
            "key1" : "value1" 
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

In the previous example, plugin config can be set using command line arguments as part of registring a key, or while signing using the key. Plugin config values provided by `notation sign` override values already set in `config.json` E.g.

* `notation key add --name "mysigningkey" --id "keyid" --plugin "com.example.nv2plugin" --pluginConfig.key1 "value1"`
* `notation sign $IMAGE --key "mysigningkey" --pluginConfig.key1 "value2"`

### Plugin contract

* Notation will invoke the plugin executable for each command (e.g. sign, verify), pass inputs through `stdin` and get output through `stdout` and `stderr`.
* The command will be passed as the first argument to the plugin e.g. `notary-{plugin-name} <command>`. A JSON request is passed using `stdin`. The plugin is expected to return a JSON response through `stdout` with a `0` exit code for successful response, and a non-zero exit code with a JSON error response in `stderr` for error response. Each command defines its request, response and error contract. To avoid any additional content like debug or info level logging from dependencies and inbuilt libraries, the plugin implementation should redirect any output to `stdout` on initialization, and only send the JSON response away from `stdout` when the command execution completes. E.g. For golang, set [`os.Stdout`](https://pkg.go.dev/os#pkg-variables) to point to a log file.
* Every request JSON will contain a `contractVersion` top level attribute whose value will indicate the plugin contract version. Contract version is revised when there are changes to command request/response, new plugin commands are introduced, and supported through Notation.
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

*Request* - None

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
  
  // Currently one of 
  // SIGNATURE_GENERATOR or
  // SIGNATURE_ENVELOPE_GENERATOR
  "capabilities" : [
    
  ]
}
```

*plugin-name* - Plugin name uses reverse domain name notation to avoid plugin name collisions.

*supported-contract-versions* - The list of contract versions supported by the plugin. Currently this list must include only one version, per major version. Post initial release, Notation may add new features through plugins, in the form of new commands (e.g. tsa-sign for timestamping), or additional request and response parameters. Notation will publish updates to plugin interface along with appropriate contract version update. Backwards compatible changes (changes for which older version of plugin continue to work with versions of Notation using newer contract version) like new optional parameters on existing contracts, and new commands will be supported through minor version contract updates, breaking changes through major version updates. Plugin `get-plugin-metadata` command returns the contract version a plugin supports. Notation will evaluate the minimum plugin version required to satisfy a user's request, and reject the request if the plugin does not support the required version.

*capabilities* - Non empty list of features supported by a plugin. Each capability such as `SIGNATURE_ENVELOPE_GENERATOR` requires one of more commands to be implemented by the plugin. When new features are available for plugins to implement, an implementation may choose to not implement it, and therefore will not include the feature in capabilities. Notation will evaluate the capability required to satisfy a user’s request, and reject the request if the plugin does not support the required capability.

## Signing interfaces

### Requirements

* The interface MUST NOT be tied to a specific signature envelope format.

Notation will support plugins to be developed against the following interfaces - *Signature Generator*, and *Signature Envelope Generator*. These interfaces target abstraction levels that satisfy most plugin integration scenarios.

### Signature Generator

This interface targets plugins that integrate with providers of basic cryptographic operations. E.g. Local PIV/PKCS#11 hardware tokens, remote KMS, or key vault services. Plugins that target this interface will only generate a raw signature given a payload to sign. Notation will package this signature into a signature envelope, and generate the signature manifest. Notation will also generate the TSA signature if indicated by the user. The plugin does not need to be signature envelope format aware, and will continue to work if Notary adopts additional signature formats.

#### Signing workflow using plugin

1. Given a user request to sign `oci-artifact`, with signing key `keyName` (the friendly key name)
2. Pull the image manifest using `oci-artifact` url, and construct a descriptor
3. Append any user provided metadata and Notary metadata as descriptor annotations.
4. Determine if the registered key uses a plugin
5. Execute the plugin with `get-plugin-metadata` command
    1. If plugin supports capability `SIGNATURE_GENERATOR`
        1. Execute the plugin with `describe-key` command, set `request.keyName`, `request.keyId` and the optional `request.pluginConfig` to corresponding values associated with signing key `keyName` in `config.json`.
        2. Generate the payload to be signed for [JWS](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md#supported-signature-envelopes) envelope format.
           1. Create the protected headers collection and set `alg` to value corresponding to `describe-key.response.keySpec` as per [signature algorithm selection](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md#algorithm-selection).
           2. Create the `JWSPayload` with appropriate private (`subject`) and public (`iat,exp`) claims.
           3. The payload to sign is then created as - `ASCII(BASE64URL(UTF8(ProtectedHeaders)) ‘.’ BASE64URL(JWSPayload))`
        3. Execute the plugin with `generate-signature` command. Set `request.keyName`, `request.keyId` and the optional `request.pluginConfig` to corresponding values associated with signing key `keyName` in `config.json`. Set `request.payload` as the payload to sign.
        4. Validate the generated signature, return an error if any of the checks fails.
           1. Check if `response.signingAlgorithm` is one of [supported signing algorithms](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md#algorithm-selection).
           2. Check that the plugin did not modify `request.payload` before generating the signature, and the signature is valid for the given payload. Verify the hash of the `request.payload` against `response.signature`, using the public key of signing certificate (leaf certificate) in `response.certificateChain` along with the `response.signingAlgorithm`. This step does not include certificate chain validation (certificate chain leads to a trusted root configured in Notation's Trust Store), or revocation check.
           3. Check that the `response.certificateChain` conforms to [Certificate Requirements](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md#certificate-requirements).
        5. Assemble the JWS Signature envelope using `response.signature`, `response.signingAlgorithm` and `response.certificateChain`. Notation may also generate and include timestamp signature in this step.
        6. Generate a signature manifest for the given signature envelope.
    2. Else if, plugin supports capability `SIGNATURE_ENVELOPE_GENERATOR` *(covered in next section)*
    3. Return an error

#### describe-key

This command is used to get metadata for a given key.

*Request*

```jsonc
{
  "contractVersion" : "<major-version.minor-version>",

  // Key name, key id for the key associated with the plugin
  // in config.json /signingKeys/keys
  "keyName": "<key name>",
  "keyId": "<key id>",

  // Optional plugin configuration, map of string-string
  "pluginConfig" : { }
}
```

*keyName* and *keyId* : Required fields that contain the friendly signing key name (`keyName`) and key identifier (`keyId`) as specified in `config.json`.

*pluginConfig* : Optional field for plugin configuration. For details, see [Plugin Configuration](#plugin-configuration) section.

*Response*

```jsonc
{
   // The same key id as passed in the request.
  "keyId" : "<key id>",
  "keySpec" : "<key type and size>"
}
```

*keySpec* : One of following [supported key types](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md#algorithm-selection) - `RSA_2048`, `RSA_3072`, `RSA_4096`, `EC_256`, `EC_384`, `EC_512`.

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

  // Key name, key id for the key associated with the plugin
  // in config.json /signingKeys/keys
  "keyName": "<key name>",
  "keyId": "<key id>",

  // Optional plugin configuration, map of string-string
  "pluginConfig" : { },

  // The signature envelope type associated with the payload.
  "signatureEnvelopeType" : "application/vnd.cncf.notary.v2.jws.v1",
  
  "payload" : "<payload to be signed>"
}
```

*keyName* and *keyId* : Required fields that contain the friendly key name (`keyName`), key identifier (`keyId`) as specified in `config.json`.

*pluginConfig* : Optional field for plugin configuration. For details, see [Plugin Configuration section](#plugin-configuration).

*signatureEnvelopeType* - The signature envelope type associated with the payload. This is used by the plugin to appropriately decode the payload in case a binary payload is passed.

*payload* : Required field that contains payload to be signed. For JWS envelope type, the payload to sign is sent as is, without any additional encoding.

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

*signingAlgorithm* : One of following [supported signing algorithms](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md#algorithm-selection), Notation uses this validate the signature, and to set the appropriate attribute in signature envelope (e.g. JWS `alg`). `RSASSA_PSS_SHA_256`, `RSASSA_PSS_SHA_384`, `RSASSA_PSS_SHA_512`,  `ECDSA_SHA_256`, `ECDSA_SHA_384`, `ECDSA_SHA_512`.

*certificateChain* : Ordered list of certificates starting with leaf certificate and ending with root certificate.

#### Error codes for describe-key and generate-signature

* VALIDATION_ERROR - Any of the required request fields was empty, or a value was malformed/invalid. Includes condition where the key referenced by `keyId` was not found.
* UNSUPPORTED_CONTRACT_VERSION - The contract version used in the request is unsupported.
* ACCESS_DENIED - Authentication/authorization error to use given key.
* TIMEOUT - The operation to generate signature timed out and can be retried by Notation.
* THROTTLED - The operation to generate signature was throttles and can be retried by Notation.
* ERROR - Any general error that does not fall into previous error categories.

### Signature Envelope Generator

This interface targets plugins that in addition to signature generation want to generate the complete signature envelope. This interface allows plugins to have full control over the generated signature envelope, and can append additional signed and unsigned metadata, including timestamp signatures. The plugin must be signature envelope format aware, and implement new formats when Notary adopts new formats.

#### Signing workflow using plugin

1. Given a user request to sign `image`, with `keyName` (friendly key name)
1. Pull the image manifest using `image` url, and construct a descriptor
1. Append any user provided metadata and Notary metadata as descriptor annotations.
1. Determine if the registered key uses a plugin
1. Execute the plugin with `get-plugin-metadata` command
    1. If plugin supports capability `SIGNATURE_ENVELOPE_GENERATOR`
        1. Execute the plugin with `generate-envelope` command. Set `request.keyName`, `request.keyId` and the optional `request.pluginConfig` to corresponding values associated with signing key `keyName` in `config.json`. Set `request.payload` to base64 encoded descriptor, `request.payloadType` to `application/vnd.oci.descriptor.v1+json` and `request.signatureEnvelopeType` to a pre-defined type (default to `application/vnd.cncf.notary.v2.jws.v1`).
        1. `response.signatureEnvelope` contains the base64 encoded signature envelope, value of `response.signatureEnvelopeType` MUST match `request.signatureEnvelopeType`.
        1. Validate the generated signature, return an error if of the checks fails.
           1. Check if `response.signatureEnvelopeType` is a supported envelope type and `response.signatureEnvelope`'s format matches `response.signatureEnvelopeType`. 
           1. Check if the signing algorithm in the signature envelope is one of [supported signing algorithms](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md#algorithm-selection).
           1. Check that the [`subject` descriptor](https://github.com/notaryproject/notaryproject/blob/main/signature-specification.md#supported-signature-envelopes) in JWSPayload in `response.signatureEnvelope` matches `request.payload`. Plugins MAY append additional annotations but MUST NOT replace/override existing descriptor attributes and annotations.
           1. Check that `response.signatureEnvelope` can be verified using the public key and signing algorithm specified in the signing certificate, which is embedded as part of certificate chain in `response.signatureEnvelope` . This step does not include certificate chain validation (certificate chain leads to a trusted root configure in Notation), or revocation check.
           1. Check that the certificate chain in `response.signatureEnvelope` confirm to [Certificate Requirements].
        1. Generate a signature manifest for the given signature envelope, and append `response.annotations` to manifest annotations.
    1. Else if plugin supports capability `SIGNATURE_GENERATOR` *(covered in previous section)*
    1. Return an error

#### generate-envelope

This command is used to generate the complete signature envelope for a given payload.

*Request*
All request attributes are required.

```jsonc
{
  "contractVersion" : "<major-version.minor-version>",
  
  // Key name, key id for the key associated with the plugin
  // in config.json /signingKeys/keys
  "keyName": "<key name>",
  "keyId": "<key id>",

  // Optional plugin configuration, map of string-string
  "pluginConfig" : { },

  "payload" : "<Base64 encoded payload to be signed>",
  
  // The type of payload - currently a descriptor
  "payloadType" : "application/vnd.oci.descriptor.v1+json",
  
  // The expected response signature envelope
  "signatureEnvelopeType" : "application/vnd.cncf.notary.v2.jws.v1"
}
```

*keyName* and *keyId* : Required fields that contain the friendly key name (`keyName`), key identifier (`keyId`) as specified in `config.json`.

*pluginConfig* : Optional field for plugin configuration. For details, see [Plugin Configuration section](#plugin-configuration).

*signatureEnvelopeType* - defines the type of signature envelope expected from the plugin. As  Notation clients need to be updated in order to parse and verify new signature formats, the default signature format can only be changed with new major version releases of Notation. Users however can opt into using an updated signature format supported by Notation, by passing an optional parameter.
e.g. `notation sign $IMAGE --key {key-name} --signatureFormat {some-new-format}`

*Response*
All response attributes are required.

```jsonc
{
   "signatureEnvelope": "<Base64 encoded signature envelope>",
   "signatureEnvelopeType" : "application/vnd.cncf.notary.v2.jws.v1",
   
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

TBD [notaryproject/roadmap#27](https://github.com/notaryproject/roadmap/issues/27)

## Threat Model

TBD [#135](https://github.com/notaryproject/notaryproject/issues/135)

## FAQ

**Q: Will Notation generate timestamp signature for Signature Envelope Generator plugin or its responsibility of plugin publisher?**

**A :** If the envelope generated by a Signature Envelope Generator plugin contains timestamp signature, Notation will not append additional timestamp signature, else it will generate the timestamp signature and append it to the envelope as an unsigned attribute.

## Open Items

* [Issue #151](https://github.com/notaryproject/notation/issues/151) - Add Notation command line options to pass raw signature generated by existing crypto tools.
* What standard providers should be supported?
* Support for chaining plugins. It allows us to separate out and compose things like signing, TSA integration, push to transparency log.
