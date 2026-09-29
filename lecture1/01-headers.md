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
