# go-ssb

## Introduction

Go-SSB implements most of the existing JavaScript functionality but is less modular, due to the typed nature of the language. Most of the code lives in a single repository: [go-ssb](https://github.com/cryptoscope/ssb) with a couple of packages split out, like `go-muxrpc` and `secretstream`.

The import path for the code is `go.cryptoscope.co/ssb`.

If you are not familiar with the [Go programming language](https://golang.org), you can get started with learning using the official [guided tour](https://tour.golang.org/welcome/1).


## Packages Overview

This section serves as a treasure map, outlining points of interest (important packages & folders) in the go-ssb [monorepo](https://github.com/cryptoscope/ssb/).

### [`cmd/`](https://github.com/cryptoscope/ssb/tree/master/cmd/)

By coding convention, this contains the executable tools. Most importantly these two:

* `go-sbot` which hosts the database and p2p server.
* `sbotcli` a muxrpc client to do common tasks, like publishing a message or querying messages.

### [`message/`](https://github.com/cryptoscope/ssb/tree/master/message/)

Message contains all the code to create and verify messages.

The current established format is

* [message](https://pkg.go.dev/go.cryptoscope.co/ssb/message) - abstract verification and publish helpers
* [message/legacy](https://pkg.go.dev/go.cryptoscope.co/ssb/message/legacy) - how to encode and verify [the current ssb messages](https://spec.scuttlebutt.nz/feed/messages.html)
* message/multimsg - encoding multiple kinds of messages to disk (see _Database Concept Overview_ below)

### [`plugins/`](https://github.com/cryptoscope/ssb/tree/master/plugins/)

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



### [`sbot/`](https://github.com/cryptoscope/ssb/tree/master/sbot/)

Package `sbot` ties together network, repo and plugins like graph and blobs into a large server that offers data-access APIs and background replication. It's name dates back to a time where ssb-server was still called scuttlebot, in short: sbot.

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


### Triplets: [`repo/`](https://github.com/cryptoscope/ssb/tree/master/repo), [`indexes/`](https://github.com/cryptoscope/ssb/tree/master/indexes) and [`multilogs/`](https://github.com/cryptoscope/ssb/tree/master/multilogs)

* `repo/` - contains utility modules to open offset logs and create different kinds of indexes
* `indexes/` - contains functions to create indexing  for `get(%ref) -> message`. Also contains a utility to open the contact trust graph using the `repo` and `graph` packages.
* `multilogs/` - currently contains the indexing functions for the `by user` and `private readable` multilogs (See the _MultiLog_ section in _Database concept overview_ for more)

### The rest of the gang

* [blobstore](https://pkg.go.dev/go.cryptoscope.co/ssb/blobstore) - the filesystem storage and sympathy managment for [ssb-blobs](https://github.com/ssbc/ssb-blobs)
* network - a utility module for dialing and listening to secret-handshake powered muxrpc connections
* [graph](https://pkg.go.dev/go.cryptoscope.co/ssb/graph) - derives trust/block relations by consuming type:contact message and offers lookup APIs between two feeds.
* [client](https://pkg.go.dev/go.cryptoscope.co/ssb/client) - a simple muxrpc interface to common ssb methods, similar to [ssb-client](https://github.com/ssbc/ssb-client)
* [invite](https://pkg.go.dev/go.cryptoscope.co/ssb/invite) - This package contains functions for parsing invite codes and dialing a pub as a guest to redeem a token. The muxrpc handlers and storage are found in `plugins/legacyinvite`.
* [private](https://pkg.go.dev/go.cryptoscope.co/ssb/private) - utility functions to de- and encrypted messages. Maps to `box()` and `unbox()` in [ssb-keys](https://github.com/ssbc/ssb-keys).

### On Testing: [`tests/`](https://github.com/cryptoscope/ssb/tree/master/tests/)

While most folders contain `_test.go` files where functionality is tested as unit tests, this folder
contains sepcial code to run tests against the JavaScript implementation.

## Database: Conceptual overview

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

### [`margaret/multilog`](https://pkg.go.dev/go.cryptoscope.co/margaret)

We wanted to have a unified way to access different kinds of sublogs, that is, logs-as-slices into a larger log. One example: accessing parts of the big receive log (the posts by one particular feed, for instance).

Enter [MultiLog](https://pkg.go.dev/go.cryptoscope.co/margaret/multilog), which returns `margaret.Log`s for specific addresses.

Package `multilog` is one of a few ways that go-ssb creates an overarching structure (indexes and views) of the
messages being received from all over the scuttleverse. Another prominent indexing solution is
the [`margaret/indexes`](https://godocs.io/go.cryptoscope.co/margaret/indexes) package, which
more or less functions as a key-value store. One of the differences between `margaret/indexes` and
`margaret/multilog`: the former stores arbitrary data (think: strings of your choosing), while
the latter only stores (references to) ssb messages.

#### A `multilog` explanation

One way to think about the `multilog` package: 

A multilog can be seen as a _[tree](https://en.wikipedia.org/wiki/Tree_(data_structure))-like_
index. The multilog itself is a log—or the root of the tree—which leads to other logs. These
other logs are called _sublogs_, and each sublog has many leaves. Each leaf corresponds to some
message. What type of message is stored depends on the particular type of data being indexed by the multilog in question (i.e. all `type:contact` messages, all messages by `<peer>`, etc.).

This multilog tree can then be thought of as having a depth of 2: 

```
root (multilog)              // depth 0
    -> sublog (margaret.Log) // depth 1
        -> leaf (message)    // depth 2
        -> leaf (message)
        -> leaf (message)
```

In other words: one multilog has many sublogs, and each sublog has many messages.

#### Example: Query by User

A standalone example of using `multilog` would be accessing a single users feed, as identified by its public key.

```go

func ex() {
  // let's create the default bot
  s := sbot.Sbot()

  // decode the string representation
  p13zsa, err := refs.ParseFeedRef("@p13zSAiOpguI9nsawkGijsnMfWmFd5rlUNpzekEE+vI=.ed25519")
  check(err)

  // get the address for this reference and open the log
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

#### Multilog implementation details

The first attempt we made stored an entry like `@publicKey:N -> ReceiveSeqM` (where N is the logical sequence entry on the feed and M the sequence of the message in the receive log) in a key-value database (`margaret/multilog/badger`). This worked but came with a lot of churn since the values are quite small, additionally keys are large and redundant.

A newer approach is to store the set of receivelog sequences as a special kind of [Bitmap](https://en.wikipedia.org/wiki/Bitmap). (tl;dr: You can imagine these as a compressed array of integers.) This is advantageous not only because we only store one bitmap for all the messages in a set (like `by author:x` or `by type:y`) but also because it allows us to use them as compound indexes since they can be logically combined via boolean algebra (`x AND y` gives us the intersection, `x OR y` gives us the union) since they all map to the same messages in the receive log.

## Snippets
Code examples and snippets for commonly useful tasks, especially when dealing with
[metafeeds](https://github.com/ssb-ngi-pointer/ssb-meta-feeds-spec). The examples are
incomplete snippets, aimed at helping get developers over conceptuals bumps by demonstrating
the gist of a task without creating a full on example program.

**Note**: Some of the snippets below are demonstrating the expected behaviour using the assertion package
`testify`:

```golang
import `github.com/stretchr/testify/require`
// ...
func TestAnExampleFunction (t *testing)
r := require.New(t)
```

#### Signing an announcement with the root metafeed's key
```golang
announcement := legacy.NewMetafeedAnnounce(kpMetafeed.ID(), kpMain.ID())
signedAnnouncement, err := announcement.Sign(kpMetafeed.PrivateKey, nil)
r.NoError(err)
```


#### Verify mf announcement is correct
```golang
_, ok := legacy.VerifyMetafeedAnnounce(signedMsg, theUpgradingOne.ID(), &hmacSecret)
r.True(ok, "verify failed")
```


#### Creating an announcement message
```golang
ma := legacy.NewMetafeedAnnounce(bot.KeyPair.ID(), subfeed)

signedMsg, err := ma.Sign(bot.KeyPair.Secret(), nil)
r.NoError(err)

ref, err := bot.MetaFeeds.Publish(subfeed, signedMsg)
r.NoError(err)

msg, err := bot.Get(ref)
r.NoError(err)
t.Log("content:", string(msg.ContentBytes()))

mm, ok := msg.(*multimsg.MultiMessage)
r.True(ok, "wrong type: %T", msg)

lm, ok := mm.AsLegacy()
r.True(ok)
```

#### Add a main feed to the metafeed `metafeed/add/existing`
```golang
mfAddExisting := metamngmt.NewAddExistingMessage(kpMetafeed.ID(), kpMain.ID(), "main")
mfAddExisting.Tangles["metafeed"] = refs.TanglePoint{Root: nil, Previous: nil}

// sign the bendybutt message with mf.Secret + main.Secret
signedAddExistingContent, err := metafeed.SubSignContent(kpMain.Secret(), mfAddExisting)
if err != nil {
	return err
}
mf.publish.Append(signedAddExistingContent)
```

#### Register indexes (and then get them)
```golang
err = bot.MetaFeeds.RegisterIndex(mfId, mainFeedRef, "about")
r.NoError(err)

err = bot.MetaFeeds.RegisterIndex(mfId, mainFeedRef, "contact")
r.NoError(err)

// get the actual index feeds so we can assert on them
aboutIndexId, err := bot.MetaFeeds.GetOrCreateIndex(mfId, mainFeedRef, "index", "about")
r.NoError(err)
aboutIndex := getFeed(aboutIndexId)
checkSeq(aboutIndex, int(margaret.SeqEmpty))

contactIndexId, err := bot.MetaFeeds.GetOrCreateIndex(mfId, mainFeedRef, "index", "contact")
r.NoError(err)
contactIndex := getFeed(contactIndexId)
checkSeq(contactIndex, int(margaret.SeqEmpty))
```

#### Get a feed using a feedref
```golang
func getFeed(bot *sbot.Sbot, feedID refs.FeedRef) (margaret.Log, error) {
	feed, err := bot.Users.Get(storedrefs.Feed(feedID))
	if err != nil {
		return nil, fmt.Errorf("get feed failed", err)
	}

	// convert from log of seqnos-in-rxlog to log of refs.Message and return
	return mutil.Indirect(bot.ReceiveLog, feed), nil
}
```

#### Debug print an entire log using its feedref
```golang
// error wrap - a helper util
// string header will be prefixed before each message. typically it is the context we're generating errors within.
// msg is the specific message, err is the error (if passed)
func ew(header string) func(msg string, err ...error) error {
	return func(msg string, err ...error) error {
		if len(err) > 0 {
			return fmt.Errorf("[gossb: %s] %s (%w)", header, msg, err[0])
		}
		return fmt.Errorf("[gossb: %s] %s", header, msg)
	}
}

func printLog(bot *sbot.Sbot, feedid refs.FeedRef) error {
	e := ew("print log")

	l, err := getFeed(bot, feedid)
	if err != nil {
		return e(fmt.Sprintf("couldnt get log %s", feedid.ShortSigil()), err)
	}

	src, err := l.Query()
	if err != nil {
		return e("failed to query log", err)
	}

	seq := l.Seq()
	i := int64(0)
	inform("last seqno:", seq)

	for {
		v, err := src.Next(context.TODO())
		if luigi.IsEOS(err) {
			break
		}
		inform("value:", v)
		mm, ok := v.(refs.Message)
		if !ok {
			return e(fmt.Sprintf("expected %T to be a refs.Message (wrong log type? missing indirection to receive log?)", v))
		}

		fmt.Printf("log seq: %d - %s:%d (%s)\n",
			i,
			mm.Author().ShortSigil(),
			mm.Seq(),
			mm.Key().ShortSigil())

		b := mm.ContentBytes()
		if n := len(b); n > 128 {
			fmt.Println("truncating", n, " to last 32 bytes")
			b = b[len(b)-32:]
		}
		fmt.Printf("\n%s\n", hex.Dump(b))

		i++
	}

	// margaret is 0-indexed
	seq++
	if seq != i {
		return e(fmt.Sprintf("seq differs from iterated count: %d vs %d", seq, i))
	}
	return nil
}
```

## Links

Listed below are modules and packages that are not part of the [`go.cryptoscope.co/ssb`](https://github.com/cryptoscope/ssb/) monorepo:

* The [secret-handshake](https://secret-handshake.club) key exchange is available as [secretstream](https://godoc.org/go.cryptoscope.co/secretstream)
* RPC interoperability with JS through [go-muxprc](https://godoc.org/go.cryptoscope.co/muxrpc)
* Embedded datastore, no external database required ([margaret](https://godoc.org/go.cryptoscope.co/margaret) and [`margaret/indexes`](https://godoc.org/go.cryptoscope.co/margaret/indexes), similar to [flumedb](https://github.com/flumedb/flumedb))
* [pull-stream](https://pull-stream.github.io)-like abstraction (called [luigi](https://godoc.org/go.cryptoscope.co/luigi)) to pipe between rpc and database.
* [go ssb-refs](https://godoc.org/go.mindeco.de/ssb-refs) utility types and parsing for message and feed identifiers.
* [go ssb-multiserver](https://godoc.org/go.mindeco.de/ssb-multiserver) [address](https://github.com/ssbc/multiserver-address) parsing.

## Contact

go-ssb and the surrounding modules were built by @cryptix with support from @keks.
To reach us either post to the #go-ssb channel on the ssb mainnet or mention us individually:

* cryptix: `@p13zSAiOpguI9nsawkGijsnMfWmFd5rlUNpzekEE+vI=.ed25519`
* keks: `@YXkE3TikkY4GFMX3lzXUllRkNTbj5E+604AkaO1xbz8=.ed25519`
