# go-ssb

## Introduction

Go-SSB implements most of the existing JavaScript functionality but is less modular, due to the typed nature of the language. Most of the code lives in a single repository: [go-ssb](https://github.com/cryptoscope/ssb) with a couple of packages split out, like `go-muxrpc` and `secretstream`.

The import path for the code is `go.cryptoscope.co/ssb`.

If you are not familiar with the [Go programming language](https://golang.org), I suggest you get started with the [guided tour](https://tour.golang.org/welcome/1).


## Packages Overview

This should serve as guardrails when looking into the repo for the first time.

### cmd

By coding convention, this contains the executable tools. Most importantly these two:

* `go-sbot` which hosts the database and p2p server.
* `sbotcli` a muxrpc client to do common tasks, like publishing a message or querying messages.

### Message

Message contains all the code to create and verify messages.

The current established format is

* [message](https://pkg.go.dev/go.cryptoscope.co/ssb/message) - abstract verification and publish helpers
* [message/legacy](https://pkg.go.dev/go.cryptoscope.co/ssb/message/legacy) - how to encode and verify [the current ssb messages](https://spec.scuttlebutt.nz/feed/messages.html)
* message/multimsg - encoding multiple kinds of messages to disk (see _Database Concept Overview_ below)

### Plugins

Plugins supply muxrpc handlers to query the database in certain ways.
They also implement other sub-protocols like ssb-blobs.

* gossip - implements the `createHistoryStream` muxrpc call. _Legacy_ (non-EBT) Replication of fetching and verifying the selected feeds is found here.
* replicate - roughly translates to [ssb-replicate](https://github.com/ssbc/ssb-replicate) and only selects _which_ feeds to block and fetch.
* control - connect to remote peers. (a naming hack, supplies `ctrl.connect` which should actually be `gossip.connect`)
* status - return some information about the running server (number of connected peers, number of messages, index state)
* publish - exposes `sbot.PublishLog.Publish(...)` as the muxrpc async call `publish`
* get - very short module, just exposes [sbot.Get](https://pkg.go.dev/go.cryptoscope.co/ssb/sbot#Sbot.Get) over muxrpc
* rawread - a bunch of _read messages from the Receive Log_ functions likes (`createLogStream` and `messagesByType`)
* legacyinvites -  follow-back sub protocol for new users. [ssb-invite](https://github.com/ssbc/ssb-invite)
* peerinvites - the server part of the newer invite system: [ssb-peer-invites](https://github.com/ssbc/ssb-peer-invites)
* private - about to be deprecated way of accessing private messages
* friends - supplies some of [ssb-friends](https://github.com/ssbc/ssb-friends), namly `isFollowing`, `isBlocking` and `hops` but not `hopStream`, `onEdge` or `createLayer`.
* whoami - returns the public key reference of the peer you are talking to.
* blobs - the muxrpc handlers for [ssb-blobs](https://github.com/ssbc/ssb-blobs). The storage and want-management is found in `blobstore`.


[Plugins2](https://github.com/cryptoscope/ssb/tree/master/plugins2) is an attempt to unify the index generation and muxrpc handler code.

* `tangles` categorizes messages by `root:%...` (v1 tangle) and also v2 form that is used in [private-groups](https://github.com/ssbc/private-group-spec/blob/master/group/add-member/README.md).
* `bytype` looks at `type:string` and thus gives cheap access to the common message types (post, contact, about, etc.).
* `names` implements the same muxrpc interface as [ssb-names](https://github.com/ssbc/ssb-names)



### Sbot

This package ties together network, repo and plugins like graph and blobs into a large server that offers data-access APIs and background replication. It's name dates back to a time where ssb-server was still called scuttlebot, in short: sbot.

It offers a flexible [functional options API](https://pkg.go.dev/go.cryptoscope.co/ssb/sbot#Option) to tune the server as desired. This includes file locations, network and signing keypairs, local discovery behavior and plugin selection.

For instance, if you wanted a bot with local disvery enabled (by default it's off) and the replication database on a custom mountpoint, you would do this:

```go
import "go.cryptoscope.co/ssb/sbot"

func ex() {
  thebot, err := sbot.Sbot(
    sbot.WithRepoPath("/mnt/external/my-go-ssb-db"),
    sbot.EnableAdvertismentBroadcasts(true), // send your own location and ID
    sbot.EnableAdvertismentDialing(true),    // connect to others
  )
  check(err)

  ctx:=context.Background()
  for { // handshake errors can make serve return, better keep serving
    err=thebot.Serve(ctx)
    if err!=nil {
      log.Println("handshake failed:", err)
    }
  }
}
```


### Repo, indexes and multilogs

* repo - contains utility modules to open offset logs and create different kinds of indexes
* indexes - contains functions to create indexing  for `get(%ref) -> message`. Also contains a utility to open the contact trust graph using the `repo` and `graph` packages.
* multilogs - currently contains the indexing functions for the `by user` and `private readable` multilogs (See the _MultiLog_ section in _Database concept overview_ for more)

### Misc

* [blobstore](https://pkg.go.dev/go.cryptoscope.co/ssb/blobstore) - the filesystem storage and sympathy managment for [ssb-blobs](https://github.com/ssbc/ssb-blobs)
* network - a utility module for dialing and listening to secret-handshake powered muxrpc connections
* [graph](https://pkg.go.dev/go.cryptoscope.co/ssb/graph) - derives trust/block relations by consuming type:contact message and offers lookup APIs between two feeds.
* [client](https://pkg.go.dev/go.cryptoscope.co/ssb/client) - a simple muxrpc interface to common ssb methods, similar to [ssb-client](https://github.com/ssbc/ssb-client)
* [invite](https://pkg.go.dev/go.cryptoscope.co/ssb/invite) - This package contains functions for parsing invite codes and dialing a pub as a guest to redeem a token. The muxrpc handlers and storage are found in `plugins/legacyinvite`.
* [private](https://pkg.go.dev/go.cryptoscope.co/ssb/private) - utility functions to de- and encrypted messages. Maps to `box()` and `unbox()` in [ssb-keys](https://github.com/ssbc/ssb-keys).

### Tests

While most folders contain `_test.go` files where functionallity is tested as unit tests, this folder
contains sepcial code to run tests against the JavaScript implementation.

## Database concept Overview

Go-SSB's choice of data storage and retrieval is very much inspired by ssb-db / flumedb.

A big part of working with ssb means dealing with append-only logs.
The current abstracted interface is implemented in [margaret.Log](https://pkg.go.dev/go.cryptoscope.co/margaret@v0.1.6#Log).

* `Append(interface{}) (Seq, error)` adds a new entry. (In go-ssb usually done through [`sbot.PublishLog`](https://pkg.go.dev/go.cryptoscope.co/ssb/sbot#Sbot) which also signs the entry, or through replication in [`message.NewVerifySink`](https://pkg.go.dev/go.cryptoscope.co/ssb/message#NewVerifySink))
* `Get(Seq) (interface{}, error)` retrieves the entry behind that sequence number or an error.
* `Query(...) (luigi.Source, error)` query returns a source from which matching elements can be read.
* `Seq() luigi.Observable` can be used to be notified when new messages are appended.

### Receive Log

The main message storage is an _offset log_, implemented in [margaret/offset2](https://pkg.go.dev/go.cryptoscope.co/margaret/offset2). Since it's an append-based(`*`) format, every message has a unique sequence number.

Since go-ssb also not only supports the json-based feed format but also [gabbygrove](https://pkg.go.dev/go.mindeco.de/ssb-gabbygrove?tab=overview), the [multimsg package](https://pkg.go.dev/go.cryptoscope.co/ssb@v0.0.0-20201008153016-a00fdfeccb87/message/multimsg) is used to support both and more in the future.

(`*`): It's not strictly _append-only_ since it's also supported to null an entry with zeros or overwrite it with something of equal size (or smaller), which is useful for formats which support content deletion.

### Reduce indexes

The `graph` and `plugins2/names` packages show examples of indexes that boil down all the messages to a reduced state.

The `graph` package goes through all the `type:contact` messages and just stores the latest information for each pair of author and (un)followed or (un)blocked contact. This information can then directly be used by `plugins/friends` to answer muxrpc `friends.isFollowing` calls or in aggregate to build the whole trust graph for range lookups (hops between A and B) or the set of feeds that should be fetched.

The `plugins2/names` package implements the same muxrpc interface as [ssb-names](https://github.com/ssbc/ssb-names). It does a similar thing to `graph` but for `type:about` messages and the name, image and description information in them.

### MultiLog

We wanted to have a unified way to access different kinds of _sub logs_ (of the big receive log, for instance).

Enter [MultiLog](https://pkg.go.dev/go.cryptoscope.co/margaret/multilog), it returns `margaret.Log`s for specific addresses.

#### Implementation details

The first attempt we made stored an entry like `@publicKey:N -> ReceiveSeqM` (where N is the logical sequence entry on the feed and M the sequence of the message in the receive log) in a key-value database (`margaret/multilog/badger`). This worked but came with a lot of churn since the values are quite small, additionally keys are large and redundant.

A newer approach is to store the set of receivelog sequences as a special kind of [Bitmap](https://en.wikipedia.org/wiki/Bitmap). (tl;dr: You can imagine these as a compressed array of integers.) This is advantageous not only because we only store one bitmap for all the messages in a set (like `by author:x` or `by type:y`) but also because it allows us to use them as compound indexes since they can be logically combined via boolean algebra (`x AND y` gives us the intersection, `x OR y` gives us the union) since they all map to the same messages in the receive log.

#### Example Query by User

A simple example would be accessing a single users feed, identified by it's public key.

```go

func ex() {
  // let's create the default bot
  s := sbot.Sbot()

  // decode the string representation
  p13zsa, err := refs.ParseFeedRef("@p13zSAiOpguI9nsawkGijsnMfWmFd5rlUNpzekEE+vI=.ed25519")
  check(err)

  // get the librarian address for this reference and open
  justTheUsersLog, err := s.Users.Get(p13zsa.StoredAddr())
  check(err)

  // query the log
  src, err := justTheUsersLog.Query(margaret.Gte(3), margaret.Limit(5))
  check(err)

  // reading from this src directly will return
  // the sequence numbers in the receive log for messages 3 to 8.

  // there is a helping mapper to directly get the messages, though
  src = mutil.Indirect(s.ReceiveLog, justTheUserslog)

  for {
    v, err := src.Next(ctx)
    if err!=nil {
      if luigi.IsEOS(err) {
        break // end of stream
      }
      check(err)
    }

    msg := v.(ssb.Message)
    fmt.Println(msg.Key().Ref())
  }
}
```

## Links

Here are modules/packages that are not part of the core mono repository:

* The [secret-handshake](https://secret-handshake.club) key exchange is available as [secretstream](https://godoc.org/go.cryptoscope.co/secretstream)
* RPC interoperability with JS through [go-muxprc](https://godoc.org/go.cryptoscope.co/muxrpc)
* Embedded datastore, no external database required ([margaret](https://godoc.org/go.cryptoscope.co/margaret) and [librarian](https://godoc.org/go.cryptoscope.co/librarian), similar to [flumedb](https://github.com/flumedb/flumedb))
* [pull-stream](https://pull-stream.github.io)-like abstraction (called [luigi](https://godoc.org/go.cryptoscope.co/luigi)) to pipe between rpc and database.
* [go ssb-refs](https://godoc.org/go.mindeco.de/ssb-refs) utility types and parsing for message and feed identifiers.
* [go ssb-multiserver](https://godoc.org/go.mindeco.de/ssb-multiserver) [address](https://github.com/ssbc/multiserver-address) parsing.

## Contact

go-ssb and the surrounding modules were built by @cryptix with support from @keks.
To reach us either post to the #go-ssb channel on the ssb mainnet or mention us individually:

* cryptix: `@p13zSAiOpguI9nsawkGijsnMfWmFd5rlUNpzekEE+vI=.ed25519`
* keks: `@YXkE3TikkY4GFMX3lzXUllRkNTbj5E+604AkaO1xbz8=.ed25519`
