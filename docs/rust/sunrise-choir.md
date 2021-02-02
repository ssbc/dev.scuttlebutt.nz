# Sunrise Choir Rust SSB implementation

## Introduction

The code for the Sunrise Choir Rust SSB implementation is split between many repositories which can be found on their [GitHub organization page](https://github.com/sunrise-choir). Further information about the team and their work can be found on the [Sunrise Choir website](https://sunrisechoir.com/) or [OpenCollective page](https://opencollective.com/sunrise-choir).

If you are not familiar with the [Rust programming language](https://www.rust-lang.org/), please visit the [Getting started](https://www.rust-lang.org/learn/get-started) page to learn more. The [Rust programming language book](https://doc.rust-lang.org/book/) and [Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/) are both excellent learning guides.

## SSB-Keyfile

[SSB-Keyfile](https://github.com/sunrise-choir/ssb-keyfile) provides basic utilities to create and read keyfiles that are compatible with the JS SSB implementation. This module reexports the [`Keypair` struct](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/struct.Keypair.html) and related methods from the [ssb-crypto](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/index.html) module.

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

TODO
