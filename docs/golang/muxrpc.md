## `["blobs", "get"]`

### Specification

* Description: Streams the contents of the file.
* Endpoint: `["blobs", "get"]`
* RPC Type: `source`
* Arguments: Must be an array containing strings (blob references).
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| N/A | array | yes | Blob reference (`&...sha256`) |

### Options

N/A

### Request

```json
{
  "name": ["blobs", "get"],
  "type": "async",
  "args": ["&grLTZFapgHZHXRYh1zgz2bTuDelottGZSfogKauo/fk=.sha256"]
}
```

### Examples

#### `sbotcli`

```
sbotcli blobs get "&grLTZFapgHZHXRYh1zgz2bTuDelottGZSfogKauo/fk=.sha256" > blob_file
```

## `["blobs", "createWants"]`

### Specification

* Description: Try to get a blob from other peers. 
* Endpoint: `["blobs", "createWants"]`
* RPC Type: `source`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| N/A | array | yes | Blob reference (`&...sha256`) |

### Options

N/A

### Request

```json
{
  "name": ["blobs", "createWants"],
  "type": "async",
  "args": ["&grLTZFapgHZHXRYh1zgz2bTuDelottGZSfogKauo/fk=.sha256"]
}
```

### Examples

#### `sbotcli`

```
sbotcli blobs want "&hB2vsBGwqPAfkBQ5IQGIrLfHXzytmExYC3iJ6FC08F8=.sha256"
```

## `["createFeedStream"]`

### Specification

* Description: Fetch all messages from the local database, ordered by message timestamps.
* Endpoint: `["createFeedStream"]`
* RPC Type: `source`
* Arguments: An array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| limit | int | no | Limit the number of results collected by this stream. This number represents a maximum number of results and may not be reached if you get to the end of the data first. A value of -1 means there is no limit. When `reverse=true` the highest keys will be returned instead of the lowest keys |
| seq | int | no | Start from this sequence|
| gt | int | no | "Greater than", define the lower bound of the range to be streamed |
| lt | int | no | "Less than", define the upper bound of the range to be streamed |
| reverse | bool | no | set true and the stream output will be reversed |
| live | bool | no | Keep the stream open and emit new messages as they are received |
| keys | bool | no | Whether the data event should contain keys. If set to true and values set to false then data events will simply be keys, rather than objects with a key property |
| values | bool | no | Whether the data event should contain values. If set to true and keys set to false then data events will simply be values, rather than objects with a value property |
| private | bool | no | Messages are private |

### Options

N/A

### Request

```json
{
  "name": ["createFeedStream"],
  "type": "source",
  "args": [{
    "limit": 3
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli sorted --limit 3
```

## `["createHistoryStream"]`

### Specification

* Description: Fetch all messages authored by the local keypair / author.
* Endpoint: `["createHistoryStream"]`
* RPC Type: `source`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| limit | int | no | Limit the number of results collected by this stream. This number represents a maximum number of results and may not be reached if you get to the end of the data first. A value of -1 means there is no limit. When `reverse=true` the highest keys will be returned instead of the lowest keys |
| seq | int | no | Start from this sequence|
| reverse | bool | no | set true and the stream output will be reversed |
| live | bool | no | Keep the stream open and emit new messages as they are received |
| keys | bool | no | Whether the data event should contain keys. If set to true and values set to false then data events will simply be keys, rather than objects with a key property |
| values | bool | no | Whether the data event should contain values. If set to true and keys set to false then data events will simply be values, rather than objects with a value property |
| private | bool | no | Messages are private |
| id | string | no | Feed reference (`@...ed25519`) |
| asJSON | bool | no | Return JSON as output format |

### Options

N/A

### Request

```json
{
  "name": ["createHistoryStream"],
  "type": "source",
  "args": [{
    "limit": 3
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli hist --limit 3
```

## `["friends", "blocks"]`

### Specification

* Description: List all peers blocked by the given feed ID.
* Endpoint: `["friends", "blocks"]`
* RPC Type: `source`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| who | string | yes | Feed reference (`@...ed25519`) |

### Options

N/A

### Request

```json
{
  "name": ["friends", "blocks"],
  "type": "source",
  "args": [{
    "who": "@fGWzOR/FXU3Acbn4P65CpMewJIynFyqocvfLAyJdDno=.ed25519"
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli friends blocks @fGWzOR/FXU3Acbn4P65CpMewJIynFyqocvfLAyJdDno=.ed25519
```

## `["friends", "hops"]`

### Specification

* Description: List all peers within the hops range of the given feed ID.
* Endpoint: `["friends", "hops"]`
* RPC Type: `source`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| start | string | yes | Feed reference (`@...ed25519`)|
| max | int | no | Hops range |

### Options

N/A

### Request

```json
{
  "name": ["friends", "hops"],
  "type": "source",
  "args": [{
    "start": "@HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519",
    "max": 0
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli friends hops --dist 0 @HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519
```

## `["friends", "isBlocking"]`

### Specification

* Description: List all peers blocked by the given feed ID.
* Endpoint: `["friends", "isBlocking"]`
* RPC Type: `source`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| who | string | yes | Feed reference (`@...ed25519`) |

### Options

N/A

### Request

```json
{
  "name": ["friends", "isBlocking"],
  "type": "source",
  "args": [{
    "who": "@fGWzOR/FXU3Acbn4P65CpMewJIynFyqocvfLAyJdDno=.ed25519"
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli friends blocks @fGWzOR/FXU3Acbn4P65CpMewJIynFyqocvfLAyJdDno=.ed25519
```

## `["friends", "isFollowing"]`

### Specification

* Description: Check if a feed is friends with another (follows back). 
* Endpoint:  `["friends", "isFollowing"]`
* RPC Type: `async`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| source | string | yes | Source feed |
| source | string | yes | Destination feed |

### Options

N/A

### Request

```json
{
  "name": ["friends", "isFollowing"],
  "type": "async",
  "args": [{
    "source": "@HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519",
    "dest": "@mfY4X9Gob0w2oVfFv+CpX56PfL0GZ2RNQkc51SJlMvc=.ed25519"
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli friends isFollowing \
  @HEqy940T6uB+T+d9Jaa58aNfRzLx9eRWqkZljBmnkmk=.ed25519 \
  @mfY4X9Gob0w2oVfFv+CpX56PfL0GZ2RNQkc51SJlMvc=.ed25519
```

## `["get"]`

### Specification

* Description: Get a single message from the local database by key
* Endpoint: `["get"]`
* RPC Type: `async`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| id | string | yes | Message reference (`%...sha256`) |
| private | bool | no | Is a private message |

### Options

N/A

### Request

```json
{
  "name": ["get"],
  "type": "async",
  "args": [
    {
      "id": "%R07yiTZwvlquWkDMFQC9VktEzxUxo/psuIjYwQiVVms=.sha256",
      "private": false
    }
  ]
}
```

### Examples

#### `sbotcli`

```
sbotcli get %R07yiTZwvlquWkDMFQC9VktEzxUxo/psuIjYwQiVVms=.sha256
```

## `["invite", "create"]`

### Specification

* Description: Register and return an invite for somebody else to accept.
* Endpoint: `["invite", "create"]`
* RPC Type: `async`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| uses | int | no | The number of times the invite can be used |
| note | string | no | A text note to help organise invites |

### Options

N/A

### Request

```json
{
  "name": ["invite", "create"],
  "type": "async",
  "args": [{
    "uses": 2,
    "note": "my cool friend"
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli invite create --uses 7
```

## `["invite", "use"]`

### Specification

* Description: Use an invite code. 
* Endpoint: `["invite", "use"]`
* RPC Type: `async`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| feed | string | yes | The feed reference of the one invited (`@...sha256`) |

### Options

N/A

### Request

```json
{
  "name": ["invite", "use"],
  "type": "async",
  "args": [{
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli invite accept \
  "[::]:8008:@Vv3Pe1prVGEShE2mrZhsE8+hEoz8m+xiMEW0l1XLmpE=.ed25519~M62f0k+Y+crWvb/bevsvMumoiNLCGdWJ8OrF/JdDASU=" \
  "@Vv3Pe1prVGEShE2mrZhsE8+hEoz8m+xiMEW0l1XLmpE=.ed25519"
```

## `["names", "get"]`

### Specification

* Description: Retrieve list of names for an identity.
* Endpoint: `["names", "get"]`
* RPC Type: `async`
* Arguments: N/A
* Options: N/A

### Arguments

N/A

### Options

N/A

### Request

```json
{
  "name": ["names", "get"],
  "type": "async"
}
```

### Examples

N/A

## `["names", "getImageFor"]`

### Specification

* Description: Retrieve the profile avatar for an identity.
* Endpoint: `["names", "getImageFor"]`
* RPC Type: `async`
* Arguments: Must be a string.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| N/A | string | yes | Feed reference (`@...ed25519`) |

### Options

N/A

### Request

```json
{
  "name": ["names", "getImageFor"],
  "type": "async",
  "args": "@r6Lzb9OT3/dlVYNDTABmsF+HWnhBsA1twZaobYhjVUY=.ed25519"
}
```

### Examples

N/A

## `["names", "getSignifier"]`

### Specification

* Description: Retrieve signifier for identity.
* Endpoint: `["names", "getSignifier"]`
* RPC Type: `async`
* Arguments: Must be a string.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| N/A | string | yes | Feed reference (`@...ed25519`) |

### Options

N/A

### Request

```json
{
  "name": ["names", "getSignifier"],
  "type": "async",
  "args": "@r6Lzb9OT3/dlVYNDTABmsF+HWnhBsA1twZaobYhjVUY=.ed25519"
}
```

### Examples

N/A

## `["partialReplication", "getSubset"]`

### Specification

* Description: Fetch subsets of messages from the log
* Endpoint: `["partialReplication", "getSubset"]`
* RPC Type: `source`
* Arguments: An array containing exactly one object.
* Options: An array containing exactly one object.

### Arguments

> At least one argument is required.

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| type | json | no | Filter by type of message (e.g. `{ op: "type", string: "post"}`) |
| author | json | no | Filter by author of feed (e.g. `{ op: "author", string: "<@...ed25519>"}`)|
| and | json | no | Use logical AND on subset query (e.g. `{"op":"and","args":[{"op":...`)|
| or | json | no | Use logical OR on subset query (e.g. `{"op":"or","args":[{"op":...`)|

### Options

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| limit | int | no | The maximum number of messages to fetch |
| desc | bool | no | Sort in descending order |
| keys | bool | no | Whether the data event should contain keys. If set to true and values set to false then data events will simply be keys, rather than objects with a key property |

### Request

```json
{
  "name": ["partialReplication", "getSubset"],
  "type": "source",
  "args": [
    {
      "op": "type",
      "string": "post"
    },
    {
      "limit": 3
    }
  ]
}
```

### Examples

#### `sbotcli`

```
sbotcli subset --limit 3 '{"op":"type", "string": "post"}'
```

## `["publish"]`

### Specification

* Description: Publish a message by type (raw, post, about, contact and vote).
* Endpoint: `["publish"]`
* RPC Type: `async`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| N/A | json | yes | The message content to publish |

### Options

N/A

### Request

```json
{
  "name": ["publish"],
  "type": "async",
  "args": [{
    "type": "post",
    "text": "foo"
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli publish post "Gophers create a network of tunnel systems that provide protection."
```

## `["private", "publish"]`

### Specification

* Description: Publish a message privately.
* Endpoint: `["private", "publish"]`
* RPC Type: `async`
* Arguments: Must be an array.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| N/A | string | yes | Content of the message |
| N/A | string | yes | Recipients of the mssage |

### Options

N/A

### Request

```json
{
  "name": ["private", "publish"],
  "type": "async",
  "args": ["foo", ["@r6Lzb9OT3/dlVYNDTABmsF+HWnhBsA1twZaobYhjVUY=.ed25519"]]
}
```

### Examples

N/A

## `["tangles", "thread"]`

### Specification

* Description: Fetch all replies to the given root message. 
* Endpoint: `["tangles", "thread"]`
* RPC Type: `async`
* Arguments: Must be an array containing exactly one object.
* Options: N/A

### Arguments

| Name | Type | Required | Description |
| ---- | ---- | -------- | ----------- |
| root | string | yes | The root of the reply thread (`%...sha256`) |
| limit | int | no | Limit the number of results collected by this stream. This number represents a maximum number of results and may not be reached if you get to the end of the data first. A value of -1 means there is no limit. When `reverse=true` the highest keys will be returned instead of the lowest keys |
| seq | int | no | Start from this sequence|
| gt | int | no | "Greater than", define the lower bound of the range to be streamed |
| lt | int | no | "Less than", define the upper bound of the range to be streamed |
| reverse | bool | no | set true and the stream output will be reversed |
| live | bool | no | Keep the stream open and emit new messages as they are received |
| keys | bool | no | Whether the data event should contain keys. If set to true and values set to false then data events will simply be keys, rather than objects with a key property |
| values | bool | no | Whether the data event should contain values. If set to true and keys set to false then data events will simply be values, rather than objects with a value property |
| private | bool | no | Messages are private |
| tname | string | no | Tangle name (v2) |

### Options

N/A

### Request

```json
{
  "name": ["tangles", "thread"],
  "type": "async",
  "args": [{
    "root": "%6Jke7N/zqjxBlv3c6GgmQ96mGfpzszdhnKAnh2RnPR8=.sha256"
  }]
}
```

### Examples

#### `sbotcli`

```
sbotcli replies %6Jke7N/zqjxBlv3c6GgmQ96mGfpzszdhnKAnh2RnPR8=.sha256
```

## `["whoami"]`

### Specification

* Description: Retrieve own identity.
* Endpoint: `["whoami"]`
* RPC Type: `sync`
* Arguments: N/A
* Options: N/A

### Arguments

N/A

### Options

N/A

### Request

```json
{
  "name": ["whoami"],
  "type": "sync"
}
```

### Examples

#### `sbotcli`

```
sbotcli call whoami
```

