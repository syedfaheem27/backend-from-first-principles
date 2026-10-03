## HTTP Caching
- Implemented using `Cache-Control`, `ETag`, `Last Modified`
- The client sends these headers
  - `If-None-Match` - Carries the ETag the client already holds
  - `If-Modified-Since` - carries the last modified time stamp the client already holds.

## Content negotiation and compression

- `Content negotiation:` A mechanism by which the client and the server agree on the best format for exchanging data. The client states it preferrence and the server tries to honour them, falling back to something else if it can't.

| Type | Client header | Example values |
|---|---|---|
| **Media type** | `Accept` | `application/json`, `application/xml`, `text/html` |
| **Language** | `Accept-Language` | `en`, `es` |
| **Encoding** | `Accept-Encoding` | `gzip`, `deflate`, `br`, `zstd` |

### HTTP compression
- Driven by `Accept-encoding(client)` and `Conten-Encoding(server)`
- The client accepts gzip and the server serves a gzip compressed file which the client decodes. Browsers do this automatically.


### Persistent connections and keep-alive
- A single tcp connection is used for multiple requests.
- Thus, for n number of requests, the 3 way handshake is done once.
- Uses headers - `Connection: keep-alive` and `Connection: close`
- The transport layer ensures reliable transfer and then the application layer parses the data to pass the whole request to the server rather than a part of the request because the transport layer transfers data in packets.
- For large data the content is streamed into chunks which then the application layer parses and forwards to the cient or the server.

## SSL/TLS and HTTPS

- `SSL` was initially used to encrypt data transmitted over the network but due to security vulnerabilities, it is not used today.
- `TLS` It is used nowadays to encrypt data being transmitted over the network.
- `HTTPS` => HTTP + TLS
- `Digital certificates:` These are certificates issued to the server by CA authorities and serves as a claim for the server to actually serve the data for some domain. Otherwise any server can impersonate a domain and claim it is the server for the said domain.
- `TLS gives you a secure encrypted channel. The certificate tells you who you're securely talking to`