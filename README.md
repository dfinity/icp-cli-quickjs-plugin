# icp-js-plugin

An [icp-cli](https://github.com/dfinity/icp-cli) **sync plugin** that runs a
JavaScript script against the canister being synced. It implements the
`icp:sync-plugin` WIT world (see [`sync-plugin.wit`](sync-plugin.wit)) and
exposes to the script roughly the same capabilities a native sync plugin has —
calling the target canister, reading its metadata, setting its environment
variables, the sync inputs, and read-only filesystem access — plus Candid,
principal, and encoding helpers convenient for canister work.

Scripts run on [QuickJS](https://bellard.org/quickjs/) via
[rquickjs](https://crates.io/crates/rquickjs); it is a small ES2020-class engine
without Node or Web APIs, so no `require`/`import`, no `fetch`, no timers — just
the language plus the host functions the plugin provides.

**[API.md](./API.md) documents the scripting API in full.**

## Building

The plugin is a WebAssembly component targeting `wasm32-wasip2`:

```sh
rustup target add wasm32-wasip2
cargo build --target wasm32-wasip2 --release
```

The component is emitted at `target/wasm32-wasip2/release/icp_js_plugin.wasm`.

The crate also builds for the host, so `cargo check` and `cargo test` run
without a WebAssembly runtime.

## Using it

Declare the plugin as a sync step, with the entry script under the `script` key
(or inline in a `script` field). Any other files declared are read by the host
and handed to the script by path; directories under `dirs:` are preopened
read-only.

```yaml
sync:
  steps:
    - plugin: ./icp_js_plugin.wasm
      canisters: [ledger]
      files:
        script: sync.js
        config: config.json
```

A script runs to completion for a clean sync; throwing fails the step with the
thrown message.

## Examples

Push a list of authorized principals from a JSON file to a sibling canister:

```js
// { "authorized": ["aaaaa-aa", "ryjl3-tyaaa-aaaaa-aaaba-cai"] }
const config = JSON.parse(files["config.json"]);

// `set_authorized : (vec principal) -> ()`. The strings out of the file are
// wrapped in `Principal.from(..)`, which both validates them and tells the encoder
// they are principals rather than text.
const authorized = config.authorized.map((p) => Principal.from(p));
callUpdate("example", "set_authorized", candid`(${authorized})`);
```

Or the same call written against the canister's own interface, which turns the
strings into principals itself:

```js
const config = JSON.parse(files["config.json"]);
callTyped("example", "set_authorized", config.authorized);
```

Bump a counter on the canister being synced and report the new value:

```js
console.error("count before sync: " + callTyped(self, "get"));

callTyped(self, "increment");

console.error("count after sync: " + callTyped(self, "get"));
```

## What a script gets

Each of these is covered in [API.md](./API.md):

| | |
| --- | --- |
| [Sync inputs](./API.md#sync-inputs-globals) | `canisterId`, `identity`, `environment`, `proxy`, `files`, `dirs`, `fields`, `canisterIds` and friends, as globals. |
| [Canister calls](./API.md#canister-calls) | `callQuery` / `callUpdate` / `canisterCall`, against the synced canister or any canister the step declared. |
| [Coerced calls](./API.md#coerced-calls) | `callTyped` / `canisterCallTyped` and `CandidInterface`, which encode and decode against the callee's own `.did`. |
| [Candid](./API.md#candid) | The `candid` template tag, `CandidArgs`, `candidEncode` / `candidDecode`, the [number types](./API.md#number-types), and the [exact-encoding classes](./API.md#exact-encoding-classes) for variants, optionals, tuples and references. |
| [Metadata sections](./API.md#metadata-sections) | `canisterMetadata`, reading a canister's custom sections. |
| [Environment variables](./API.md#environment-variables) | `canisterSetenv`, setting one runtime variable on a canister. |
| [Principals](./API.md#principals) | The `Principal` class of [icp-js-core](https://github.com/dfinity/icp-js-core). |
| [Helpers](./API.md#encoding-helpers) | `sha256`, `encodeUtf8` / `decodeUtf8`, and [`randomBytes`](./API.md#randomness). |
| [Filesystem](./API.md#filesystem) | Read-only reads, predicates and `joinPath` over the declared `dirs:`. |
| [Output](./API.md#output) | `print` / `eprint` and the `console` methods. |

## License

This project is licensed under the [Apache-2.0](./LICENSE) license.

## Contribution

This project does not accept external contributions. Pull requests from individuals outside the organization will be automatically closed.
