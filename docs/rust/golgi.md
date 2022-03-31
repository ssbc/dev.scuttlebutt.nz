## Golgi (client library)

### Introduction

[Golgi](http://golgi.mycelial.technology) is an asynchronous, experimental Scuttlebutt client that aims to facilitate Scuttlebutt application development. It provides a high-level API for interacting with an sbot instance and uses the [kuska-ssb](https://github.com/Kuska-ssb) libraries to make RPC calls. It is a great fit if you wish to build a simple client that communicates with a standalone sbot (Scuttlebutt node or server). Development efforts are currently oriented towards [go-sbot](https://github.com/cryptoscope/ssb) interoperability.

This guide will introduce some of the features of golgi which may be useful for client developers. Visit the [examples directory](https://git.coopcloud.tech/golgi-ssb/golgi/src/branch/main/examples) in the golgi repository for more comprehensive examples.

### Features

Golgi offers the ability to invoke individual RPC methods while also providing a number of convenience methods which may involve multiple RPC calls and / or the processing of data received from those calls.

Features include the ability to publish messages of various kinds; to retrieve messages (e.g. `about` and `description` messages) and formulate queries; to follow, unfollow, block and unblock a peer; to query the social graph; and to generate pub invite codes.

-----

Note: If you are not familiar with the [Rust programming language](https://www.rust-lang.org/), please visit the [Getting started](https://www.rust-lang.org/learn/get-started) page to learn more. The [Rust programming language book](https://doc.rust-lang.org/book/) and [Rust By Example](https://doc.rust-lang.org/stable/rust-by-example/) are both excellent learning guides.

You may also benefit from reading [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/) and the [async-std book](https://book.async.rs/).

-----

### Prerequisites

You will need to be running a go-sbot server (with which to connect using golgi). See the [README](https://github.com/cryptoscope/ssb) for installation details.

Note: Support for JS interoperability will likely be added in the near-future.

Add the [golgi crate](https://crates.io/crates/golgi) as a dependency in the `Cargo.toml` manifest of your project, as well as the [async-std](https://crates.io/crates/async-std) and [futures](https://crates.io/crates/futures) crates (required if you're going to use the stream methods provided by golgi):

```toml
async-std = "1.10.0"
futures = "0.3.21"
golgi = "0.1"
```

You may wish to keep the [golgi API documentation](https://docs.rs/golgi/0.1.0/golgi/) open in a tab while working with golgi. Each method is documented in detail.

### Sbot Connection

Before we can communicate with the go-sbot, we first need to initialise a connection. We do this by calling the `init()` method on the `Sbot` `struct`. In this case, we are passing `None` for the `ip_port` parameter and `None` for the `net_id` parameter, resulting in a default of `127.0.0.1:8008` and the standard network key (aka. caps key) for the Scuttleverse:

```rust
use golgi::{GolgiError, Sbot};

let mut sbot_client = Sbot::init(None, None).await?;
```

Notice the `await` call in the snippet above? This is required due to the asynchronous nature of golgi and its methods.

### Whoami

Once we have instantiated the `Sbot` we can begin calling methods. The simplest of these is `whoami()`. If successful, it returns the public key of the local sbot; this is "our" Scuttlebutt identity:

```rust
let id = sbot_client.whoami().await?;

println!("{}", id);

// @1vxS6DMi7z9uJIQG33W7mlsv21GZIbOpmWE1QEcn9oY=.ed25519
```

### Publishing Messages

One of the first things we tend to do when creating a new Scuttlebutt account is set a name and description for the identity. This is accomplished by publishing particular kinds of messages. Golgi provides convenient methods to achieve this:

```rust
let name = "endoplasmic reticulum";

let name_msg_reference = sbot_client.publish_name(name).await?;

let description = "A tubular network of membranes found within the cytoplasm of the eukaryotic cell.";

let description_msg_reference = sbot_client.publish_description(description).await?;
```

All publish-related methods in golgi return a reference (cypherlink) to the published message, unless an error is encountered in the process.

A convenient method is also provided for publishing public messages:

```rust
let text = "Once I dreamed I was a tree; with roots above and branches below.";

// We can also match on the returned `Result`, rather than using the `?` operator.
match sbot_client.publish_post(text).await {
    Ok(ref) => println!("post published, here's the reference: {}", ref),
    Err(e) => eprintln!("failed to publish post: {}", e),
}
``` 

Golgi also exposes finer-grained message posting options via the `publish()` method. This requires an `SsbMessageContent` type to be composed. Since `SsbMessageContent` is a type alias for the kuska-defined `TypedMessage` `enum`, you may wish to view the [source](https://github.com/Kuska-ssb/ssb/blob/master/src/api/dto/content.rs#L103) for a complete type definition.

Here is how one would subscribe to a Scuttlebutt channel:

```rust
// Compose a `Channel` variant of the `SsbMessageContent` enum.
let subscription = SsbMessageContent::Channel {
    // Define the channel to which we are subscribing.
    channel: "#myco".to_string(),
    // Set to `true` to subscribe or `false` to unsubscribe.
    subscribed: true,
};

let msg_ref = sbot_client.publish(subscription).await?;
``` 

### Invites

Golgi provides a means of both creating and redeeming invite codes. Invite code creation is one of the key features of a Scuttlebutt pub and is useful for onboarding peers. When running a pub, it's important to first publish a `pub address` message with the connection details of the sbot:

```rust
// Compose a `pub` address type message.
let pub_address_msg = SsbMessageContent::Pub {
    address: Some(PubAddress {
        // Host name (can be an IP address if onboarding over WiFi).
        host: Some("pub.cellular.io".to_string()),
        // Port.
        port: 8009,
        // Public key.
        key: id,
    }),
};

// Publish the `pub` address message.
// This step is required for successful invite code usage.
let pub_msg_ref = sbot_client.publish(pub_address_msg).await?;
```

Invite codes can then be generated, with the option to define the number of times the resulting code can be used:

```rust
// Generate an invite code that can be used 1 time.
let invite_code = sbot_client.invite_create(1).await?;

// Print the invite code to `stdout`.
println!("{:?}", invite_code);

// pub.cellular.io:8009:@UFpbQfKTeaQrY4XIDNbl9JsTSGZ8sYwQYwQVAD6JaME=.ed25519~cEpDvNLI3HImUmJtpaqp73SYgx78HT2feWMyieez9YE=

// Generate an invite code that can be used 7 times.
let invite_code_2 = sbot_client.invite_create(7).await?;

// Print the invite code to `stdout`.
println!("{:?}", invite_code_2);

// pub.cellular.io:8009:@0iMa+vP7B2aMrV3dzRxlch/iqZn/UM3S3Oo2oVeILY8=.ed25519~Qm0idPGjgIktjHtgPn/pkqSsIXYcU/4dbtl01FPz9/I=
```

Note that invite codes _can_ still be generated if the pub address has not been defined; the generated code will simply not include a hostname / port, for example:

`[::]:8008:@UFpbQfKTeaQrY4XIDNbl9JsTSGZ8sYwQYwQVAD6JaME=.ed25519~tpKrwY5C+Tv6ReOREAcu/F9RPVJSu4fbAfgVahX6cXc=`

Invite codes are redeemed in a similar fashion:

```rust
// Define an invite code.
let test_invite = "127.0.0.1:8009:@0iMa+vP7B2aMrV3dzRxlch/iqZn/UM3S3Oo2oVeILY8=.ed25519~Qm0idPGjgIktjHtgPn/pkqSsIXYcU/4dbtl01FPz9/I=";

// Redeem an invite code, initiating a mutual follow between the local
// identity and the identity which generated the code (`@0iMa+vP...`).
let mref = sbot_client.invite_use(test_invite).await?;

// Print the message reference to `stdout`.
println!("mref: {:?}", mref);
```

### Social Relationships

Now that we know how to publish messages and deal with invites we can move on to defining and querying social relationships.

Convenience methods are exposed to allow simple following and blocking of peers:

```rust
let ssb_id = "@zqshk7o2Rpd/OaZ/MxH6xXONgonP1jH+edK9+GZb/NY=.ed25519";

// Attempt to follow a peer (`@zqshk7...`) and match on the result.
match sbot_client.follow(ssb_id).await {
    Ok(msg_ref) => {
        println!("follow msg reference is: {}", msg_ref)
    },
    Err(e) => eprintln!("failed to follow {}: {}", ssb_id, e)
}

let ssb_id_2 = "@3QoWCcy46X9a4jTnOl8m3+n1gKfbsukWuODDxNGN0W8=.ed25519";

// Attempt to block a peer and return the block message reference 
// (or an error, if one occurs).
let block_ref = sbot_client.block(ssb_id_2).await?;
```

Direct methods for unfollowing and unblocking peers are not currently provided by golgi. These actions can be achieved via the `set_relationship()` method:

```rust
let ssb_id = "@zqshk7o2Rpd/OaZ/MxH6xXONgonP1jH+edK9+GZb/NY=.ed25519";
    
let following = false;
let blocking = false;

// Set the relationship of the local identity to the `ssb_id`. 
// In this case, the `set_relationship` method publishes a `contact`
// message which defines following as `false` and blocking as `false`.
// A message reference is returned for the published `contact` message.
let msg_ref =  sbot_client.set_relationship(ssb_id, following, blocking).await?;
```

Now that we're able to define our relationship with other peers, we may wish to query the social relationships within our vicinity. Golgi provides the `friends_hops()` method which offers a means of retrieving a list of peers within a specified hops range:

```rust
// The `friends_hops()` method expects `FriendsHops` as a parameter.
use api::friends::FriendsHops;

let peer_query = FriendsHops {
    // The hops range of our query.
    max: 0,
    // A peer identity which defines the perspective or starting point of the 
    // query. This defaults to the local identity when defined as `None`.
    start: None,
    // Reverse the perspective of the query, ie. looking toward the `start` 
    // identity instead of the default outward perspective.
    // Setting `max: 0` and `reverse: Some(true)` will return a list of 
    // followers (note: this is not currently implemented in go-sbot and will 
    // therefore fail to result in the intended behaviour). 
    reverse: None,
};

let follows = sbot_client.friends_hops(peer_query).await?;

println!("follows: {:?}", follows);

// follows: ["\"@5Pt3dKy2HTJ0mWuS78oIiklIX0gBz6BTfEnXsbvke9c=.ed25519\"\n", "\"@HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519\"\n"]
```

The `get_follows()` method offers a more convenient way to perform the peer query show above:

```rust
let follows = sbot_client.get_follows().await?;

if follows.is_empty() {
    println!("we do not follow any peers")
} else {
    follows.iter().for_each(|peer| println!("we follow {}", peer))
}
```

It is also possible to determine the relationship between a single peer and another, ie. is a peer following or blocking another peer?:

```rust
// The `friends_is_following()` and `friends_is_blocking()` methods expect 
// `RelationshipQuery` as a parameter.
use api::friends::RelationshipQuery;

let peer_a = String::new("@zqshk7o2Rpd/OaZ/MxH6xXONgonP1jH+edK9+GZb/NY=.ed25519");
let peer_b = String::new("@3QoWCcy46X9a4jTnOl8m3+n1gKfbsukWuODDxNGN0W8=.ed25519");

let friend_query = RelationshipQuery {
    source: peer_a.clone(),
    dest: peer_b.clone(),
};

match sbot_client.friends_is_following(friend_query).await {
    Ok(following) if following == "true" => {
        println!("{} is following {}", peer_a, peer_b)
    },
    Ok(_) => println!("{} is not following {}", peer_a, peer_b),
    Err(e) => eprintln!(
        "failed to query relationship status for {} and {}: {}", peer_a, peer_b, e
    )
}

let block_query = RelationshipQuery {
    source: peer_a,
    dest: peer_b,
};

match sbot_client.friends_is_blocking(block_query).await {
    Ok(blocking) if blocking == "true" => {
        println!("{} is blocking {}", peer_a, peer_b)
    },
    Ok(_) => println!("{} is not blocking {}", peer_a, peer_b),
    Err(e) => eprintln!(
        "failed to query relationship status for {} and {}: {}", peer_a, peer_b, e
    )
}
```

### Peer Data

Now might be a good time to start learning more about our peers. For example, what are their names and how do they describe themselves? As is common throughout the golgi API, we have access to lower-level methods as well as convenience methods for ease of use.

One such convenience method is `get_profile_info()`, which returns the latest `name`, `description` and `image` reference for a peer:

```rust
let ssb_id = "@zqshk7o2Rpd/OaZ/MxH6xXONgonP1jH+edK9+GZb/NY=.ed25519";

let profile_info = sbot_client.get_profile_info(ssb_id).await?;

// The type of `profile_info` is a HashMap. The `get()` method
// will either return `None` or `Some(value)`.
let name = profile_info.get("name");
let description = profile_info.get("description");
let image = profile_info.get("image");

// Match on the `Option` types for the variables assigned above.
match (name, description, image) {
    (Some(name), Some(desc), Some(image)) => {
        println!(
            "peer {} is named {}. their profile image blob reference is {} and they describe themself as follows: {}",
            ssb_id, name, image, desc,
        )
    },
    (_, _, _) => {
        eprintln!("failed to retrieve all profile info values")
    }
}
```

Methods also exist for retrieving individual `about` values: `get_name(ssb_id)`, `get_description(ssb_id)` and `get_name_and_image(ssb_id)`. These methods act in the same way as `get_profile_info()` and are thus not illustrated here (see the API documentation for per-method example code).

The `get_about_message_stream()` method allows for more advanced interactions with `about` message types. This method returns a stream of message values (`SsbMessageValue`) which can then be handled as desired:

```rust
// These traits are required when dealing with streams.
use async_std::stream::{Stream, StreamExt};

let ssb_id = "@zqshk7o2Rpd/OaZ/MxH6xXONgonP1jH+edK9+GZb/NY=.ed25519";

let about_message_stream = sbot_client.get_about_message_stream(ssb_id).await?;

// Pin the stream to the stack.
// This allows us to iterate on the stream.
futures::pin_mut!(about_message_stream);

// Here we are using `for_each()` but we could just as easily use `map()`, 
// `filter()`, `find()` etc.
about_message_stream.for_each(|msg| {
    match msg {
        Ok(val) => println!("msg value: {:?}", val),
        Err(e) => eprintln!("error: {}", e),
    }
}).await;
```

### Subset Queries

Of course, there are more message types than just `about` messages. If we wish to query the sbot database for messages of a particular type or from a particular author, or even some combination of author and type, we can use the `get_subset_stream()` method (recently introduced during the SSB NGI Pointer project). It is recommended to read the [Subset replication for SSB](https://github.com/ssb-ngi-pointer/ssb-subset-replication-spec) document, paying particular attention to the section on [ssb-ql-1](https://github.com/ssb-ngi-pointer/ssb-subset-replication-spec#ssb-ql-1):

```rust
// This trait is required when dealing with streams.
use async_std::stream::StreamExt;
// These types are required to compose subset queries.
use golgi::{
    api::get_subset::{
        SubsetQuery,
        SubsetQueryOptions
    }
};

let post_query = SubsetQuery::Type {
    op: "type".to_string(),
    string: "post".to_string()
};

let post_query_opts = SubsetQueryOptions {
    descending: Some(true),
    keys: None,
    page_limit: Some(5),
};

// Return 5 `post` type messages from any author in descending order.
let query_stream = sbot_client
    .get_subset_stream(post_query, Some(post_query_opts))
    .await?;

// Iterate over the stream and pretty-print each returned message
// value while ignoring any errors.
query_stream.for_each(|msg| match msg {
    Ok(val) => println!("{:#?}", val),
    Err(_) => (),
});
```

An author query would be composed in a similar fashion, with the main difference being the `SubsetQuery` variant:

```rust
let author_query = SubsetQuery::Author {
    op: "author".to_string(),
    feed: "@HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519".to_string(),
};

// Return all messages authored by `@Heqy9...`.
let author_query_opts = SubsetQueryOptions {
    descending: None,
    keys: None,
    page_limit: None,
};

let query_stream = sbot_client
    .get_subset_stream(author_query, Some(author_query_opts))
    .await?;
```

More complex queries can also be composed using the `and` and `or` operators. See the `SubsetQuery` type definition in the golgi docs and the subset query specification document linked above.

### History Stream

Another way of returning messages from a single author is provided by the `create_history_stream()` method. This method takes a single SSB ID (public key) and returns all available messages authored by that identity.

Note: this method does not currently support optional parameters such as `reverse`, `seq` and `live`. These will likely be added in the future.

```rust
// This trait is required when dealing with streams.
use async_std::stream::StreamExt;

let ssb_id = "@zqshk7o2Rpd/OaZ/MxH6xXONgonP1jH+edK9+GZb/NY=.ed25519".to_string();

let history_stream = sbot_client.create_history_stream(ssb_id).await?;

history_stream.for_each(|msg| {
    match msg {
        Ok(val) => println!("msg value: {:?}", val),
        Err(e) => eprintln!("error: {}", e),
    }
}).await;
```

### Conclusion

With a bit of luck, you should now have a basic understanding of golgi and enough code examples to start building your own unique Scuttlebutt client application. Golgi is still very young and is expected to evolve rapidly as we put it into action.

Please feel free to contact @notplants or @glyph (contact info below) if you have any questions. We would love to hear your feedback on this guide and any suggestions you may have! You're also invited to open issues on the [git repo](https://git.coopcloud.tech/golgi-ssb/golgi) for any bugs or feature requests you may have.

### Contact

Golgi is authored by [@notplants](https://mfowler.info/) and [@glyph](https://mycelial.technology). This guide was written by @glyph. Both can be found in the Scuttleverse:

* notplants: `@5Pt3dKy2HTJ0mWuS78oIiklIX0gBz6BTfEnXsbvke9c=.ed25519`
* glyph: `@HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519`
