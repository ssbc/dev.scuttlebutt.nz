# dev.scuttlebutt.nz

## Introduction

There are a variety of teams and technologies that implement **Secure Scuttlebutt (SSB)**, but the canonical 'reference implementation' is SSB-JS. This implementation is built on interesting and experimental technologies that you need to know about if you're using SSB-JS.

You should be familiar with JavaScript, Node.js, and at least the basics mentioned in the [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/#keys-and-identities). Before we get started, these are a few terms that you should be familiar with:

- **Keys** are Ed25519 keypairs used for signing and encryption (like a PGP key).
- **Messages** are signed payloads that link to the previous message (like a Git commit).
- **Feeds** are an ordered collection of messages (like a Git branch).
- **Secret Handshake (SHS)** is an authenticated key exchange protocol (like TLS).
- **MuxRPC** is a communication protocol (like HTTP).
- **Multiserver address** is [usually] a combined address and public key (like HTTPS with certificate pinning).


## Implementations

### node.js

The original implementation
[node.js](javascript/)

### golang

[golan](golan/)

### rust

There are several rust implementations
[rust](rust/)
