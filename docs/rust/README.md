# Rust SSB 

## Introduction

There are currently two major Rust SSB implementations: [Kuska-ssb](rust/kuska) and [Sunrise Choir](rust/sunrise-choir); as well as a client library: [golgi](rust/golgi).

In order to work with either of these sets of libraries, you should be familiar with Rust and at least the basics mentioned in the [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/#keys-and-identities). Before we get started, these are a few terms that you should be familiar with:

- **Keys** are Ed25519 keypairs used for signing and encryption (like a PGP key).
- **Messages** are signed payloads that link to the previous message (like a Git commit).
- **Feeds** are an ordered collection of messages (like a Git branch).
- **Secret Handshake (SHS)** is an authenticated key exchange protocol (like TLS).
- **MuxRPC** is a communication protocol (like HTTP).
- **Multiserver address** is [usually] a combined address and public key (like HTTPS with certificate pinning).

[](kuska.md ':include')

[](sunrise-choir.md ':include')

[](golgi.md ':include')
