# Headers(Both request and response)

The obvious question: **Why not stuff everything into the URL or the body? Why invent a separate section?**

`Analogy:` When you ship a parcel, the recipient's name, address, phone number, and PIN code go **on the outside**, not inside the box. Everyone handling the package along the route — sorting facility, delivery driver, customs — needs to read that metadata repeatedly. If it were inside, every handler would have to open the parcel.

Headers work the same way. `Proxies, load balancers, caches, and the server itself can all read routing and handling metadata without parsing (or even decompressing) the body.`

### Categories of headers

#### Request headers
Sent by the client, describing the request and the client itself.

- `User-Agent` — identifies the client: browser, Postman, mobile app, another server.
- `Authorization` — credentials, e.g. a bearer token, identifying the user.
- `Accept` — what content types the client wants back (JSON, text, HTML).

Together, these let the server understand the client's environment, preferences, and capabilities.

#### General headers
Used in both requests and responses, carrying metadata about the message itself.

- `Date` — when the message was generated.
- `Cache-Control` — caching directives like `no-cache`, `max-age`.
- `Connection` — `keep-alive` or `close`.

Lets look at some of the headers above.
- `Cache-Control`: Dictates who will store the response and for how long.
- `Connection`: It dictates whether the underlying `TCP connection` should stay alive
  or be teared down. This header is not passed on after hitting the first proxy.

```
Connection: keep-alive
Keep-Alive: timeout=5, max=1000
```
`The TCP connection will be alive if it stays idle for 5 seconds and this connection will handle a max of 1000 connections before being closed.`


**Will just adding a header make all its corresponding functions work?**

- Some headers are pure declaration and need real logic behind them;
  some trigger behaviour that's already built into the browser or the server;
  some are handled entirely below your code.

- `Cache-Control` — header alone is enough. This is the one case where sending the header genuinely
  does the whole job. You write `Cache-Control: max-age=600` on the response and you're done.
  The browser's HTTP cache implements the storage, the expiry timer, and the
  `serve from cache without a network request` logic. You wrote no caching code.
- `ETag / 304` — header alone is NOT enough. You need to implement server side logic to cater to this.
    - If the browser never compares the `ETag` and just slaps a static `ETag` on every request, then it is
      sending a lot of redundant responses.

        ```
          Request1: GET /users/123

          Response1:  HTTP/1.1 200 OK
                      ETag: "abc123"

                      {
                        "name": "Faheem"
                      }

          Request2: GET /users/123
                    If-None-Match: "abc123" (This essentially means: "I already have version abc123. Is that still the current version?")

          Server WITHOUT ETag comparison
          Imagine your server code looks like this
          app.get("/users/123", (req, res) => {
              res.set("ETag", "abc123");
              res.status(200).json({
                  name: "Faheem"
              });
          });

          The server sets the ETag but never looks at the header
          req.headers["if-none-match"]

          So, the request comes with the header req.headers["if-none-match"] and the server simply
          ignores it.

          Response2:  HTTP/1.1 200 OK
                      ETag: "abc123"

                      {
                        "name": "Faheem"
                      }
        ```

    - Even worse, the browser never updates the `ETag` to be sent to the client and thus the browser
      will show stale data.

        ```
          Request1: GET /users/123

          Response1:  HTTP/1.1 200 OK
                      ETag: "abc123"

                      {
                        "name": "Faheem"
                      }

          Later the data changes in the db
                {
                  "name": "Faheem",
                  "age": 26
                }


          But imagine the server still returns the same ETag: "abc123"


          Request2: GET /user/123
                    If-None-Match: "abc123"

          The server compares the ETag
          Current ETag = "abc123"
          Client's ETag = "abc123"

          They match! and thus sends a 304 not modified status code

          HTTP/1.1 304 Not Modified

          The browser interprets 304 as:
            "My cached version is still valid."

          Thus, the browser will show stale data.
        ```



- `Connection: keep-alive` — below your code entirely.
  This happens at the network layer and not the application layer. Thus, the
  HTTP Library, the OS socket layer and the server handle connection reuse.
  In HTTP/1.1 it's the default and setting it changes nothing.

#### Representation headers
Describe the body being transmitted, in either direction.

- `Content-Type` — the media type: `application/json`, `text/html`.
- `Content-Length` — size of the resource in bytes.
- `Content-Encoding` — compression applied: `gzip`, `deflate`.
- `ETag` — a unique identifier for the resource version, used heavily in caching.

These ensure both sides know how to interpret and process the payload.

**Why does ETag come up in representational headers?**

- Because computing ETag for the same data but in xml and json would be different.
- SO it is purely presentational.