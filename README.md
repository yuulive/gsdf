# Ye

[![crates.io](https://img.shields.io/crates/v/ye.svg)](https://crates.io/crates/ye)
[![Documentation](https://docs.rs/ye/badge.svg)](https://docs.rs/ye)
[![Apache-2 licensed](https://img.shields.io/crates/l/ye.svg)](./LICENSE.txt)

## About

[`Ye`] is a framework for working with VCs and DIDs from different providers and on different platforms in a constant manner.
Even if the actual implementation and logic behind them may change, [`Ye`] offers a consistent interface to work with them.
It has been developed with [wasm support] in mind to allow it not only to run on servers but also on different clients
with limited resources like IoT devices.

The name "Ye" is an acronym for "VC and DID engine" and focuses on working with VCs and DIDs. It has been designed with the idea of offering a consistent interface to work with while supporting to move the actual work into plugins.

This library is currently under development. Behavior, as well as provided exports, may change over time.

Documentation about [`Ye`]s functions and their meaning can be found [`here`](https://docs.rs/ye/*/ye/struct.Ye.html).

## Plugins

[`Ye`] is relying on plugins to run interact provider specific logic. The current set of Plugins can be seen below:

### DID plugins

| Method | Info |
| ------ | ---- |
| did:evan | [![crates.io](https://img.shields.io/crates/v/ye-evan.svg)](https://crates.io/crates/ye-evan) |
| did:example | [ye-example-plugin](https://github.com/evannetwork/ye-example-plugin) |
| (universal resolver method list) | in development  |

More coming soon. To write your own plugins, have a look at [writing own plugins].

### VC plugins

| Method | Info |
| ------ | ---- |
| did:evan | [![crates.io](https://img.shields.io/crates/v/ye-evan.svg)](https://crates.io/crates/ye-evan) |

More coming soon. To write your own plugins, have a look at [writing own plugins].

## Example Usage

```rust
use ye::Ye;
// use some_crate:ExamplePlugin;
# use ye::YePlugin;
# struct ExamplePlugin { }
# impl ExamplePlugin { pub fn new() -> Self { ExamplePlugin {} } }
# impl YePlugin for ExamplePlugin {}

async fn example_ye_usage() {
    let ep: ExamplePlugin = ExamplePlugin::new();
    let mut ye = Ye::new();
    ye.register_plugin(Box::from(ep));

    match ye.did_create("did:example", "", "").await {
        Ok(results) => {
            let result = results[0].as_ref().unwrap().to_string();
            println!("created did: {}", result);
        },
        Err(e) => panic!(format!("could not create did; {}", e)),
    };
}
```

As you can see, an instance of `ExamplePlugin` is created and handed over to a [`Ye`] instance with [`register_plugin`]. To be a valid argument for this, `ExamplePlugin` needs to implement [`YePlugin`].

[`Ye`] delegates the call *all* functions with the same name as the functions of [`YePlugin`] to *all* registered plugins, so the result of such calls is a `Vec` of optional `String` values (`Vec<Option<String>>`).

## Basic Plugin Flow

Calls of plugin related functions follow the rule set described here:

- a [`Ye`] instance delegates **all** calls of plugin related functions to **all** registered plugins
- those [`YePlugin`] instances then may or may not process the request
- requests may be ignored due to not being implemented or due to ignoring them due to plugin internal logic (e.g. if a did method is not supported by the plugin, requests for this method are usually ignored)
- ignored plugin requests do not end up in the result `Vec`, so a [`Ye`] may have registered multiple plugins, but if only on plugin caters to a certain did method, calls related to this method will only yield a single result

![ye_plugin_flow](https://user-images.githubusercontent.com/1394421/85983296-8f3dd700-b9e7-11ea-92ee-47e8c441e576.png)

## Ye Features

The current set of features can be grouped into 3 clusters:

- management functions
- DID interaction
- zero knowledge proof VC interaction

### Management Functions

**[`register_plugin`]**

Registers a new plugin. See [`YePlugin`](https://docs.rs/ye/*/ye/struct.YePlugin.html) for details about how they work.

### DID Interaction

**[`did_create`]**

Creates a new DID. May also persist a DID document for it, depending on plugin implementation.

-----

**[`did_resolve`]**

Fetch data about a DID. This usually returns a DID document.

-----

**[`did_update`]**

Updates data related to a DID. May also persist a DID document for it, depending on plugin implementation.

### Zero Knowledge Proof VC Interaction

**[`vc_zkp_create_credential_schema`]**

Creates a new zero-knowledge proof credential schema. The schema specifies properties a credential
includes, both optional and mandatory.

-----

**[`vc_zkp_create_credential_definition`]**

Creates a new zero-knowledge proof credential definition. A credential definition holds cryptographic key material
and is needed by an issuer to issue a credential, thus needs to be created before issuance. A credential definition
is always bound to one credential schema.

-----

**[`vc_zkp_create_credential_proposal`]**

Creates a new zero-knowledge proof credential proposal. This message is the first in the
credential issuance flow.

-----

**[`vc_zkp_create_credential_offer`]**

Creates a new zero-knowledge proof credential offer. This message is the response to a credential proposal.

-----

**[`vc_zkp_request_credential`]**

Requests a credential. This message is the response to a credential offering.

-----

**[`vc_zkp_create_revocation_registry_definition`]**

Creates a new revocation registry definition. The definition consists of a public and a private part.
The public part holds the cryptographic material needed to create non-revocation proofs. The private part
needs to reside with the registry owner and is used to revoke credentials.

-----

**[`vc_zkp_update_revocation_registry`]**

Updates a revocation registry for a zero-knowledge proof. This step is necessary after revocation one or
more credentials.

-----

**[`vc_zkp_issue_credential`]**

Issues a new credential. This requires an issued schema, credential definition, an active revocation
registry and a credential request message.

-----

**[`vc_zkp_revoke_credential`]**

Revokes a credential. After revocation the published revocation registry needs to be updated with information
returned by this function.

-----

**[`vc_zkp_request_proof`]**

Requests a zero-knowledge proof for one or more credentials issued under one or more specific schemas.

-----

**[`vc_zkp_present_proof`]**

Presents a proof for a zero-knowledge proof credential. A proof presentation is the response to a
proof request.

-----

**[`vc_zkp_verify_proof`]**

Verifies a one or multiple proofs sent in a proof presentation.

### Custom Functions

**[`run_custom_function`]**

Calls a custom function. Plugins may subscribe to such custom calls, that are not part of the default set of [`Ye`]s default feature set, which allows to add custom plugin logic while using `Ye. Examples for this may be connection handling and key generation.

-----

Except for the management functions all functions will be delegated to plugins. Plugins handling follows the following rules:

- a [`Ye`] instance delegates **all** calls of plugin related functions to **all** registered plugins
- those [`YePlugin`] instances then may or may not process the request
- requests may be ignored due to not being implemented or due to ignoring them due to plugin internal logic (e.g. if a did method is not supported by the plugin, requests for this method are usually ignored)
- ignored plugin requests do not end up in the result `Vec`, so a [`Ye`] may have registered multiple plugins, but if only on plugin caters to a certain did method, calls related to this method will only yield a single result

## Writing own Plugins

Writing own plugin is rather simple, an example and details how to write them can be found in the [`YePlugin`] documentation.

## Wasm Support

Ye supports Wasm! ^^

For an example how to use [`Ye`] in Wasm and a how to guide, have a look at our [ye-wasm-example] project.

[`did_create`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.did_create
[`did_resolve`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.did_resolve
[`did_update`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.did_update
[`register_plugin`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.register_plugin
[`run_custom_function`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.run_custom_function
[`ye-evan`]: https://docs.rs/ye-evan
[`Ye`]: https://docs.rs/ye/*/ye/struct.Ye.html
[`YePlugin`]: https://docs.rs/ye/*/ye/trait.YePlugin.html
[`vc_zkp_create_credential_definition`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_create_credential_definition
[`vc_zkp_create_credential_offer`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_create_credential_offer
[`vc_zkp_create_credential_proposal`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_create_credential_proposal
[`vc_zkp_create_credential_schema`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_create_credential_schema
[`vc_zkp_create_revocation_registry_definition`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_create_revocation_registry_definition
[`vc_zkp_issue_credential`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_issue_credential
[`vc_zkp_present_proof`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_present_proof
[`vc_zkp_request_credential`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_request_credential
[`vc_zkp_request_proof`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_request_proof
[`vc_zkp_revoke_credential`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_revoke_credential
[`vc_zkp_update_revocation_registry`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_update_revocation_registry
[`vc_zkp_verify_proof`]: https://docs.rs/ye/*/ye/struct.Ye.html#method.vc_zkp_verify_proof
[ye-wasm-example]: https://github.com/evannetwork/ye-wasm-example
<!--
[wasm support]: https://docs.rs/ye/*/ye/#wasm-support
[writing own plugins]: https://docs.rs/ye/*/ye/#writing-own-plugins
-->
<!-- for Readme -->
[wasm support]: #wasm-support
[writing own plugins]: #writing-own-plugins
<!-- -->
