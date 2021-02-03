# Sunrise Choir Rust SSB implementation

## Introduction

The code for the Sunrise Choir Rust SSB implementation is split between many repositories which can be found on their [GitHub organization page](https://github.com/sunrise-choir). Many of the crates have excellent low-level documentation in the form of HTML pages generated from documentation comments in the source code. Links for crate-specific documentation can usually be found in the crate repository READMEs. Further information about the team and their work can be found on the [Sunrise Choir website](https://sunrisechoir.com/) or [OpenCollective page](https://opencollective.com/sunrise-choir).

If you are not familiar with the [Rust programming language](https://www.rust-lang.org/), please visit the [Getting started](https://www.rust-lang.org/learn/get-started) page to learn more. The [Rust programming language book](https://doc.rust-lang.org/book/) and [Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/) are both excellent learning guides.

## SSB-Keyfile

[SSB-Keyfile](https://github.com/sunrise-choir/ssb-keyfile) provides basic utilities to create and read keyfiles that are compatible with the JS SSB implementation. This module reexports the [`Keypair` struct](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/struct.Keypair.html) and related methods from the [ssb-crypto](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/index.html) module. As with most of the Sunrise Choir Rust SSB modules, SSB-Keyfile can either be imported into another project as a library or compiled into a standalone binary which takes parameters as arguments.

```rust
use ssb_keyfile::Keypair;
```

The most basic operation is generating keys: this produces an identity encoded as a `struct` with two fields: `public` (of type [`PublicKey`](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/struct.PublicKey.html)) and `secret` (of type [`SecretKey`](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/struct.SecretKey.html)). We'll call this identity **Aruna**.

```rust
let aruna = Keypair::generate();

println!("{}", aruna.as_base64());
// vS8uDOK1nU+y3Oxskhfta7AzjbNQ70IpFyoVpzH/IPCzA2COB7U1s7c7NE/2DPjcky67YgZHInOhNWtmqnXVrw==

println!("{}", aruna.public.as_base64());
// swNgjge1NbO3OzRP9gz43JMuu2IGRyJzoTVrZqp11a8=
```

Alternatively, we can load or create a key that's saved in a file. These methods allow compatibility with keyfiles conforming to the Javascript implementation. We'll call this identity **Benedict**.

```rust
use ssb_keyfile::{Keypair, KeyFileError};

// create a keypair and write to file at the given path
let benedict = ssb_keyfile::generate_at_path("/home/benedict/.ssb/secret")?;

println!("{:#?}", benedict);
// Keypair {
//     secret: SecretKey([90, 172, 77, 159, 123, 231, 130, 138, 234, 241, 16, 19, 155, 252, 232, 137, 180, 246, 152, 184, 239, 94, 140, 159, 86, 167, 124, 145, 105, 158, 129, 253]),
//     public: PublicKey([28, 74, 178, 247, 141, 19, 234, 224, 126, 79, 231, 125, 37, 166, 185, 241, 163, 95, 71, 50, 241, 245, 228, 86, 170, 70, 101, 140, 25, 167, 146, 105])
// }

// read a keypair from file
let benedict = ssb_keyfile::read_from_path("/home/benedict/.ssb/secret")?;
```

### Signing

The most common operation with our keys is signing objects and then verifying their signatures. SSB has a very specific message encoding format, but for this example we'll use an example object.

```rust
// create an example object with json encoding (using serde_json)
let example_object = json!({ "type": "example" });
```

We'll use Aruna's private key to sign this object. Since the `keypair.sign()` method takes a byte slice, we convert the JSON object to a `String` and then a byte slice (`[u8]`).

```rust
let example_string = example_object.to_string();

let signature = aruna.sign(&example_string.as_bytes());

println!("{:?}", signature.as_base64());
// "o5w5E3Kt2AmJlwx4mocLz4V622m5Y0C3jirs1xTDoHdxvAXbfZMMJ7PfEE5ZV6eLWpI3HyhtmiqyUlRuMSZsBw=="

// note: notice that the output does not include a ".ed25519" suffix or a sigil prefix
```

### Verification

Now when we compare the signature against Aruna's public key, we'll see that the signature is valid and that Aruna must have signed this message. This is important for ensuring that an object hasn't been tampered with. Any change to the object will invalidate the signature.

```rust
let is_valid = aruna.public.verify(&signature, &example_string.as_bytes());

println!("{}", is_valid);
// true
```

If we try to do the same verification with Benedict's public key, we'll see that the signature is **not valid**. Either the public key is incorrect or the object has been tampered with, but the signature is invalid.

```rust
let is_valid = benedict.public.verify(&signature, &example_string.as_bytes());

println!("{}", is_valid);
// false
```

## Cryptography

The object signing illustrated above is made possible by the [ssb_crypto](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/index.html) crate:

> This crate provides the cryptographic functionality needed to implement the Secure Scuttlebutt networking protocols and content signing and encryption.

> There are two implementations of the crypto operations available; one that uses [libsodium](https://libsodium.gitbook.io/) C library (via the [sodiumoxide](https://crates.io/crates/sodiumoxide) crate), and a pure-rust implementation that uses [dalek](https://dalek.rs/) and [RustCrypto](https://github.com/RustCrypto/) crates (which is the default). You can select which implementation to use via Cargo.toml feature flags (see documentation - link above).

### Verification

A valid signature isn't the only constraint that SSB messages have to meet. The [ssb-verify-signatures](https://github.com/sunrise-choir/ssb-verify-signatures) crate allows us to verify the signature of an entire SSB message. It also allows the verification of multiple messages in parallel.

Let's first attempt to verify a message we know to be invalid. This message contains a `key` field and nothing else.

```rust
let key_only_message = r##"{ "key": "%kmXb3MXtBJaNugcEL/Q7G40DgcAkMNTj3yhmxKHjfCM=.sha256" }"##;

// attempt to verify the signature of the key_only_message
match ssb_verify_signatures::verify_message(key_only_message.as_bytes()) {
    Ok(_) => println!("verified"),
    Err(e) => eprintln!("{}", e)
};
// Error parsing ssb message as json, it is invalid. Errored with: missing field `value` at line 1 column 65
```

Thankfully, the `verify_message()` method returns a very helpful error message to explain why the given message could not be verified. The `ssb_verify_signatures::Error` type is an `enum` with 11 variants. You can read the full list of variants in the [Rust docs for the custom Error type](https://sunrise-choir.github.io/ssb-verify-signatures/ssb_verify_signatures/enum.Error.html).

Now let's try to verify a message that conforms to the required specification.

```rust
let valid_message = r##"{
    "key": "%kmXb3MXtBJaNugcEL/Q7G40DgcAkMNTj3yhmxKHjfCM=.sha256",
    "value": {
        "previous": "%IIjwbJbV3WBE/SBLnXEv5XM3Pr+PnMkrAJ8F+7TsUVQ=.sha256",
        "author": "@U5GvOKP/YUza9k53DSXxT0mk3PIrnyAmessvNfZl5E0=.ed25519",
        "sequence": 8,
        "timestamp": 1470187438539,
        "hash": "sha256",
        "content": {
            "type": "contact",
            "contact": "@ye+QM09iPcDJD6YvQYjoQc7sLF/IFhmNbEqgdzQo3lQ=.ed25519",
            "following": true,
            "blocking": false
        },
        "signature": "PkZ34BRVSmGG51vMXo4GvaoS/2NBc0lzdFoVv4wkI8E8zXv4QYyE5o2mPACKOcrhrLJpymLzqpoE70q78INuBg==.sig.ed25519"
    },
    "timestamp": 1571140551543
}"##;

match ssb_verify_signatures::verify_message(valid_message.as_bytes()) {
    Ok(_) => println!("verified"),
    Err(e) => eprintln!("{}", e)
};
// verified
```

TODO: verify multiple messages in parallel

### Validation

```rust
// let is_valid = validate_message_hash_chain::<_, &[u8]>(&msg, None).is_ok();
```

## Contact

The Sunrise Choir consists of [@Mikey](https://github.com/ahdinosaur), [@Piet](https://github.com/pietgeursen), [@Matt](https://github.com/mmckegg) and [@Sean](https://github.com/sbillig). This documentation was compiled by [@glyph](https://github.com/mycognosist). All can be found in the Scuttleverse:

* mikey: `@6ilZq3kN0F+dXFHAPjAwMm87JEb/VdB+LC9eIMW3sa0=.ed25519`
* piet: `@U5GvOKP/YUza9k53DSXxT0mk3PIrnyAmessvNfZl5E0=.ed25519`
* matt: `@FbGoHeEcePDG3Evemrc+hm+S77cXKf8BRQgkYinJggg=.ed25519`
* sean: `@N/vWpVVdD1e8IbACUQE4EVGL6+aodQfbQZ8ByC+k79s=.ed25519`
* glyph: `@HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519`
