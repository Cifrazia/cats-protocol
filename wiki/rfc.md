# Common rules

> Variables and data types in this wiki displayed like `<VariableName> dataType` <br>
> Data and types are displayed as in Go programming language

- Protocol version: `3`
- Charset: `UTF-8`
- Numeric order: `Big-Endian`
- IP version: `ipv4` + `ipv6`
- Scheme: [MsgPack](https://msgpack.org)
- Time is measured in milliseconds, unless stated otherwise

## Terminology

- **Addr** - short for IP address
- **Codec** - two-way function that converts some data type to binary
- **Compressor** - two-way function that reduces binary data size
- **Cypher** - two-way function that encrypts (compressed)binary data
- **Action** - structured command, that client and server exchange
- **Action Head** - required arguments that all actions convey, list of arguments is different for each action
- **Handshake** - a process of exchanging data between client and server to determine if connection is safe-ish
- **Message** - a subclass of **Action**, commonly known as `request/response` in HTTP world
- **Input Message** - a lesser message, that server and client exchange during **Message** handling
- **Handler** - a function that handles **Message**s
- **Middleware** - decorator, function that handles **Message**s before and after **Handlers**
- **API** - a set of **Handlers**
- **Service** - a set of **API**s on a single server
- **Cluster** - a set of intertwined **Service**s, usually working behind **gateway**
- **Gateway** - a single point of entry, that navigates **Action**s to their **Endpoint**s
- **Endpoint** - path to a **Handler** within **API** within **Service** within **Cluster**

## Terms

### Error handling

- Any error occurred inside of **Handler** is converted to **Error Message** structure and used as a reply.
- Any error occurred outside of **Handler** is treated as critical and connection get closed immediately.

Example of error:

```json5
{
  "error": {
    // Use to determine broadly what went wrong
    "code": 400,
    // Use to determine exactly what went wrong
    "exception": "InvalidFieldValue",
    // Display to user
    "message": "Field value is invalid",
    "meta": {
      // May use to i18n message, bind to fields, etc.
      "field": "username",
    },
    // Traceback
    "cause": {
      "error": {
        "code": 400,
        "exception": "ValueError",
        "message": "'username' is invalid",
      }
    }
  }
}
```

### Concurrency

- CATS protocol can only send one **action** in one way at the same time.
- It can send and read simultaneously for different **actions**.
- Replies to requests are mapped using `<ActionID> uint32`.
- `<ActionID>` must be unique for active **actions**.
- Actions must be answered in N seconds, otherwise connection is treated broken.
- **Client** can increase throughput of **actions** by spawning more connections.
- Each **server** allows up to N simultaneous connections from same **Addr**.

### Multi-service _(microservice)_

Every **handler** in CATS protocol can be found by using `<EndpointID> [96]byte`: <br>
`<ServiceID> [32]byte` + `<ApiID> [32]byte` + `<HandlerID> [32]byte` <br>
`<ServiceID>`,`<ApiID>` and `<HandlerID>`, must be separately right-padded with `\x00` byte. <br>  
Multiple **service**s can lay behind single point of entry **Gateway**, that would use service discovery to find and
proxy **action**s.

By using `<EndpointID>` **client** can send request to **gateway** w/o need to open multiple connections to different
servers.

### API versioning

CATS protocol supports global **client**'s API versioning, that is used in every **service** behind **gateway**.<br>
Each separate **endpoint** can enable versioning by setting ranges of versions `[min; max)`.<br>
Protocol does not support API versioning of specific **service**s or **api**s. It can be implemented in a higher level.

### Chained Input actions

Just like in real life, before giving an answer, we can ask clarifying questions. Input actions does exactly that.<br>
You can request client/server to `input` data while still working on handling **message**.<br>
It works exactly like `input()` function, but works over the wire and support all the same features **message** does.

### Handshake

Handshake is used to verify that client is somewhat safe to talk to.<br>
Handshake is an SHA-256 binary digest (32 bytes) and is calculated by concatenating following parts:

- `<SecretKey> []byte` - secret key, that client and server agree on beforehand
- `<Time> []byte` - decimal representation of current server time, converted to string
- `<Phrase> [32]byte` - random phrase, that server used on accepting connection

> Time is a current UNIX timestamp in seconds, which last digit is replaced with 0. <br>
> Client must synchronize local time, using metadata from server.

## Variables

- `<ActionType> uint8` - int identifier for types: Message, Input, CancelInput, Config, Ping, etc.
- `<ActionID> uint32` - unique for current action queue, used to map requests and responses
    - First (left in BigEndian) bit represents action issuer: `0` for client and `1` for server
    - May be and increment counter or RNG, uniqueness is all that matters.
- `<Head> struct` - binary formatted object, unique for every action type, length must be derived from `<ActionType>`
- `<Compressors> uint32` - binary flags, representing supported `<CompressorID>` (1 | 2 | 4)
- `<Cyphers> uint32` - binary flags, representing supported `<CypherID>` (1 | 2 | 4)
- `<CompressorID> uint32` - binary flag, mapped to specific compressor (`1 << n`)
- `<CypherID> uint32` - binary flag, mapped to specific cypher (`1 << n`)

## Codecs

Data type IDs, names and descriptions of existing codecs

### Binary `0x00`

Raw binary data, no encoding applied

### Scheme `0x01`

Structural JSON-like data, packed with [MsgPack](https://msgpack.org)

### Files `0x02`

List of files, concatenated together in singular byte stream. Files metadata is stored in header `files`,
see [headers](#headers)

### Struct `0x03`

Structural data, packed as binary with built-in tools. <br>
It is faster to decode, takes less space, but requires both client and server agree on strict binary scheme.

## Headers

Headers are passed as `<length> uint32` + `<headers> [length]bytes` encoded with [MsgPack](https://msgpack.org).<br>
Every header key must be transformed into kebab-case, under the hood. External headers API must allow any case.<br>
Headers shown below are used by protocol and _should not_ be touched by client. <br>

### Files

Files header contains metadata, when (input)message payload is encoded as `files 0x02`. <br>
It is a list or FileInfo structure, sorted in order in which files are concatenated inside message payload:

```json5
{
  "files": [
    {
      // Custom file name, used to bind to retrieve from Files map:  `skin = request.files["skin"]`
      "key": "skin",
      // local filename
      "name": "steve.png",
      // approx file MIME type, may be incorrect or missing, don't rely on it
      "mime": "image/png",
      // total file size after decompression
      "size": "4020",
    },
    // ...
  ]
}
```

### Range

Range header is a set of `inclusive [start, end) exclusive` tuples, that represents which parts of RAW (uncompressed,
unencrypted) payload should be sent by server.<br>
Value of `end` must be >= `start`.<br>
Value of `start` or `end` may be `null`<br>

[//]: #  (@formatter:off)
```json5
{
  "range": [
    // bytes[:10]     - first index is 0 and last index is 9
    [null, 10],
    // bytes[200:400] - first index is 200 and last index is 399
    [200, 400],
    // bytes[512:]    - first index is 512, last index is len(bytes)-1 
    [512, null],
  ]
}
```
[//]: #  (@formatter:on)

### DataLength, ContentLength

These headers may contain:

- total size of raw data for `data-length`
- total size of encoded,compressed,cyphered data for `content-length`

These headers may be **missing**, or containing `null` as value. Do not rely on them completely. <br>
It's sole purpose is to give a bit more information to a client.

```json5
{
  "data-length": 150,
  "content-length": null,
}
```

## Compressors

Compressors are applied on each separate chunk, and compressed length of said chunk will be placed instead of original
length during `<Payload>` transmission. <br>
The following list shows supported compression algorithms:

### Dummy `0x00`

No compression applied

### Zlib `0x01`

zLib compressor with Adler32 check-sum. <br>
For each compressed chunk, it will return 3 parts of payload and their combined length:

- `<Length> uint32` - Combined length of the following chunk
- `<Chunk> [Length]byte` - Compressed chunk
    - `<Data>` - compressed data
    - `<Adler32> int64` - check sum to validate if data is not broken
    - `<Length> uint32` - size of uncompressed chunk to compare

## Cyphers

Cyphers allow to encrypt data before sending it over wire. Currently, no cypher is supported.

## Action heads

Every action consists of `<ActionType> uint8` + `<ActionID> uint32` + `<Head> struct` + `<Payload> []byte`. <br>
The following list describes which fields are placed as `<Head>` for each specific `<ActionType>`

### Message `0x00`

Total size: 8+4+8+1+1+1 = 23 bytes

- `<EndpointID> [96]byte` - combined path: `<ServiceID> [32]byte` + `<ApiID> [32]byte` + `<HandlerId> uint32`
- `<IdempotencyID> uint32` - unique temp random number that should be used to guarantee idempotency of an endpoint
- `<SendTime> int64` - time when one side started sending message, used to calculate parts from total response time
- `<CodecID> uint8` - data type ID: `0x00: binary` `0x01: scheme` `0x02: files`
- `<CompressorID> uint8` - compressor, used on data chunks.
- `<CypherID> uint8` - cypher, used to encrypt data chunks.

### Input Message `0x01`

Total size: 1+1+1 = 3 bytes

- `<CodecID> uint8` - data type ID
- `<CompressorID> uint8` - compressor, used on data chunks
- `<CypherID> uint8` - cypher, used to encrypt data chunks.

### Cancel Input `0x02`

Total size: 0 bytes

No action specific fields presented

### Ping `0xF0`

Total size: 8 bytes

- `<Time> int64` - local time in ms

### Config `0xFF`

Total size: 8 bytes

- `<TransferSpeed> uint32` - limit speed at which server sends data in bytes/s
- `<ApiVersion> uint32` - change API version that client uses

## Communication

### Initialization

1. **Client** connects to **Server** via TCP
2. **Client** sends greeting bytes, stating that it wants to use **CATS**: `CATS\x00\x00\xFF\xFF` (`[8]byte`)
    1. If **Client** exceeds limit of connections per **Addr**
        1. **Server** sends `32 empty bytes` to **Client**
        2. **Server** closes connection
    2. If **Client** is banned by internal connection rate-limiter
        1. **Server** closes connection
3. **Server** sends statement to **Client**:
    1. `<ProtocolVersion> uint8`
    2. `<ServerTime> int64`
    3. `<ServiceID> [32]byte`
    4. `<Compressors> uint32` - compressors, supported by server
    5. `<Cyphers> uint32` - cyphers, supported by server
    6. `<IdleTimeout> int64` - amount of milliseconds connection can hang idle before force closed
    7. `<InputTimeout> int64` - amount of milliseconds server will wait for a response to `Input` action
    8. `<HandshakeQuestion> [32]byte` - open random phrase, that client should use alongside local key
4. **Client** decides whether `<ProtocolVersion>` and `<ServiceID>` matches expectations and abort connection if not.
5. **Client** sends statement to **Server**:
    1. `<ProtocolVersion> uint8` - Must be supplied by client even if client knows it matches
    2. `<ClientTime> int64`
    3. `<Compressors> uint32`
    4. `<Cyphers> uint32`
    5. `<ApiVersion> uint32` - API version that client expects to use
    6. `<HandshakeAnswer> [32]byte` - [SHA256 digest](#handshake), calculated from current time, question phrase and
       local key
6. **Server** validates client statement and handshake and return confirmation
    1. If `<ProtocolVersion>` match - send `0x00`, otherwise send `<ProtocolVersion>`
    2. If `<HandshakeAnswer>` is valid - send `0x00`, otherwise send `0x01`
    3. If any error occurred, terminate connection
7. **Server** and **Client** can now exchange actions

#### Binary example

==- Client to Server: Greeting

| Value                  | Binary                    |
|------------------------|---------------------------|
| `CATS\x00\x00\xFF\xFF` | `43 41 54 53 00 00 FF FF` |

==- Server to Client: Statement

| Value                                               | Binary                                                                                            |
|-----------------------------------------------------|---------------------------------------------------------------------------------------------------|
| ProtocolVersion: `3`                                | `03`                                                                                              |
| ServerTime: `1257894000000`                         | `00 00 01 24 E0 53 35 80`                                                                         |
| ServiceID: `minecraft`                              | `6D 69 6E 65 63 72 61 66 74 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |
| Compressors: `7` (1 + 2 + 4)                        | `00 00 00 07`                                                                                     |
| Cyphers: `0`                                        | `00 00 00 00`                                                                                     |
| IdleTimeout: `120000`                               | `00 00 00 00 00 01 D4 C0`                                                                         |
| InputTimeout: `120000`                              | `00 00 00 00 00 01 D4 C0`                                                                         |
| HandshakePhrase: `somerandomphrasesomerandomphrase` | `73 6F 6D 65 72 61 6E 64 6F 6D 70 68 72 61 73 65 73 6F 6D 65 72 61 6E 64 6F 6D 70 68 72 61 73 65` |

==- Client to Server: Statement

| Value                        | Binary                                                                                            |
|------------------------------|---------------------------------------------------------------------------------------------------|
| ProtocolVersion: `3`         | `03`                                                                                              |
| ClientTime: `1257894000000`  | `00 00 01 24 E0 53 35 80`                                                                         |
| Compressors: `7` (1 + 2 + 4) | `00 00 00 07`                                                                                     |
| Cyphers: `0`                 | `00 00 00 00`                                                                                     |
| ApiVersion: `13`             | `00 00 00 0D`                                                                                     |
| HandshakeAnswer              | `48 65 6C 6C 6F 20 57 6F 72 6C 64 20 4C 4F 4C 20 61 68 61 68 61 68 20 68 61 68 20 61 68 68 61 68` |

==- Server to Client: Validation

| Value               | Binary |
|---------------------|--------|
| ProtocolVersion: ok | `00`   |
| HandshakeAnswer: ok | `00`   |

===

### Action exchange

In this example action issuer will be client, however server can also send actions one its own

1. **Client** sends **action** to **Server**:
    1. `<ActionType> uint8`
    2. `<ActionID> uint32`
    3. `<Head> struct`
    4. `<Payload> []byte` - consists of `N` pairs of:`<length> uint32` + `<chunk> [length]byte`
        1. Data can be split into chunks of bytes and sent in separate waves
        2. Payload must end with `<length> = 0`
2. **Server** determines **action** type
    1. If `Ping{time: int64}`
        - read client time in ms
        - reset conn timeout
        - send `Ping{time: int64}` with server time in ms
    2. If `Config{transferSpeed uint32, apiVersion uint32}`:
        - update connection configuration:
        - send updated config `Config{transferSpeed uint32, apiVersion uint32}`
    3. If `Message`
        - pass action to worker
        - handle message wrapped in middleware
        - check and wait for all related inputs to be resolved
        - prepare response
        - send message to client
    4. If `Input`
        - pass action to worker
        - check requested inputs by any message in queue
        - if input is awaited, set result and remove from map
        - if input is not awaited, ignore
    5. If `InputCancel`
        - pass action to worker
        - check requested inputs by any message in queue
        - if input is awaited, cancel it
        - if input is not awaited, ignore
3. **Server** sends response **action** to **Client**
