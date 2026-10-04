# Binary serialisation data interchange formats

- **Why JSON cannot be always used as a data interchange format?**
- Size cost: a lot of fields are repeated, everything is represented as
  text. Even a number 1234567890 takes 10 bytes rather than 4.
- Weak types: Only string / number / boolean / null / array / object.
  No floats, int etc.
- No enforced contract.

---

- Binary formats trade human readability for some combination of smaller
  size, faster parsing, richer types, and enforced schemas.

## Protocol Buffers (Protobuff)

- You define a schema first in a `.proto` file and then the client and the
  server adhere to that schema.
- Bytes are sent directly with certain nuisances like field numbers, wire types
  where the former sets the order of the data according to some schema and the
  wire type tells the protobuff decoder on how to decode the data recieved.

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

- Certain wire types and what they mean => 0 - varint, 1 - fixed 8 bytes etc.
- Protobuff won't send id, name, active, tags, address in the data.
- The protobuff representation conceptually becomes: `[field tag][field value]`

- Let's walk through how actual data flows while being serialised and desrialised
  using protobuff.

- Suppose we want to send
```javascript
{
    age: 25
}
```

- field number = 1.
- Now, with the protobuf specification which defines the mapping, we
  figure out the wire number(0 for VarInt).
- Now, to let the server sides protobuff decoder know the wire number, we use
  a tag which is calculated as `Tag = (field_number << 3) | wire_type`.
- The first 3 bits represent the wire type and then the others represent the
  field number.
- So tag for the above data would be `1 <<< 3 | 0` => 08`(0 - varint => 25 is a 32bit)`
- Now 25 is VarInt which is 19 in hex notation.
- Therefore, the transferred data would be `08 19`

---

## Message Pack

- It's like JSON, but fast and small.
- `Schemaless and self-describing:` the same data model as
  JSON (maps, arrays, strings, numbers, booleans, nil), plus
  binary blobs and extension types.
- Every value starts with a type byte, so you can decode any
  MessagePack payload with no schema, exactly like JSON.
- Example to understand, how this works.

- Suppose we want to send
```javascript
const data = {
  age: 25
};
```

- Serialisation
```
JavaScript object
      ↓
MessagePack encoder
      ↓
binary bytes
```

- **Step 1:** Identify the object type
    - The top-level value is an object.
    - MessagePack represents JavaScript objects as a map.
    - MessagePack has a family of binary values called `fixmap`.
    - The fixmap helps get the fixed byte
    - For a map containing 1–15 entries, the encoding is:
      `1000 | number_of_entries` => 1000 | 0001 (only one entry => age) = 1001
      So, the 1st byte is 0x81
      0x81 => tells the message pack decoder, that the next thing is a map with
      a single entry.

- **Step 2:** Encode the key "age"
    - "age" is a string with 3 characters `a g e`.
    - MessagePack has a compact representation called `fixstr` for short strings.
    - Its format is: 101xxxxx where the lower bytes contain the string length
    - so for age => 10100011 => 0xA3 => next 3 bytes are string
    - Then we put the UTF-8 bytes for age:
    ```
    a = 61
    g = 67
    e = 65
    ```
    - So far the data looks like this `0x81 0xA3 61 67 65`

- **Step3:** Encode the value 25
    - MessagePack has a special compact representation for small
      positive integers called positive `fixint`.
    - Positive fixInts are 00xxxxxx for values 0-127
    - 25 decimal = 0x19

- So the complete MessagePack representation is: `81 A3 61 67 65 19`
- On the server side, the message pack decoder does the following
- Reads from left to right.
- 81: an object with a single entry
- A3: the key is a string of length 3.
- 61 67 65: Decodes it and gets age
- 19 => 25
- Thus, the server gets an object with a single entry with key as age and value as 25.


---

## Avro

- `Apache Avro` is a binary serialization system originally developed
  for Hadoop and commonly used with systems like Kafka.
- Similar to protobuff in the sense, we define a schema which must be
  known to the client and the server.


- So, the keys name and age won't be needed to sent in requests again and again.
- The difference from protobuff is that it doesn't need to send field tags.

- Suppose we have:
```javascript
{
  "name": "Faheem",
  "age": 25
}
```

- An Avro schema looks like this.
```avro
{
  "type": "record",
  "name": "User",
  "fields": [
    {
      "name": "name",
      "type": "string"
    },
    {
      "name": "age",
      "type": "int"
    }
  ]
}
```

---

## BSON
- Binary JSON. Similar to JSON but employs richer data types.
- It encodes data in binary bytes instead of plain text.
- Mostly used by mongo db as it's data serialisation format.

---

### Flat Buffers
- This was specifically designed for game development to rapidly read data without
  actually deserialising it but just reading the raw bytes.
- So, It is designed so that you can read the serialized bytes directly,
  without first deserializing the entire object into another in-memory object.
- A schema needs to be defined and compiled ahead using the `flatc` compiler.
- Data is read directly from the byte buffer and figured out.
  (Do not need to get into the depths)
