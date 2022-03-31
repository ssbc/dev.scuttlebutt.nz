## Kuska implementation

### Introduction

[Kuska-ssb](https://github.com/Kuska-ssb) implements a subset of the existing JavaScript functionality. The majority of the code lives in the [ssb](https://github.com/Kuska-ssb/ssb) repository, with a separate [handshake](https://github.com/Kuska-ssb/handshake) repository containing the Secure Scuttlebutt handshake and boxstream library.

If you are not familiar with the [Rust programming language](https://www.rust-lang.org/), please visit the [Getting started](https://www.rust-lang.org/learn/get-started) page to learn more. The [Rust programming language book](https://doc.rust-lang.org/book/) and [Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/) are both excellent learning guides.

### Keystore

The [keystore modules](https://github.com/Kuska-ssb/ssb/tree/master/src/keystore) are used to create, load, read and write SSB keys.

```rust
use kuska_ssb::keystore;
use kuska_ssb::keystore::OwnedIdentity;
```

The most basic operation is generating keys: this produces an identity `struct` of type `OwnedIdentity`. We'll call this identity **Ayami**.

```rust
// struct OwnedIdentity {
//     id: String,
//     pk: ed25519::PublicKey,
//     sk: ed25519::SecretKey,
// }

let ayami = OwnedIdentity::create();
// OwnedIdentity {
//     id: "@/aCKS2hXOE1PbzwOThkXumZF3+Jlka6FBkQrln0EewI=.ed25519",
//     pk: PublicKey([253, 160, 138, 75, 104, 87, 56, 77, 79, 111, 60, 14, 78, 25, 23, 186, 102, 69, 223, 226, 101, 145, 174, 133, 6, 68, 43, 150, 125, 4, 123, 2]),
//     sk: SecretKey(****),
// }
```

Alternatively, we can load or create a key that's saved in a file. These methods allow compatibility with keyfiles conforming to the JavaScript implementation. Loading the local secret produces an identity `struct` of type `OwnedIdentity`. We'll call this identity **Bongani**.

```rust
// read from "{}/.ssb/secret" where "{}" is home directory
let bongani = keystore::from_patchwork_local().await.expect("read local secret");
```

We can also write this identity to file:

```rust
// create test file for storing keys
let mut file = File::create("bongani").await?;

// write identity to file
keystore::write_patchwork_config(&bongani, &mut file).await.expect("write local secret");
```

#### Signing

The most common operation with our keys is signing objects and then verifying their signatures. SSB has a very specific message encoding format, but for this example we'll use an example object.

```rust
// create an example object with json encoding (using serde_json)
let example_object = json!({ "type": "example" });
```

We'll use Ayami's private key to sign this object.

```rust
use kuska_ssb::feed::Message;

// returns a `Result<Message>`
let signed_object = Message::sign(None, &ayami, example_object);
```

The `Message` struct implements various convenience methods for retrieving data. For example, we can retrieve the contents and signature of the Message.

```rust
let object = signed_object.unwrap();

object.content();
// Object(
//     {
//         "type": String(
//             "example",
//         ),
//     },
// )

object.signature();
// "Ya6RkIDJDRh7UE1tJlpJ7AlpcEVeMjjmEzCm3WCy4dHWJysGYJS5dkWvsJ3xphXrVE61Yqv+dXNPLv8ypzpiAg==.sig.ed25519"
```

#### Verification

In order to verify a signed object, Kuska-ssb implements a `from_value` method on the `Message` struct. The method takes a single argument in the form of a JSON object. The first action performed by the method is to verify that each of the required message fields exists and that the type of the value for each field is correct. Checked fields include: `previous`, `sequence`, `timestamp`, `hash` and `content`.

The `signature` and `author` fields are then checked before the signature is verified.

```rust
Message::from_value(object)?;
```

In the case of a successful validation, an `Ok` `Result` type is returned with contents of type `Message`:

```rust
Ok(Message {
    value: Value::Object(v),
})
```

In the case of an error, an `Error` enum is returned which defines the cause of the error. Example errors include:

```rust
Error::InvalidJson
Error::InvalidSignature
```

See [`src/feed/error.rs`](https://github.com/Kuska-ssb/ssb/blob/90017a31fa8789e548347bb205e96be8fc9351c7/src/feed/error.rs) for the complete `Error` listing.

### Sodium Oxide

The cryptographic functionality of Kuska-ssb is provided by a fork of [sodiumoxide](https://github.com/Kuska-ssb/sodiumoxide), a type-safe and efficient Rust binding for libsodium. The best way to understand this API is to read the [crate documentation](https://docs.rs/sodiumoxide/0.2.6/sodiumoxide/). Of particular interest is the [documentation of the crypto module](https://docs.rs/sodiumoxide/0.2.6/sodiumoxide/crypto/index.html), which contains the cryptography methods that form the basis of the [sodium.rs crypto module](https://github.com/Kuska-ssb/ssb/blob/master/src/crypto/sodium.rs), [message module](https://github.com/Kuska-ssb/ssb/blob/master/src/feed/message.rs) and others.

### TODO

Feeds. Peers. Replication. etc.

### Contact

Kuska-ssb was written by [@adria0](https://github.com/adria0) with cryptographic support from [@Dhole](https://github.com/Dhole). This documentation was compiled by [@glyph](https://github.com/mycognosist). All three can be found in the Scuttleverse:

* adria0: `@ZFWw+UclcUgYi081/C8lhgH+KQ9s7YJRoOYGnzxW/JQ=.ed25519`
* Dhole: `@TnZv3sRwVxtpZm07a7jTFMAtfIbCY1EUEcftwU/XDaI=.ed25519`
* glyph: `@HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519`
