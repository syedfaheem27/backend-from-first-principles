# Serialisation and Deserialisation

- The client and the server should agree on a common data interchange
  formate before transmitting data over the network because the client
  and server may be written in different languages and hence may not
  understand each other if they didn't agree on a common data
  interchange format.
- This is where different data interchange formats come in.
- Text based data interchange formats:
    - JSON
    - YAML
    - XML
- Binary based data interchange formats:
    - Protobuff
    - Message Pack
    - Avro
    - Flat Buffers
    - BSON
- Now imagine a client side application written in javascript wants to send some
  data to the server in the request body to the server. The data is first serialised
  into the agreed data interchange formats between the client and the server and then
  transmitted. The serialisation happens at the application layer. On the server side
  the data is deserialised back and converted into the data structure that the server
  understands.
- `Serialization` = converting data from a language's in-memory representation
  into a common/standard format.

- `Deserialization` = converting data from that common format back into a
  language's own representation.
- **Why JSON is heavily popular?**
    - Although, there is a parsing overhead to convert everything into strings but
      since it is human readable and thus helps with debugging what actually was sent
      and what actually was recieved.
