# Binary Serialization Formats: Protobuf, MessagePack, Avro, BSON, FlatBuffers

A companion to [serialization-and-deserialization.md](serialization-and-deserialization.md). That chapter covered the **text-based** side (JSON). These notes cover the main **binary** formats: what each one is, how it encodes bytes on the wire, how it handles schema changes, and when to use it.

---

## Table of contents

0. [Background: why binary, and the axes that matter](#0-background)
1. [Protocol Buffers (Protobuf)](#1-protocol-buffers-protobuf)
2. [MessagePack](#2-messagepack)
3. [Apache Avro](#3-apache-avro)
4. [BSON](#4-bson)
5. [FlatBuffers](#5-flatbuffers)
6. [Side-by-side: the same object in every format](#6-side-by-side)
7. [Comparison table](#7-comparison-table)
8. [How to choose](#8-how-to-choose)

---

## 0. Background

### Why not just use JSON everywhere?

JSON is human-readable, universal, and good enough for most REST APIs. But it has costs:

| JSON cost | Why it happens |
|---|---|
| **Size** | Every message repeats every key (`"name"`, `"name"`, `"name"`…). Numbers are stored as text: `1234567890` takes 10 bytes rather than 4. |
| **Parse speed** | The parser has to scan characters, handle escapes, and convert text to numbers. |
| **Weak types** | Only string / number / boolean / null / array / object. There are no integers vs floats, dates, or raw bytes; binary data has to be base64-encoded, which adds ~33%. |
| **No enforced contract** | Nothing stops a client from sending `"age": "thirty"`. |

Binary formats trade **human readability** for some combination of **smaller size, faster parsing, richer types, and enforced schemas**.

### The axes to compare formats on

Keep these questions in mind for every format below:

1. **Schema or schemaless?** Do both sides need a schema definition up front, or does each message describe itself?
2. **Self-describing?** Can you decode the bytes with no outside information?
3. **How are fields identified?** By name, by number (tag), or by position?
4. **Schema evolution:** what happens when you add, remove, or rename fields while old and new code are both running?
5. **Parse cost:** does it need a full decode step, or can you read fields in place ("zero-copy")?
6. **Ecosystem:** where is it actually used?

### Two vocabulary terms

- **Backward compatible:** *new* code can read data written by *old* code.
- **Forward compatible:** *old* code can read data written by *new* code.

In a real backend with rolling deploys, you need both. Some servers run v1 and some run v2 at the same time, and messages in queues or on disk outlive the code that wrote them.

### Running example

To compare sizes, every section encodes the same object:

```json
{"id":150,"name":"Ali"}
```

As compact JSON, that is **23 bytes**.

---

## 1. Protocol Buffers (Protobuf)

### What it is

- Created at **Google** for internal RPC and storage, and open-sourced in **2008**.
- **Schema-first**: you define messages in a `.proto` file and run the **`protoc`** compiler to generate classes for your language (Go, Java, Python, C++, Rust, TypeScript, etc.).
- The default wire format of **gRPC**.
- **Not self-describing**: the bytes contain field *numbers*, not names or types. You need the `.proto` schema to interpret them meaningfully.

### Schema

```proto
syntax = "proto3";

message User {
  int32  id     = 1;   // ← the number is the field's identity on the wire
  string name   = 2;
  bool   active = 3;
  repeated string tags = 4;      // list
  Address address = 5;           // nested message
}

message Address {
  string country = 1;
}
```

The `= 1`, `= 2`, … are **field numbers**, not default values. They are the most important part of the schema: **the wire format only stores the number, never the name.** You can rename `name` to `full_name` freely, but you can never change `2` to something else.

### Wire format: how bytes are laid out

A message is just a sequence of **(tag, value)** pairs. There is no header, no length prefix for the whole message, and no terminator.

**Tag** = `(field_number << 3) | wire_type`, encoded as a varint.

| Wire type | Meaning | Used for |
|---|---|---|
| 0 | VARINT | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1 | I64 (fixed 8 bytes) | fixed64, sfixed64, double |
| 2 | LEN (length-delimited) | string, bytes, nested messages, packed repeated fields |
| 5 | I32 (fixed 4 bytes) | fixed32, sfixed32, float |

(Types 3 and 4 were "groups", which are deprecated.)

**Varint** is variable-length integer encoding. Each byte carries 7 bits of the number, and the high bit (MSB) is 1 if more bytes follow. Small numbers take fewer bytes:

```
150 in binary        = 1001 0110
split into 7-bit groups (least significant first): 0010110, 0000001
add continuation bit:  1|0010110 = 0x96,  0|0000001 = 0x01
→ 96 01
```

### Encoding the running example

`User { id: 150, name: "Ali" }`

```
08          tag: field 1, wire type 0 (varint)   → (1<<3)|0 = 0x08
96 01       value: 150 as varint
12          tag: field 2, wire type 2 (LEN)      → (2<<3)|2 = 0x12
03          length: 3
41 6C 69    "Ali" in UTF-8
```

**8 bytes** compared with 23 for JSON.

### Details worth knowing

- **Default values aren't sent.** In proto3, `0`, `""`, `false`, and empty lists are not written to the wire at all. This saves space, but it means the receiver can't distinguish "set to 0" from "not set". If you need that distinction, mark the field `optional` (supported again since protobuf 3.15) or use a wrapper type.
- **Field numbers 1–15** fit in a one-byte tag. Use them for the most frequent fields.
- **Negative numbers:** `int32 -1` is encoded as a 10-byte varint (it's sign-extended to 64 bits). If a field is often negative, use **`sint32`/`sint64`**, which use **ZigZag encoding** (0→0, -1→1, 1→2, -2→3, …) so small negatives stay small.
- **Unknown fields are skipped and preserved.** If old code receives a field number it doesn't know, it can still skip over it because the wire type tells it how long the value is. Modern runtimes also keep those bytes and re-emit them on re-serialization. This is what makes forward compatibility work.
- **Last one wins.** If the same non-repeated field appears twice, the parser keeps the last value. This means two serialized messages can be merged by concatenating them.

### Schema evolution rules

| Change | Safe? |
|---|---|
| Add a new field with a new number | ✅ Old readers skip it; new readers see the default if it's absent |
| Remove a field | ✅ **but** mark its number (and name) `reserved` so nobody reuses it |
| Rename a field | ✅ on the binary wire, because names aren't sent (⚠️ breaks the JSON mapping) |
| Change a field's number | ❌ Same as deleting it and adding a new field |
| Reuse an old field number for a new meaning | ❌ Old data gets silently misread |
| Change type | ⚠️ Only between compatible types (e.g. `int32` ↔ `int64` ↔ `bool`); otherwise ❌ |

```proto
message User {
  reserved 3;            // was "active", removed in v2
  reserved "active";
  int32  id   = 1;
  string name = 2;
}
```

### Usage (JavaScript, `protobufjs`)

```js
import protobuf from "protobufjs";

const root = await protobuf.load("user.proto");
const User = root.lookupType("User");

const bytes = User.encode(User.create({ id: 150, name: "Ali" })).finish(); // serialize
const user  = User.decode(bytes);                                          // deserialize
```

### Strengths and weaknesses

| ✅ Strengths | ❌ Weaknesses |
|---|---|
| Very compact; fast encode and decode | Not human-readable, so you need tooling to debug |
| A strong, explicit contract between services | Requires a build step (`protoc`) and shared `.proto` files |
| Mature schema evolution story | Must fully decode to read any field (no zero-copy) |
| Huge ecosystem: gRPC, every major language | The proto3 default-value / presence semantics trip people up |

**Typical uses:** gRPC microservice-to-microservice calls, internal APIs at scale, mobile ↔ backend where bandwidth matters, and stored events.

---

## 2. MessagePack

### What it is

- Its own tagline: *"It's like JSON, but fast and small."*
- **Schemaless and self-describing**: the same data model as JSON (maps, arrays, strings, numbers, booleans, nil), plus **binary blobs** and **extension types**.
- Every value starts with a **type byte**, so you can decode any MessagePack payload with no schema, exactly like JSON.
- It's a drop-in replacement for JSON: `JSON.stringify` / `JSON.parse` become `encode` / `decode`.

### Wire format

The key idea is the **type prefix byte**. For small values the type *and* the value (or length) are packed into that single byte ("fix" formats):

| First byte | Format | Meaning |
|---|---|---|
| `0x00`–`0x7F` | positive fixint | The integer 0–127 itself, in 1 byte |
| `0x80`–`0x8F` | fixmap | Map with 0–15 entries (low 4 bits = count) |
| `0x90`–`0x9F` | fixarray | Array with 0–15 items |
| `0xA0`–`0xBF` | fixstr | String with 0–31 bytes (low 5 bits = length) |
| `0xC0` | nil | `null` |
| `0xC2` / `0xC3` | false / true | |
| `0xC4`–`0xC6` | bin 8/16/32 | Raw bytes (no base64 needed) |
| `0xCA` / `0xCB` | float 32 / float 64 | |
| `0xCC`–`0xCF` | uint 8/16/32/64 | |
| `0xD0`–`0xD3` | int 8/16/32/64 | |
| `0xD9`–`0xDB` | str 8/16/32 | Longer strings |
| `0xDC` / `0xDD` | array 16/32 | Longer arrays |
| `0xDE` / `0xDF` | map 16/32 | Larger maps |
| `0xE0`–`0xFF` | negative fixint | -32 to -1 in 1 byte |
| `0xC7`–`0xC9`, `0xD4`–`0xD8` | ext | Application-defined types (type -1 is the built-in **Timestamp**) |

### Encoding the running example

```
82                  fixmap, 2 entries
A2 69 64            fixstr len 2: "id"
CC 96               uint8: 150      (150 > 127, so it can't be a fixint)
A4 6E 61 6D 65      fixstr len 4: "name"
A3 41 6C 69         fixstr len 3: "Ali"
```

**15 bytes** compared with 23 for JSON. Notice that **the keys are still in the payload**. That's the price of being schemaless, and it's why MessagePack is smaller than JSON but larger than Protobuf or Avro.

### Usage (JavaScript, `@msgpack/msgpack`)

```js
import { encode, decode } from "@msgpack/msgpack";

const bytes = encode({ id: 150, name: "Ali" }); // Uint8Array, 15 bytes
const obj   = decode(bytes);                    // { id: 150, name: "Ali" }
```

### Schema evolution

There is no schema, so there are no rules: you get **the same situation as JSON**. Adding fields is fine if readers ignore unknown keys. Renaming or removing fields breaks consumers unless your application code handles it. Validation is your job (e.g. Zod or JSON-Schema-style checks after decoding).

### Strengths and weaknesses

| ✅ Strengths | ❌ Weaknesses |
|---|---|
| Drop-in JSON replacement with no schema and no build step | Still repeats keys in every message |
| Smaller and usually faster than JSON | No contract or validation |
| Native binary data, distinct int/float types | Not human-readable |
| Libraries for ~every language | Gains over *compressed* JSON are often modest |

**Typical uses:** caching objects in Redis or Memcached, internal RPC and WebSocket payloads (e.g. ASP.NET SignalR's MessagePack protocol), log shippers (Fluentd's internal format), and anywhere you want "JSON but smaller" without adopting schemas.

---

## 3. Apache Avro

### What it is

- An **Apache** project created by **Doug Cutting** (the creator of Hadoop) in **2009** for the Hadoop ecosystem.
- **Schema-based**, with schemas written in **JSON** (there's also an IDL).
- The defining feature: **the binary data contains no field names and no field tags.** Values are written back to back **in schema order**. That makes it the most compact of the formats here.
- Because the data is meaningless without the schema, **the writer's schema must always be available to the reader**: embedded in the file, or looked up from a **Schema Registry**.
- Code generation is **optional**. Avro works fine with generic records built at runtime from a schema, which suits dynamic languages and data pipelines.

### Schema

```json
{
  "type": "record",
  "name": "User",
  "namespace": "com.example",
  "fields": [
    { "name": "id",    "type": "int" },
    { "name": "name",  "type": "string" },
    { "name": "email", "type": ["null", "string"], "default": null }
  ]
}
```

- Primitive types: `null`, `boolean`, `int`, `long`, `float`, `double`, `bytes`, `string`.
- Complex types: `record`, `enum`, `array`, `map`, `union`, `fixed`.
- **Nullable fields are unions**: `["null", "string"]`.
- **Logical types** sit on top of primitives: `date`, `timestamp-millis`, `decimal`, `uuid`.

### Wire format

| Type | Encoding |
|---|---|
| `int`, `long` | ZigZag + varint (like Protobuf's `sint`) |
| `string`, `bytes` | Length (as a zigzag varint `long`) followed by the raw bytes |
| `boolean` | 1 byte |
| `float` / `double` | 4 / 8 bytes, little-endian |
| `record` | Each field's value in schema order, with **nothing in between** |
| `union` | Branch index (varint), then the value |
| `array` / `map` | Written in blocks: count, items, …, terminated by a 0 count |

### Encoding the running example

Using the schema `{id: int, name: string}`:

```
AC 02         id = 150  → zigzag(150) = 300 → varint = AC 02
06            name length = 3 → zigzag(3) = 6 → 06
41 6C 69      "Ali"
```

**6 bytes** compared with 23 for JSON and 8 for Protobuf. There are no tags at all. The reader knows "first comes an int, then a string" only because it has the schema.

### Schema resolution: Avro's superpower

When reading, Avro uses **two schemas**:

- the **writer's schema** (what the data was actually written with), and
- the **reader's schema** (what the consuming code expects).

The library walks both and **resolves differences by field name**:

| Situation | Result |
|---|---|
| Field in writer, not in reader | Ignored (skipped) |
| Field in reader, not in writer | Filled with the reader's **`default`** (an error if there's no default) |
| Field in both, types differ | Allowed if promotable (`int → long → float → double`, `string ↔ bytes`); otherwise an error |
| Field order differs | Fine, because matching is by name |
| Rename | Use **`aliases`** in the reader schema |

**Rule of thumb:** to keep both backward and forward compatibility, **only add or remove fields that have defaults.**

### How the writer's schema travels

1. **Object Container Files (`.avro`)**: the file header contains the full schema once. Then come blocks of records separated by sync markers, optionally compressed (deflate, snappy, zstd…). The files are splittable, which is ideal for Hadoop/Spark-style processing of large datasets.
2. **Kafka + Schema Registry** (Confluent's wire format): each message is
   ```
   [0x00 magic byte][4-byte schema ID][Avro binary payload]
   ```
   The consumer fetches the schema by ID from the registry and caches it. The registry can also **enforce compatibility rules** (BACKWARD, FORWARD, FULL) so that an incompatible schema change is rejected before it's deployed.
3. **RPC**: the client and server exchange schemas during the handshake.

### Usage (JavaScript, `avsc`)

```js
import avro from "avsc";

const type = avro.Type.forSchema({
  type: "record", name: "User",
  fields: [{ name: "id", type: "int" }, { name: "name", type: "string" }],
});

const bytes = type.toBuffer({ id: 150, name: "Ali" }); // 6 bytes
const user  = type.fromBuffer(bytes);
```

### Strengths and weaknesses

| ✅ Strengths | ❌ Weaknesses |
|---|---|
| The most compact format here (no names, no tags) | Useless without the writer's schema, so you need infrastructure (registry, file headers) |
| Excellent, well-defined schema evolution | A single message isn't self-contained |
| Schemas are plain JSON, and codegen is optional | Less common for request/response web APIs |
| First-class in Kafka, Hadoop, Spark, Hive | Debugging raw bytes is nearly impossible |

**Typical uses:** **Kafka** event streams, data lakes and big-data pipelines, change-data-capture (e.g. Debezium), and long-term storage of records whose schema changes over the years.

---

## 4. BSON

### What it is

- **"Binary JSON"**, created by **MongoDB** as its storage format and wire format.
- **Schemaless and self-describing**: keys are stored as strings in every document, like JSON and MessagePack.
- The design goal is **not** minimal size. It's **fast traversal and richer types** for a database:
  - Every document and string is **length-prefixed**, so a reader can **skip** whole sub-documents without parsing them, which is useful when a query only needs one field.
  - Numbers are fixed-width, so the database can **update a value in place** without rewriting the document.
  - It adds types that JSON lacks: **dates, ObjectId, 32/64-bit ints, Decimal128, binary, regex**, and more.

### Wire format

```
document ::= int32 total_length   (little-endian, includes itself and the trailing 0x00)
             element*
             0x00

element  ::= type_byte  key_as_cstring(…\0)  value
```

| Type byte | Type | Value encoding |
|---|---|---|
| `0x01` | double | 8 bytes |
| `0x02` | string | int32 length (including `\0`) + UTF-8 + `\0` |
| `0x03` | embedded document | A nested document |
| `0x04` | array | A document whose keys are `"0"`, `"1"`, `"2"`… |
| `0x05` | binary | int32 length + subtype byte + bytes |
| `0x07` | ObjectId | 12 bytes |
| `0x08` | boolean | 1 byte |
| `0x09` | UTC datetime | int64 milliseconds since the Unix epoch |
| `0x0A` | null | No value bytes |
| `0x10` | int32 | 4 bytes |
| `0x11` | timestamp | 8 bytes (MongoDB-internal, for replication) |
| `0x12` | int64 | 8 bytes |
| `0x13` | Decimal128 | 16 bytes |

### Encoding the running example

```
1B 00 00 00                         total length = 27
10 69 64 00 96 00 00 00             0x10 int32, key "id\0", value 150 (4 bytes LE)
02 6E 61 6D 65 00 04 00 00 00       0x02 string, key "name\0", length 4 (incl. \0)
   41 6C 69 00                      "Ali\0"
00                                  end of document
```

**27 bytes**, which is **larger than the 23 bytes of JSON**. This surprises many people: BSON isn't about size. The length prefixes, null terminators, fixed-width ints, and array index keys (`"0"`, `"1"`…) all add overhead. What you get in return is speed of traversal and real types.

### ObjectId

MongoDB's default `_id` is a 12-byte ObjectId:

```
[ 4-byte timestamp (seconds) ][ 5-byte random value ][ 3-byte incrementing counter ]
```

Clients can generate ObjectIds without coordinating with each other, and the IDs sort roughly by creation time.

### Usage (JavaScript, `bson`)

```js
import { BSON, ObjectId } from "bson";

const bytes = BSON.serialize({ _id: new ObjectId(), createdAt: new Date(), id: 150 });
const doc   = BSON.deserialize(bytes); // createdAt comes back as a real Date
```

### Things to know

- MongoDB caps a single document at **16 MB**.
- When the MongoDB shell or REST tools show you JSON, it's actually **Extended JSON**, which encodes BSON types as e.g. `{"$oid": "..."}` and `{"$date": "..."}` so they survive the round trip.
- Schema evolution works the same as JSON (schemaless). MongoDB can optionally enforce `$jsonSchema` validation at the collection level.

### Strengths and weaknesses

| ✅ Strengths | ❌ Weaknesses |
|---|---|
| Rich types (dates, int64, decimal, binary, ObjectId) | Often **bigger** than JSON |
| Fast to traverse and skip, with in-place updates | Basically tied to the MongoDB ecosystem |
| Self-describing, so no schema is needed | Repeats keys in every document |

**Typical uses:** MongoDB storage and the MongoDB driver ↔ server protocol. Outside MongoDB, it's rarely the best choice: pick MessagePack for "binary JSON on the wire".

---

## 5. FlatBuffers

### What it is

- Created at **Google** by Wouter van Oortmerssen in **2014**, originally for **game development** and other performance-critical apps.
- **Schema-based** (`.fbs` files), with the **`flatc`** compiler generating code.
- The defining feature: **zero-copy deserialization**. The serialized buffer **is** the in-memory representation. There's no parse or unpack step: generated accessor methods read fields **directly from the byte buffer** using offsets.

### Why zero-copy matters

With Protobuf, JSON, or MessagePack:

```
bytes ──parse──▶ allocate objects ──▶ copy every field ──▶ use 1 field
```

With FlatBuffers:

```
bytes ──▶ read 1 field directly via offset (nothing else is touched)
```

If you receive a 50 MB buffer and only need 3 fields, FlatBuffers reads 3 fields. The others are never parsed and never allocated. You can even **`mmap`** a file and access it immediately. Startup cost is close to zero and there's no garbage-collector pressure.

### Schema

```fbs
namespace Example;

table User {
  id: int;
  name: string;
  active: bool = true;
  tags: [string];
}

struct Vec3 {           // fixed layout, stored inline
  x: float;
  y: float;
  z: float;
}

root_type User;
```

**Tables vs structs:**

| | `table` | `struct` |
|---|---|---|
| Fields optional? | Yes (missing → default) | No, all fields always present |
| Evolvable? | Yes, you can add fields | **No**, the layout is frozen forever |
| Storage | Through a **vtable** of offsets | Inline, fixed size, no overhead |
| Use for | Most data | Small, stable, performance-critical values (vectors, colors) |

### Wire format (conceptually)

- The buffer starts with an offset to the **root table**.
- Each table points to a **vtable**: a small array saying "field 0 is at +4, field 1 is at +8, field 2 is absent…".
- To read `user.name()`, the accessor looks up the vtable entry for `name`, jumps to that offset, and reads the string (length-prefixed, null-terminated) **in place**.
- If a field is absent, the vtable entry is 0 and the accessor returns the **default**. Defaults usually aren't stored at all.
- Identical vtables are **shared** between tables to save space.
- Everything is **little-endian and aligned**, so on most CPUs values can be read with plain memory loads.
- Offsets are 32-bit, so a single buffer is limited to about **2 GB**.

### Building is back to front

Writing is the awkward part. A `Builder` constructs the buffer **from the end toward the start**, and **children must be created before parents** (strings and sub-tables first, then the table that references them):

```ts
import * as flatbuffers from "flatbuffers";
import { User } from "./generated/example/user";

// serialize
const builder = new flatbuffers.Builder(1024);
const name = builder.createString("Ali");   // child first
User.startUser(builder);
User.addId(builder, 150);
User.addName(builder, name);
const user = User.endUser(builder);
builder.finish(user);
const bytes = builder.asUint8Array();

// "deserialize": just wrap the bytes, nothing is parsed
const u = User.getRootAsUser(new flatbuffers.ByteBuffer(bytes));
u.id();    // 150, read straight from the buffer
u.name();  // "Ali"
```

### Size

For tiny messages like the running example, FlatBuffers is **larger than Protobuf**: vtables, offsets, and alignment padding add fixed overhead. It wins on **access speed**, not on bytes. For large messages with many numeric fields, the overhead becomes proportionally small.

### Schema evolution rules

- Add new fields **only at the end** of a table, or give fields explicit `(id: N)` attributes.
- **Never remove** a field: mark it `(deprecated)` so the slot stays reserved.
- Don't change a field's type or the default value of an existing field.
- `struct`s can never change.

### Things to know

- **Mutation is limited.** You can overwrite scalar fields that are present in the buffer, but you can't add fields or resize strings in place. Otherwise, rebuild the buffer.
- **Untrusted input:** because reads follow offsets, a malicious buffer could point out of bounds. Run the generated **Verifier** on data from untrusted sources before accessing it.
- **Cap'n Proto** (by Kenton Varda, the author of Protobuf v2) is a close cousin with the same zero-copy idea and a built-in RPC system.

### Strengths and weaknesses

| ✅ Strengths | ❌ Weaknesses |
|---|---|
| Zero-copy: reads are nearly free, with no allocations | Clunky builder API (back to front, children first) |
| Random access to any field without parsing the whole buffer | Larger than Protobuf/Avro for small messages |
| Works with `mmap`, so huge files load instantly | Immutable in practice |
| Schema evolution for tables | Smaller ecosystem; needs a verifier for untrusted data |

**Typical uses:** games (level data, network state), **TensorFlow Lite** model files (`.tflite` is a FlatBuffer), mobile apps where startup time and memory matter (Facebook used it to speed up its Android app's data loading), and any read-heavy hot path where parse time dominates.

---

## 6. Side-by-side

The same object, `{"id":150,"name":"Ali"}`, in each format:

| Format | Bytes | Hex | Keys in payload? |
|---|---|---|---|
| JSON | 23 | `7B 22 69 64 22 3A 31 35 30 2C …` | Yes (as text) |
| BSON | 27 | `1B 00 00 00 10 69 64 00 96 00 00 00 02 6E 61 6D 65 00 04 00 00 00 41 6C 69 00 00` | Yes |
| MessagePack | 15 | `82 A2 69 64 CC 96 A4 6E 61 6D 65 A3 41 6C 69` | Yes |
| Protobuf | 8 | `08 96 01 12 03 41 6C 69` | No, field **numbers** |
| Avro | 6 | `AC 02 06 41 6C 69` | No, **position** only |
| FlatBuffers | larger than Protobuf (vtable + offsets + padding) | — | No, **vtable offsets** |

The pattern: **the more the reader is expected to already know (the schema), the less the bytes have to say.**

```
self-describing ◀──────────────────────────────────────────▶ schema-dependent
JSON · BSON · MessagePack          Protobuf          Avro       FlatBuffers*
(names in data)               (numbers in data)  (nothing)   (offsets, zero-copy)
```

\*FlatBuffers is placed by its access model rather than its size.

---

## 7. Comparison table

| | **Protobuf** | **MessagePack** | **Avro** | **BSON** | **FlatBuffers** |
|---|---|---|---|---|---|
| Origin | Google, 2008 | Sadayuki Furuhashi, 2008 | Apache / Doug Cutting, 2009 | MongoDB, 2009 | Google, 2014 |
| Schema | Required (`.proto`) | None | Required (JSON) | None | Required (`.fbs`) |
| Self-describing | No | Yes | No (needs the writer's schema) | Yes | No |
| Field identity on the wire | Field number | Key string | Position | Key string | vtable slot |
| Code generation | Yes (`protoc`) | No | Optional | No | Yes (`flatc`) |
| Size vs JSON | Much smaller | Smaller | Smallest | Often **larger** | Smaller to similar |
| Decode cost | Full parse (fast) | Full parse (fast) | Full parse (fast) | Full parse, skippable | **Zero-copy** |
| Schema evolution | Good (tags + `reserved`) | Manual (like JSON) | Excellent (reader/writer resolution) | Manual (like JSON) | Good (append-only tables) |
| Rich types (dates, int64, bytes) | Yes (well-known types) | Bytes + ext types | Yes (logical types) | Yes, very rich | Yes |
| Flagship use | gRPC | Redis/caching, RPC | Kafka, data lakes | MongoDB | Games, TF Lite |

---

## 8. How to choose

| If you need… | Choose |
|---|---|
| Public REST API, browser clients, easy debugging | **JSON** (plus gzip/brotli). Don't over-engineer. |
| Service-to-service RPC with strict contracts | **Protobuf** (with gRPC) |
| "JSON but smaller/faster" without adopting schemas | **MessagePack** |
| Event streaming or data pipelines with schemas that change over years | **Avro** + a Schema Registry |
| You're using MongoDB | **BSON** (handled by the driver; you rarely touch it directly) |
| Read-heavy hot paths, large buffers, mmap, games, ML models | **FlatBuffers** (or Cap'n Proto) |

### Practical advice

1. **Start with JSON.** Switch only when you've *measured* a problem (payload size, CPU time spent parsing, or the need for a strict contract).
2. **Compression changes the math.** Gzipped JSON is often within a small factor of raw binary formats, so measure with compression on.
3. **Schemas pay off in teams.** The biggest real-world win of Protobuf and Avro is often not speed but the **enforced, versioned contract** between teams and services.
4. **Plan for evolution on day one.** Never reuse Protobuf field numbers, always give Avro fields defaults, and only append to FlatBuffers tables.
5. **Validate untrusted input** regardless of format: schemaless formats need validation after decoding, and FlatBuffers needs its verifier.
