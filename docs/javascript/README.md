# Javascript stack

## Introduction

You should be familiar with JavaScript, Node.js, and at least the basics mentioned in the [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/#keys-and-identities). Before we get started, these are a few terms that you should be familiar with:

- **Keys** are Ed25519 keypairs used for signing and encryption (like a PGP key).
- **Messages** are signed payloads that link to the previous message (like a Git commit).
- **Feeds** are an ordered collection of messages (like a Git branch).
- **Secret Handshake (SHS)** is an authenticated key exchange protocol (like TLS).
- **MuxRPC** is a communication protocol (like HTTP).
- **Multiserver address** is [usually] a combined address and public key (like HTTPS with certificate pinning).
- **DB** is where messages are stored locally and from where they can be queried.


## Core Modules

These are modules which are always used because they are either fundamental to the stack,
or are tools for canonically testing e.g. a message Id.

[](core_modules.md ':include')


## Plugins

Short of "secret-stack plugins" these are modules which extend the functionality of `secret-stack`

[](plugins.md ':include')


## Example Setups


### Butt-cast

This is an application designed pub publishing and annotating podcasts
Source: https://github.com/ZELFs/ButtCast

NOTE: This app is in development and currently doesn't peer replication

[filename](example/butt-cast.md ':include')


