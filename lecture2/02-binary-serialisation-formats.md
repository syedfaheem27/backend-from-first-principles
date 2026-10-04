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

- field number is 1.
- Now, with the protobuf specification which defines the mapping, we
  figure out the wire number.
- Now, to let the server sides protobuff decoder know the wire number, we use
  a tag which is calculated as `Tag = (field_number << 3) | wire_type`.
- The first 3 bits represent the wire type and then the others represent the
  field number.
- So tag for the above data would be `1 <<< 3 | 0` => 08`(0 - varint => 25 is a 32bit)`
- Now 25 is varInt which is 19 in hex notation.
- Therefore, the transferred data would be 08 19

---

## Message Pack