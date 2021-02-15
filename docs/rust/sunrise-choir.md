# Sunrise Choir Rust SSB implementation

## Introduction

The code for the Sunrise Choir Rust SSB implementation is split between many repositories which can be found on their [GitHub organization page](https://github.com/sunrise-choir). Many of the crates have excellent low-level documentation in the form of HTML pages generated from documentation comments in the source code. Links for crate-specific documentation can usually be found in the crate repository READMEs. Further information about the team and their work can be found on the [Sunrise Choir website](https://sunrisechoir.com/) or [OpenCollective page](https://opencollective.com/sunrise-choir).

If you are not familiar with the [Rust programming language](https://www.rust-lang.org/), please visit the [Getting started](https://www.rust-lang.org/learn/get-started) page to learn more. The [Rust programming language book](https://doc.rust-lang.org/book/) and [Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/) are both excellent learning guides.

A playground repository exists as a partner reference to this document: [Sunrise SSB Playground](https://github.com/mycognosist/sunrise-ssb-playground). The repo contains examples which can be run using the Rust compiler, and which demonstrate the features documented here.

## SSB-Keyfile

[SSB-Keyfile](https://crates.io/crates/ssb-keyfile) provides basic utilities to create and read keyfiles that are compatible with the JS SSB implementation. This module reexports the [`Keypair` struct](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/struct.Keypair.html) and related methods from the [ssb-crypto](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/index.html) module. As with most of the Sunrise Choir Rust SSB modules, SSB-Keyfile can either be imported into another project as a library or compiled into a standalone binary which takes parameters as arguments.

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

Alternatively, we can load or create a key that's saved in a file. These methods allow compatibility with keyfiles conforming to the JavaScript implementation. We'll call this identity **Benedict**.

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

The object signing illustrated above is made possible by the [ssb-crypto](https://crates.io/crates/ssb-crypto) crate:

> This crate provides the cryptographic functionality needed to implement the Secure Scuttlebutt networking protocols and content signing and encryption.

> There are two implementations of the crypto operations available; one that uses [libsodium](https://libsodium.gitbook.io/) C library (via the [sodiumoxide](https://crates.io/crates/sodiumoxide) crate), and a pure-rust implementation that uses [dalek](https://dalek.rs/) and [RustCrypto](https://github.com/RustCrypto/) crates (which is the default). You can select which implementation to use via Cargo.toml feature flags (see documentation - link above).

### Verification

The [ssb-verify-signatures](https://crates.io/crates/ssb-verify-signatures) crate allows us to verify the signature of an entire SSB message. It also allows the verification of multiple messages in parallel.

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

The `ssb-verify-signatures` crate also provides a method to verify the signature of the message `value` fields, without including the `key` and `timestamp` fields.

```rust
let valid_message_value = r##"{
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
}"##;

match ssb_verify_signatures::verify_message_value(valid_message_value.as_bytes()) {
    Ok(_) => println!("verified"),
    Err(e) => eprintln!("{}", e)
};
// verified
```

These verification methods are great when dealing with single messages but are not ideal for batch processing a collection of messages. Luckily, we also have access to two methods which allow parallel verification.

```rust
// convert message to bytes for easier batch processing
let valid_msg = valid_message.as_bytes();
let messages = [valid_msg, valid_msg, valid_msg];

// chunk_size is passed as the second argument (defaults to `50` for `None`)
match ssb_verify_signatures::par_verify_messages(&messages, None) {
    Ok(_) => println!("verified"),
    Err(e) => eprintln!("{}", e)
};
// verified

// convert message value to bytes
let valid_msg_value = valid_message_value.as_bytes();
let values = [valid_msg_value, valid_msg_value, valid_msg_value];

match ssb_verify_signatures::par_verify_message_values(&values, None) {
    Ok(_) => println!("verified"),
    Err(e) => eprintln!("{}", e)
};
// verified
```

The batch processing methods will return an error (`Err(InvalidSignature)`) if the verification of any message signature fails. Unfortunately, the error does not identify which message failed to verify. If you need to work out which message failed, you might have to find it using the `ssb_verify_signatures::verify_message_value` method after the parallel method has returned an error.

### Validation

A valid signature isn't the only constraint that SSB messages have to meet. The [ssb-validate](https://crates.io/crates/ssb-validate) crate helps us to verify that messages conform to the [message format](https://ssbc.github.io/scuttlebutt-protocol-guide/#message-format).

```rust
use ssb_validate;
```

The validity of a message depends on some previous state -- for example, each message needs to include a link to the previous message. The first message in a feed will have a `previous` value of `null` and `sequence` of `1`. Here we have an example of message validation using two `about` type messages: the first message declares a `name` for the given `id` (that of the `author`), while the second message defines the `image` for the given `id` by linking to a blob.

```rust
let valid_message_1 = r##"{
    "key": "%/v5mCnV/kmnVtnF3zXtD4tbzoEQo4kRq/0d/bgxP1WI=.sha256",
    "value": {
        "previous": null,
        "author": "@U5GvOKP/YUza9k53DSXxT0mk3PIrnyAmessvNfZl5E0=.ed25519",
        "sequence": 1,
        "timestamp": 1470186877575,
        "hash": "sha256",
        "content": {
            "type": "about",
            "about": "@U5GvOKP/YUza9k53DSXxT0mk3PIrnyAmessvNfZl5E0=.ed25519",
            "name": "Piet"
        },
        "signature": "QJKWui3oyK6r5dH13xHkEVFhfMZDTXfK2tW21nyfheFClSf69yYK77Itj1BGcOimZ16pj9u3tMArLUCGSscqCQ==.sig.ed25519"
    },
    "timestamp": 1571140551481
}"##;

let valid_message_2 = r##"{
    "key": "%kLWDux4wCG+OdQWAHnpBGzGlCehqMLfgLbzlKCvgesU=.sha256",
    "value": {
        "previous": "%/v5mCnV/kmnVtnF3zXtD4tbzoEQo4kRq/0d/bgxP1WI=.sha256",
        "author": "@U5GvOKP/YUza9k53DSXxT0mk3PIrnyAmessvNfZl5E0=.ed25519",
        "sequence": 2,
        "timestamp": 1470187292812,
        "hash": "sha256",
        "content": {
            "type": "about",
            "about": "@U5GvOKP/YUza9k53DSXxT0mk3PIrnyAmessvNfZl5E0=.ed25519",
            "image": {
                "link": "&MxwsfZoq7X6oqnEX/TWIlAqd6S+jsUA6T1hqZYdl7RM=.sha256",
                "size": 642763,
                "type": "image/png",
                "width": 512,
                "height": 512
            }
        },
        "signature": "j3C7Us3JDnSUseF4ycRB0dTMs0xC6NAriAFtJWvx2uyz0K4zSj6XL8YA4BVqv+AHgo08+HxXGrpJlZ3ADwNnDw==.sig.ed25519"
    },
    "timestamp": 1571140551485
}"##;

match ssb_validate::validate_message_hash_chain(valid_message_2.as_bytes(), Some(valid_message_1.as_bytes())) {
    Ok(_) => println!("validated"),
    Err(e) => eprintln!("{}", e)
};
// validated
```

Let's try to validate a hash chain using a message we know isn't valid.

```rust
let invalid_message = r##"{ "field": "value" }"##;

match ssb_validate::validate_message_hash_chain(invalid_message.as_bytes(), Some(valid_message_2.as_bytes())) {
    Ok(_) => println!("validated"),
    Err(e) => eprintln!("{}", e)
};
// Message was invalid. Decoding failed with: Message("missing field `key`")
```

Again, as with the `verify_message` methods, `validate_message_hash_chain` returns a very helpful error message to explain why the given message could not be verified. The `ssb_validate::Error` type is an `enum` with 11 variants. You can read the full list of variants in the [Rust docs for the custom Error type](https://sunrise-choir.github.io/ssb-validate/ssb_validate/enum.Error.html).

Just like the `ssb-verify-signatures` crate, `ssb-validate` provides additional methods to validate a message value hash chain and to validate both message and message value hash chains in parallel.

```rust
match ssb_validate::validate_message_value_hash_chain(valid_msg_value_2.as_bytes(), Some(valid_msg_value_1.as_bytes())) {
    Ok(_) => println!("validated"),
    Err(e) => eprintln!("{}", e)
};
// validated
```

The batch processing methods validate a collection of messages or message `value` fields. The messages must all have been created by the same author and must be ordered by ascending sequence number, with no missing messages.

These methods will mainly be useful during replication. For example, we can collect all the latest messages from a feed we're replicating and batch validate all the messages at once.

```rust
let messages = [valid_message_1.as_bytes(), valid_message_2.as_bytes()];

match ssb_validate::par_validate_message_hash_chain_of_feed::<_, &[u8]>(&messages, None) {
    Ok(_) => println!("validated"),
    Err(e) => eprintln!("{}", e)
};
// validated
```

## SSB Multiformats

You may have noticed that entities in SSB are referenced with a string starting with a sigil:

- **`%`** -- Message
- **`&`** -- Blob
- **`@`** -- Feed

This is followed by a base64-encoded integer and a suffix that describes what kind of data this is. We _could_ write gourmet parsers for this every time we need to parse an SSB reference, but it's so common that we have a crate which offers a solution for this exact problem: [ssb-multiformats](https://crates.io/crates/ssb-multiformats).

```rust
use ssb_multiformats::multifeed::Multifeed;

let maybe_feed = "@nUtgCIpqOsv6k5mnWKA4JeJVkJTd9Oz2gmv6rojQeXU=.ed25519".as_bytes();

if Multifeed::from_legacy(&maybe_feed).is_ok() {
    println!("is feed")
};
```

We also have access to an equivalent method for handling message and blob references.

```rust
use ssb_multiformats::multihash::Multihash;

let ssb_ref = "%x60lINqpVU9Fw5cdNq/7raSy/N2zUs8NT9TLsXu5qSQ=.sha256".as_bytes();

let is_msg_or_blob = Multihash::from_legacy(&ssb_ref).is_ok();
println!("{}", is_msg_or_blob);
// true
```

Note that `Multihash::from_legacy()` returns `Ok(Multihash, &[u8])` on success, where `Multihash` is an `enum` with two variants: `Message([u8; 32])` and `Blob([u8; 32])`.

## Peer Connections

We've got our keypair, we know how to make messages, our feed seems to be valid -- but none of that is very useful unless we can send those messages to peers. Before we think about connecting to peers, we first have to be able to define the Scuttleverse we're acting in by creating or referencing a network key.

### Network Key

The 'network key' - also known as the 'network identifier', 'app key' or 'SHS key' - is used during the [secret handshake](https://ssbc.github.io/scuttlebutt-protocol-guide/#handshake) to prove that both parties are participating in the same SSB network. It is a 32-byte key which allows distinct, isolated Scuttlebutt networks to be created. If you're building a network of temperature sensors on Secure Scuttlebutt you probably don't want to be peering with people sharing source code or building a social network (👋). In that case, you would supply a unique network key when performing a handshake.

The [ssb-crypto](https://crates.io/crates/ssb-crypto) crate provides us with a [`NetworkKey struct`](https://docs.rs/ssb-crypto/0.2.2/ssb_crypto/struct.NetworkKey.html) with convenient methods for working with network keys.

```rust
use ssb_crypto::NetworkKey;
```

We'll use the most common app key, implemented as a `const` for `NetworkKey`, which is used for the offline-first social network that you might be familiar with.

```rust
let netkey = NetworkKey::SSB_MAIN_NET;
```

If you wish to create a distinct, isolated Scuttlebutt network, generate a random network key instead of using the `SSB_MAIN_NET` key.

```rust
let alt_netkey = NetworkKey::generate();

// ensure the generated key is distinct from the main network key
assert_ne!(alt_netkey, NetworkKey::SSB_MAIN_NET);
```

Alternatively, generate a random network key using a given cryptographically-secure random number generator.

```rust
let mut rng = rand::thread_rng();

let rng_alt_netkey = NetworkKey::generate_with_rng(&mut rng);
```

### Discovery

As described in the Scuttlebutt Protocol Guide's section on [Discovery](https://ssbc.github.io/scuttlebutt-protocol-guide/#discovery), we need to know the IP address, port number and public key of a peer in order to make a connection. The Scuttlebutt protocol offers three methods for peers to discover one other: local network discovery (via UDP broadcast messages), invite codes and pub "about" messages.

The Sunrise Choir implementation does not define discovery mechanisms or peer address data models for us. These aspects of the SSB application are left up to us to develop. Fortunately, two existing implementations are available for us to study and replicate: the [Scuttle-chat](https://github.com/clevinson/scuttle-chat) application implements a local network [discovery](https://github.com/clevinson/scuttle-chat/blob/master/src/discovery.rs) mechanism using UDP, as does the [solar](https://github.com/Kuska-ssb/solar) application in [lan_discovery](https://github.com/Kuska-ssb/solar/blob/master/src/actors/lan_discovery.rs).

### Secret Handshake

Once we have discovered the necessary details of our peer (IP address, port, public key), we can create a TCP stream between us and perform the handshake: a 4-step process to authenticate both parties and establish an encrypted channel. In the Sunrise Choir SSB implementation, the [ssb-handshake](https://crates.io/crates/ssb-handshake) crate provides us with the necessary functionality. The crate provides two primary functions: one for the [`server_side()`](https://docs.rs/ssb-handshake/0.5.1/ssb_handshake/fn.server_side.html) of the handshake and one for the [`client_side()`](https://docs.rs/ssb-handshake/0.5.1/ssb_handshake/fn.client_side.html).

```rust
// initiate the server side of the handshake
let handshake_keys = ssb_handshake::server_side(&mut stream, &net_key, &server_keypair).await?;
```

Note that we need to know the public key of the server we are attempting to authenticate with. 

```rust
// initiate the client side of the handshake
let handshake_keys = ssb_handshake::client_side(&mut stream, &net_key, &client_keypair, &server_pk).await?;
```

Working code examples for these methods (using [`async_std`](https://docs.rs/async-std/1.9.0/async_std/)) can be found in the Sunrise SSB Playground: [`handshake_server.rs`](https://github.com/mycognosist/sunrise-ssb-playground/blob/main/examples/handshake_server.rs) and [`handshake_client`](https://github.com/mycognosist/sunrise-ssb-playground/blob/main/examples/handshake_client.rs).

The result of our successful handshake is the [`HandshakeKeys`](https://github.com/sunrise-choir/ssb-handshake/blob/0ee6c9b099a56b9493d835740d946177748a9288/src/crypto/outcome.rs) `struct` (assigned to the `handshake_keys` variable in our example code above). This `struct` contains the public key of our remote peer, as well as the keys and nonces required to encrypt further communications via box stream.

### Box Stream

Box stream is the bulk encryption protocol used to exchange messages following the handshake until the connection ends. It is designed to protect messages from being read or modified by a man-in-the-middle.

The [ssb-boxstream crate](https://crates.io/crates/ssb-boxstream) exposes a [`BoxStream struct`](https://docs.rs/ssb-boxstream/0.2.2/ssb_boxstream/struct.BoxStream.html) comprised of two fields: a reader and a writer. These are used to send and receive data from our peer. In order to create a new boxstream, we first need our TCP stream (which we previously declared as `stream`) and the set of keys returned from our successful secret handshake attempt (declared as `handshake_keys`). Let's see how this works in practice.

```rust
use ssb_boxstream::BoxStream;

// split the tcp stream into a reader and writer
// note: async_std tcp stream does not implement `.split()`.
// we are cloning the stream here to allow separate access.
let tcp_reader = stream.clone();
let tcp_writer = stream;

// create a new boxstream and split it into a reader and writer
let (mut box_reader, mut box_writer) = BoxStream::new(
    tcp_reader,
    tcp_writer,
    handshake_keys.read_key,
    handshake_keys.read_starting_nonce,
    handshake_keys.write_key,
    handshake_keys.write_starting_nonce,
)
.split();

// test data to be written to the boxstream
let body = [1, 1, 0, 2, 2, 0, 2, 1];

// write some data to the box
box_writer.write_all(&body[0..8]).await?;

// flush the writer buffer (send the data)
box_writer.flush().await?;
```

The above example illustrates writing data to the boxstream, but how do we receive the data on the other side? We create a buffer (after a successful handshake and the creation of our boxstream) and read into it from the boxstream reader.

```rust
// create a buffer to store incoming data
let mut buf = [0; 8];

// read some data from the box
box_reader.read(&mut buf).await?;
```

### Packet Stream (RPC Protocol)

While it's possible to read and write raw data over the box stream, SSB implements an RPC (remote procedure call) protocol to define the format of messages passed between peers. The [ssb-packetstream](https://crates.io/crates/ssb-packetstream) crate provides the data types and functionality we need to send data packets over the stream. Let's continue building on our box stream example to read and write packets.

```rust
use ssb_packetstream::*;
use use futures::sink::SinkExt;

// create a new packet sink (writer)
let mut packet_sink = PacketSink::new(box_writer);

// create a packet to send (write to sink)
let p = Packet::new(
    IsStream::Yes,
    IsEnd::No,
    BodyType::Binary,       // binary, utf8 or json
    12345,                  // id (i32)
    vec![1, 2, 3, 4, 5],    // body (Vec<u8>)
);

// send the packet
packet_sink.send(p).await.unwrap();

// close the sink (sends a 'goodbye message')
packet_sink.close().await.unwrap();
```

Now let's create a packet stream to receive the packet on the client-side of our connection.

```rust
// create a new packet stream (reader)
let mut packet_stream = PacketStream::new(box_reader);

// attempt to receive a packet (read from stream)
let packet = packet_stream.next().await.unwrap().unwrap();

println!("Packet received: {:?} from {:?}", &packet.body, packet.id);
// Packet received: [1, 2, 3, 4, 5] from 12345

// check if data packet is part of a stream
println!("IsStream: {}", packet.is_stream());
// IsStream: true

// check if data packet represents the end of a stream
println!("IsEnd: {}", packet.is_end());
// IsEnd: false
```

Once the communication is complete, we send a 'goodbye message' to close the stream with our peer.

```rust
// close the sink (send a 'goodbye message')
packet_sink.close().await.unwrap();
```

```rust
// attempt to receive the next packet ('goodbye message' in this case)
match packet_stream.next().await {
    Some(p) => println!("{:?}", p),             // Ok(Packet { /* fields */ })
    None => println!("received rpc goodbye")    // [0u8; 9]
};
// received rpc goodbye
```

That covers the basics of working with packet streams using the `ssb-packetstream` crate. If you're looking for a more comprehensive reference for the RPC protocol, remember to check out the exquisitely-craft [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/index.html#rpc-protocol).

## TODO

Replication.

## Contact

The Sunrise Choir consists of [@Mikey](https://github.com/ahdinosaur), [@Piet](https://github.com/pietgeursen), [@Matt](https://github.com/mmckegg) and [@Sean](https://github.com/sbillig). This documentation was compiled by [@glyph](https://github.com/mycognosist). All can be found in the Scuttleverse:

* mikey: `@6ilZq3kN0F+dXFHAPjAwMm87JEb/VdB+LC9eIMW3sa0=.ed25519`
* piet: `@U5GvOKP/YUza9k53DSXxT0mk3PIrnyAmessvNfZl5E0=.ed25519`
* matt: `@FbGoHeEcePDG3Evemrc+hm+S77cXKf8BRQgkYinJggg=.ed25519`
* sean: `@N/vWpVVdD1e8IbACUQE4EVGL6+aodQfbQZ8ByC+k79s=.ed25519`
* glyph: `@HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519`
