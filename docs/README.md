# SSB-JS Handbook

## Introduction

There are a variety of teams and technologies that implement **Secure Scuttlebutt (SSB)**, but the canonical 'reference implementation' is SSB-JS. This implementation is built on interesting and experimental technologies that you need to know about if you're using SSB-JS.

You should be familiar with JavaScript, Node.js, and at least the basics mentioned in the [Scuttlebutt Protocol Guide](https://ssbc.github.io/scuttlebutt-protocol-guide/#keys-and-identities). Before we get started, these are a few terms that you should be familiar with:

- **Keys** are Ed25519 keypairs used for signing and encryption (like a PGP key).
- **Messages** are signed payloads that link to the previous message (like a Git commit).
- **Feeds** are an ordered collection of messages (like a Git branch).
- **Secret Handshake (SHS)** is an authenticated key exchange protocol (like TLS).
- **MuxRPC** is a communication protocol (like HTTP).
- **Multiserver address** is [usually] a combined address and public key (like HTTPS with certificate pinning).

## SSB-Keys

The [SSB-Keys](https://github.com/ssb-js/ssb-keys/) is used to manage keys and provide the basic utilities for encryption, signatures, and verification. This module forms the backbone of Secure Scuttlebutt, and is absolutely critical for understanding how the rest of the stack fits together.

```javascript
const ssbKeys = require("ssb-keys");
```

The most basic operationg is generating keys: this produces an identity encoded as an object. We'll call this identity **Alice**.

```javascript
const alice = ssbKeys.generate();
// {
//   curve: 'ed25519',
//   public: 'DC0jeNHsHdftr56o22W5DDOGk0y89pR5eEBu2tBUvDI=.ed25519',
//   private: 'dNZ6wqID3ySKK62O36r+V7gUaQp0wOoJy7vOLJ6RTYsMLSN40ewd1+2vnqjbZbkMM4aTTLz2lHl4QG7a0FS8Mg==.ed25519',
//   id: '@DC0jeNHsHdftr56o22W5DDOGk0y89pR5eEBu2tBUvDI=.ed25519'
// }
```

Alternatively, we can load or create a key that's saved in a file. This method will return the same result as long as the file exists. We'll call this identity **Bob**.

```javascript
const bob = ssbKeys.loadOrCreateSync("./bob");
// {
//   curve: 'ed25519',
//   public: 'MUO8i/oc8ksxyj9ImJdoU9iv8JBlyq4VdvXhjTVMKuc=.ed25519',
//   private: 'FpU+ob+96tSMHQfRp7nGIvUb9Fe7V9NosvxLKWDeHXcxQ7yL+hzySzHKP0iYl2hT2K/wkGXKrhV29eGNNUwq5w==.ed25519',
//   id: '@MUO8i/oc8ksxyj9ImJdoU9iv8JBlyq4VdvXhjTVMKuc=.ed25519'
// }
```

### Signing

The most common operation with our keys is signing objects and then verifying their signatures. SSB has a very specific message encoding format, but for this example we'll use an example object.

```javascript
const exampleObject = {
  type: "example",
};
```

We'll use Alice's private key to sign this object, which produces an object with an inline signature.

```javascript
const signedObject = ssbKeys.signObj(alice.private, null, exampleObject);
// {
//   type: 'example',
//   signature: 'i+3ro67reK8BeTv38nXrUlnbwK2agswCviRK8mudl0pOv/K+ji9m37pszb8aOSys2EvShoex/916n9KExrdvBw==.sig.ed25519'
// }
```

### Verification

Now when we compare the signature against Alice's public key, we'll see that the signature is valid and that Alice must have signed this message. This is important for ensuring that an object hasn't been tampered with. Any change to the object will invalidate the signature.

```javascript
const isValid = ssbKeys.verifyObj(alice.public, null, signedObject);
// true
```

If we try to do the same verification with Bob's public key, we'll see that the signature is **not valid**. Either the public key is incorrect or the object has been tampered with, but the signature is invalid.

```javascript
ssbKeys.verifyObj(bob.public, null, signedObject);
// false
```

## Chloride

The SSB-Keys project is built on top of [Chloride](https://github.com/ssb-js/chloride), a wrapper around libsodium or TweetNaCl. The best way to understand this API is to [read the code](https://github.com/dominictarr/sodium-chloride/blob/master/index.js), which exports 20 cryptography methods that form the basis of SSB-Keys and other modules. This module is reasonably small but absolutely critical.

## SSB-Validate

A valid signature isn't the only constraint that SSB messages have to meet. The [SSB-Validate](https://github.com/ssb-js/ssb-validate) module helps us create and verify messages to ensure that they conform to the [message format](https://ssbc.github.io/scuttlebutt-protocol-guide/#message-format).

```javascript
const ssbValidate = require("ssb-validate");
```

The validity of a message depends on some previous state -- for example, each message needs to include a link to the previous message. This state is encapsulated in a variable called `state`, which starts out being empty.

```javascript
const state = ssbValidate.initial();
// { validated: 0, queued: 0, queue: [], feeds: {}, error: null }
```

Just to make sure it's working alright, we'll try passing a message that we know isn't valid.

```javascript
try {
  ssbValidate.append(state, null, { oops: true });
} catch (err) {
  // Error: invalid message: must have author
  //   at Object.exports.checkInvalidCheap (ssb-validate/index.js:133:12)
  //   at Object.exports.checkInvalid (ssb-validate/index.js:164:21)
  //   at Object.exports.appendKVT (ssb-validate/index.js:226:20)
  //   at Object.exports.append (ssb-validate/index.js:255:18)
}
```

We could hand-craft a gourmet message for the validator, but luckily we don't have to.

```javascript
const message = ssbValidate.create(
  state.feeds[alice.public],
  alice,
  null,
  exampleObject,
  Date.now()
);
// {
//   previous: null,
//   sequence: 1,
//   author: '@ZByBkJp2fBgfw0ql3wbQFOWa9Fphzv0T0pq6FEvizuc=.ed25519',
//   timestamp: 1597344109162,
//   hash: 'sha256',
//   content: { type: 'example' },
//   signature: 'TMxuh2fAkPs9/6MNcyasYx+7m/kzD992zxLaBb4cnDDGnlYyplMhgpeA6uTtn7oQLJd3bY0XBw8mUu/9vnd3Bg==.sig.ed25519'
// }
```

Now we can confidently append this message to SSB-Validate, which mutates and returns the state object.

```javascript
ssbValidate.append(state, null, message);
// {
//   validated: 1,
//   queued: 0,
//   queue: [
//     {
//       key: '%QuIfUIRC0Iwjepy86Rzd6w3Yhh4ui9HF6E8TxPiR3EE=.sha256',
//       value: [Object],
//       timestamp: 1597344109166
//     }
//   ],
//   feeds: {
//     '@ZByBkJp2fBgfw0ql3wbQFOWa9Fphzv0T0pq6FEvizuc=.ed25519': {
//       id: '%QuIfUIRC0Iwjepy86Rzd6w3Yhh4ui9HF6E8TxPiR3EE=.sha256',
//       sequence: 1,
//       timestamp: 1597344109162,
//       queue: []
//     }
//   },
//   error: null
// }
```

When we repeat this process the sequence number increases and each message links to the previous message, which forms a feed of messages.

## SSB-Ref

You may have noticed that entities in SSB are referenced with a string starting with a sigil:

- **`%`** -- Message
- **`&`** -- Blob
- **`@`** -- Feed

This is followed by a base64-encoded integer and a suffix that describes what kind of data this is. We _could_ write gourmet parsers for this every time we need parse an SSB reference, but it's so common that we have a module dedicated to solving this exact problem: [SSB-Ref](https://github.com/ssb-js/ssb-ref).

```javascript
const ssbRef = require("ssb-ref");

const maybeFeed = ssbRef.isFeed(
  "@nUtgCIpqOsv6k5mnWKA4JeJVkJTd9Oz2gmv6rojQeXU=.ed25519"
);
// true
```

## Secret-Stack

We've got our keys, we know how to make messages, our feed seems to be valid -- but none of that is very useful unless you can send those messages to peers. The [Secret-Stack](https://github.com/ssb-js/secret-stack) module provides encrypted communication with peers, you just need to know their [Multiserver](https://github.com/ssb-js/multiserver) address.

```javascript
const secretStack = require("secret-stack");
```

Secret-Stack is painfully unopinionated, and is usually wrapped with lots of sugary helpers. Don't worry, this explicit configuration is mostly for educational purposes and you will rarely have to hink about this.

### Key

The 'app key', also called an 'SHS key', or 'network identifier' in the [handshake protocol documentation](https://ssbc.github.io/scuttlebutt-protocol-guide/#handshake), is a 32-byte key used to keep Secret-Stack networks isolated from each other. If you're building a network of temperature sensors on Secure Scuttlebutt you probably don't want to be peering with people sharing source code or building a social network (👋), so when you configure initialize Secret-Stack you have to supply an app key.

We'll use the most common app key, which is used for the offline-first social network that might be familiar with:

```javascript
const stack = secretStack({
  appKey: "1KHLiKZvAvjbY1ziZEHMXawbCEIM6qwjCDm3VYRan/s=",
});
```

### Configuration

We also need to give our peer a port and keys. We can create an app which exposes methods that can be run locally or over the network.

```javascript
const exampleApp = stack({ port: 3838, keys: alice });
```

These methods can be enumerated by calling the 'manifest' method, which returns the [MuxRPC manifest](https://github.com/ssb-js/muxrpc#manifest).

```javascript
exampleApp.manifest();
// {
//   auth: 'async',
//   address: 'sync',
//   manifest: 'sync',
//   multiserver: { parse: 'sync', address: 'sync' },
//   multiserverNet: {}
// }

exampleApp.address();
// "net:localhost:3939~shs:kjFBD0605Q459/fQgO41VBInkItp51gx+dy4129trUc="
```

Once we finish using an app, we can end all connections and stop listening on network interfaces.

```javascript
exampleApp.close();
```

### Plugins

Unfortunately this isn't very useful by itself, so we need to add some useful methods to our Secret-Stack app. This is accomplished by the [plugin system](https://github.com/ssb-js/secret-stack/blob/master/PLUGINS.md), which adds plugins to the stack that ends up being initialized by each app.

We'll make our own example little plugin that reads an option from the config.

```javascript
const examplePlugin = {
  name: "example",
  version: "1.0.0",
  manifest: {
    hello: "sync",
  },
  init: (api, options) => {
    return {
      hello: (input) => `Hello ${input}! My name is ${options.name}.`,
    };
  },
};
```

Adding the plugin to our stack is easy too.

```javascript
stack.use(examplePlugin);
```

When we create a new app from this stack, it will have our plugin available and documented in the manifest.

```javascript
const pluginExampleApp = stack({ port: 4040, keys: alice, name: "Alice" });

pluginExampleApp.manifest();
// {
//   auth: 'async',
//   address: 'sync',
//   manifest: 'sync',
//   multiserver: { parse: 'sync', address: 'sync' },
//   multiserverNet: {},
//   example: { hello: 'sync' }
// }
```

We can run it too, of course.

```javascript
console.log(pluginExampleApp.example.hello("world"));
// "Hello world! My name is Alice."

pluginExampleApp.close();
```

### Connection

The real magic starts to happen when you connect peers. Secret-Stack lets us take our peers and connect them over an encrypted TCP connection.

```javascript
const aliceApp = stack({ port: 4141, keys: alice });
const bobApp = stack({ port: 4242, keys: bob, name: "Bob" });

aliceApp.connect(bobApp.address(), async (err, bobApi) => {
  if (err) throw err;
  console.log(await bobApi.example.hello("world"));
  // Hello world! My name is Bob.

  aliceApp.close();
  bobApp.close();
});
```

## Multiserver

It's important to know how this works -- we use a module called [Multiserver](https://github.com/ssb-js/multiserver) as a pluggable network protocol framework. Multiserver has its own address specification, similar to a URI, and creates a duplex stream between peers. Multiserver extends basic protocols like TCP or WebSocket with _transforms_, which give us the ability to encode 'SHS over TCP' and implement it in a way that the layers don't need to know about each other. This system of nested tunnels and explicit public keys is used to advertise how peers can connect to us.

## MuxRPC

Unfortunately these streams aren't very useful by themselves -- we use MuxRPC, an pluggable remote procedure call framework, which gives us the ability to send and receive procedures through Multiserver streams. There are at least four types of procedures:

- **async** -- Asynchronous, wait for a response.
- **sync** -- Synchronous, response is immediate when called via process and an alias for 'async' when called through the network.
- **source** -- Download stream, returns a source with data coming through it.
- **sink** -- Upload stream, returns a sink that you can put data in.
- **duplex** -- Two-way stream, returns a duplex stream.

As mentioned earlier, all peers implement a 'manifest' procedure (sync), which returns a list of procedures and their types.

[1]: https://ssbc.github.io/scuttlebutt-protocol-guide/
[2]: https://github.com/ssb-js/secret-stack/
[3]: https://github.com/auditdrivencrypto/secret-handshake
[4]: https://github.com/ssb-js/muxrpc
[5]: https://github.com/ssb-js/ssb-config#configuration
[6]: https://github.com/ssb-js/muxrpc#manifest
