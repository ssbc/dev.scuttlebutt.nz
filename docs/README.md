# Scuttlebutt Treasure Map for Developers

![](./assets/bandana.jpg)

Welcome traveller! **Secure Scuttlebutt (SSB)** is a project which spans several communities and technologies.
This documentation is a meta-guide designed to familiarise you with some core concepts,
and help you find the current tools in you language of choice.

More information:
* If you are looking for an **implementation-agnostic** protocol specification, see the [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/) site
* If you want to learn more about Scuttlebutt in general (such as talks and how to get started), go visit the [main site](https://ssb.nz).

## Implementations

Jump straight in?

Language               | Name | Status | In production
-----------------------|:---:| :---: | :---:
[Node.js](javascript/) | N/A, see [SSBC](https://github.com/orgs/ssbc/repositories?language=javascript&type=all) | Stable | ✓ 
[Go](golang/)          | [`go-ssb`](https://github.com/ssbc/go-ssb) | Alpha | ✓
[Go](golang/)          | [`scuttlego`](https://github.com/planetary-social/scuttlego) | Alpha (?) | ?
[Rust](rust/)          | N/A, see [Kuska-SSB](https://github.com/Kuska-ssb) | Pre-alpha (?) | ?
[Python](python/)      | [`pyssb`](https://github.com/pferreir/pyssb) | Pre-alpha (?) | No

## Learn some core ideas

The reference implementation of scuttlebutt was built in Node.js, and to date this languages ecosystem is the mostly developed,
but it also uses the most experimental / homegrown technologies.

All implementations reference some common concepts:

### Feeds

Data in scuttlebutt is held in feeds. Each peer in the network holds it's own feed of messages it has written, along with the feeds of peers that it cares about.


```mermaid
graph RL
  subgraph "@Zelf's Feed"
    A([msg 1])
    B([msg 2])
    C([msg 3])
    D([msg 4])

    D-->C-->B-->A
  end

classDef default fill: #42b983, color: white, stroke: none;
classDef edgePath stroke: ##42b983, stroke-width: 2;
classDef cluster fill: #fff, color: #240e5e, stroke: #240e5e, stroke-width: 1;
```
_Diagram showing a chain of messages all belonging to a particular feed. Each references the message before it in the feed by its unique ID (the hash of the message)._

- A **Feed** is an linked list of messages which are all signed by a particular set of keys.
- **Messages** are signed payloads that link to the _previous_ message (like a Git commit).
- **Keys** are Ed25519 keypairs used for signing and encryption (like a PGP key).

This construction makes it easy to verify messages (we check the signature on the message matches the author),
as well as to ask other peers for the latest updates (e.g. "can I have any messages after message 4 for @Zelf").

### Replication

Replication is about sharing updates (messages added to feeds) with other peers.
```mermaid
graph RL
  subgraph "@Zelf's records"
      ZZ4((4))-->ZZ3((3))-->ZZ2((2))-->ZZ1((1))

      ZM2((2))-->ZM1((1))
  end

  subgraph "@Mix's records"
      MM3((3))-->MM2((2))-->MM1((1))
  end

  MM3-.->3((3))-.->ZM2


classDef default fill: #42b983, color: white, stroke: none;
classDef edgePath stroke: ##42b983, stroke-width: 2;
classDef cluster fill: #fff, color: #240e5e, stroke: #240e5e, stroke-width: 1;

classDef mix fill: #d03ae6, color: white, stroke: none;
class ZM1,ZM2,MM1,MM2,MM3,3 mix 
```
_Diagram showing @Zelf who holds her own feed (green) and 2 messages of @mix's feed (fuschia), and her replicating the latest message (3) from Mix_

In Scuttlebutt this is done through
- **Multiserver address** [usually] a combined address and public key (like HTTPS with certificate pinning).
- **Secret Handshake (SHS)** - an authenticated key exchange protocol (like TLS).
- **MuxRPC** - a communication protocol (like HTTP).

### Protocol Guide

You can read about this in depth in the [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/#keys-and-identities).

If you'd like an erotic representation of the Secret Handshake Protocol, you can see that <a href="assets/handshake-erotica.png">here</a> O.o

## Web presence

> **WARNING**: We understand that it might be a bit confusing to navigate the
> SSB web presence at the moment. It is indeed spread over several websites and
> quite often not up-to-date. We're working on it. Feel free to [dive in and
> help us out](/#contributing)! Also, if there is a link missing below, please
> free to add it.

- [dev.scuttlebutt.nz](https://dev.scuttlebutt.nz) ([sources](https://github.com/ssbc/dev.scuttlebutt.nz))
- [handbook.scuttlebutt.nz](https://handbook.scuttlebutt.nz) ([sources](https://github.com/ssbc/handbook.scuttlebutt.nz))
- [scuttlebot.io](https://scuttlebot.io) ([sources](https://git.scuttlebot.io/%25hg8wG6xCDKVWoPYCS84HY7Adrd6JEUYoM23%2BGwn24I4%3D.sha256) (`git-ssb`))
- [scuttlebutt.eu](https://scuttlebutt.eu) ([sources](https://github.com/scuttlebutt-eu/scuttlebutt-eu.github.io))
- [scuttlebutt.nz](https://scuttlebutt.nz) ([sources](https://gitlab.com/ssbc/scuttlebutt.nz))
- [ssbc.github.io](https://ssbc.github.io) ([sources](https://github.com/ssbc/ssbc.github.io))

## Contributing

> If you're already on SSB and want to help onboard new development butts,
> please add yourself to the list below.

The simplest way is to [join SSB](https://scuttlebutt.nz/get-started) and message the following folks:

* decentral1se `@i8OXtTYaK0PrF002pd4vpXmrlg98As7ZMaHGKoXixdM=.ed25519`
