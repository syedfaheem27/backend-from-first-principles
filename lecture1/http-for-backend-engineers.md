# Understanding HTTP for Backend Engineers

Detailed notes from the video transcript. The framing throughout is deliberately practical: the goal is not to cover all of HTTP, but the ~90% of it that shows up in real codebases, explained from first principles rather than through framework-specific code.

---

## Table of contents

1. [The two core ideas: statelessness and client–server](#1-the-two-core-ideas)
2. [What HTTP sits on: TCP and the OSI model](#2-what-http-sits-on)
3. [HTTP versions: 1.0 → 3.0](#3-http-versions)
4. [Anatomy of a request and response message](#4-anatomy-of-a-message)
5. [Headers](#5-headers)
6. [HTTP methods and idempotency](#6-http-methods-and-idempotency)
7. [CORS and the OPTIONS method](#7-cors-and-the-options-method)
8. [Status codes](#8-status-codes)
9. [HTTP caching](#9-http-caching)
10. [Content negotiation and compression](#10-content-negotiation-and-compression)
11. [Persistent connections and keep-alive](#11-persistent-connections-and-keep-alive)
12. [Large requests and responses](#12-large-requests-and-responses)
13. [SSL, TLS, HTTPS](#13-ssl-tls-https)
14. [Quick reference tables](#14-quick-reference)

---

## 1. The two core ideas

### Statelessness

HTTP has no memory of past interactions. Each request carries everything the server needs to process it — headers, URL, method, body. Once the server responds, it forgets the request. A follow-up request from the same client is treated as a brand new, unrelated event.

This forces **self-contained requests**. If you want to fetch a user profile, the client must send credentials (a cookie, a bearer token) on *every* request, because the server has no recollection of who you were a moment ago.

**Why this design is good:**

- **Simplicity** — the server doesn't have to store session state, which would cost memory and add complexity.
- **Scalability** — any server in a pool can handle any request, because no single server "owns" a session. Load balancing becomes trivial.
- **Crash resilience** — if a server dies, there's no session state to restore. The client's next request just goes somewhere else.

**The trade-off:** real applications *do* need continuity — logins, shopping carts. So developers layer state management on top: cookies, server-side sessions, JWTs. Statelessness is a property of the protocol, not a constraint on your application.

### The client–server model

Every HTTP exchange has exactly two roles:

- **Client** — a browser, mobile app, Postman, another server. It *initiates* communication and is responsible for supplying everything the server needs (URL, headers, body).
- **Server** — hosts resources (web pages, APIs, files), waits for requests, processes them, and sends back a response: HTML, JSON, plain text, an error, anything.

The critical rule: **communication is always initiated by the client.** The server never spontaneously starts an HTTP conversation. (Server push in HTTP/2 is a narrow exception to the data flow, not to who opens the connection.)

Throughout these notes, HTTP and HTTPS can be treated as interchangeable. HTTPS is HTTP with encryption and certificates layered underneath — the semantics of methods, headers, and status codes are identical.

---

## 2. What HTTP sits on

Before any request can be sent, the client and server need a communication channel. HTTP doesn't strictly require a *connection-oriented* transport — it requires a **reliable** one that doesn't silently lose messages (and surfaces an error if it does).

Of the two dominant transport protocols on the internet, TCP and UDP, TCP is the reliable one. So HTTP (through version 2) rides on TCP, which is connection-based and uses a three-way handshake to establish a connection.

### Where you live in the OSI model

The OSI model is the standard layered view of network communication. As a backend engineer, you work at **Layer 7 — the application layer**. That's where HTTP lives.

Topics like the TCP three-way handshake, TLS negotiation, packet routing, and MTU sizing sit in the lower layers and are properly network-engineering concerns. Worth knowing they exist; not worth a rabbit hole. The one thing to internalize:

> A network connection gets established between client and server, and messages flow across it. That's the level of detail the application layer needs.

---

## 3. HTTP versions

| Version | Key change | Why it mattered |
|---|---|---|
| **HTTP/1.0** | One connection per request | Every request/response pair required opening and tearing down a TCP connection. Expensive and slow. |
| **HTTP/1.1** | Persistent connections | Multiple requests and responses reuse a single TCP connection. Also added chunked transfer encoding and better caching. Still the most widely used version. |
| **HTTP/2.0** | Multiplexing | Multiple requests/responses concurrently over one connection. Uses **binary framing** instead of text, **HPACK** header compression, and **server push** (server sends resources before the client asks). |
| **HTTP/3.0** | Built on QUIC (over UDP) | Faster connection establishment, lower latency, better packet-loss handling. Keeps multiplexing but eliminates **head-of-line blocking**, which still affects HTTP/2. |

Note the inversion in HTTP/3: it abandons TCP for UDP, and rebuilds reliability inside QUIC at the transport layer.

---

## 4. Anatomy of a message

### Request message

```http
PUT /api/users/42 HTTP/1.1
Host: api.example.com
Origin: https://example.com
Authorization: Bearer eyJhbGciOi...
Content-Type: application/json
Content-Length: 48

{"name": "Faheem", "role": "backend engineer"}
```

Breaking it down:

| Part | Example | Meaning |
|---|---|---|
| Request method | `PUT` | The intent of the action |
| Resource URL | `/api/users/42` | What is being requested |
| HTTP version | `HTTP/1.1` | Protocol version in use |
| Host header | `api.example.com` | The target domain |
| Other headers | `Authorization`, `Content-Type`, … | Metadata (see next section) |
| **Blank line** | | Signals headers are over, body begins |
| Request body | `{...}` | Data the client is sending |

### Response message

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=10
ETag: "3141"

{"id": 42, "name": "Faheem"}
```

| Part | Example | Meaning |
|---|---|---|
| HTTP version | `HTTP/1.1` | |
| Status code + reason | `200 OK` | Outcome of the request |
| Response headers | `Content-Type`, `ETag`, … | Metadata about the response |
| **Blank line** | | Separator |
| Response body | `{...}` | The actual payload |

The blank line separating headers from body is structurally important in both directions — it's how the parser knows where metadata ends and payload begins.

---

## 5. Headers

**Definition:** headers are key–value pairs carrying metadata about the request or the response.

### Why headers exist at all

The obvious question: why not stuff everything into the URL or the body? Why invent a separate section?

The transcript's analogy is a good one. When you ship a parcel, the recipient's name, address, phone number, and PIN code go **on the outside**, not inside the box. Everyone handling the package along the route — sorting facility, delivery driver, customs — needs to read that metadata repeatedly. If it were inside, every handler would have to open the parcel.

Headers work the same way. Proxies, load balancers, caches, and the server itself can all read routing and handling metadata without parsing (or even decompressing) the body.

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

#### Representation headers
Describe the body being transmitted, in either direction.

- `Content-Type` — the media type: `application/json`, `text/html`.
- `Content-Length` — size of the resource in bytes.
- `Content-Encoding` — compression applied: `gzip`, `deflate`.
- `ETag` — a unique identifier for the resource version, used heavily in caching.

These ensure both sides know how to interpret and process the payload.

#### Security headers
Control browser behaviour to defend against specific attacks.

| Header | Protects against |
|---|---|
| `Strict-Transport-Security` (HSTS) | Protocol downgrade attacks — forces HTTPS only |
| `Content-Security-Policy` (CSP) | Cross-site scripting (XSS) — restricts where scripts, CSS, and images may load from |
| `X-Frame-Options` | Clickjacking — prevents the page being embedded in an iframe |
| `X-Content-Type-Options` | MIME-type sniffing — stops the browser guessing content types |
| `Set-Cookie` with `HttpOnly` / `Secure` | Cookie theft — blocks JavaScript access, forces HTTPS-only transmission |

### Two big ideas about headers

**1. Extensibility.** HTTP is highly extensible *because* of headers. You can add or customise headers without changing the underlying protocol. Entire behaviours change based on metadata alone:

- Security enhancements (HSTS and friends) were bolted on this way.
- Custom application headers (`X-Custom-Header`) can be invented freely.
- Content negotiation (`Accept`, `Accept-Language`, `Accept-Encoding`) lets one endpoint serve many representations.

**2. Headers as a remote control.** Headers let the client send *instructions and preferences* that influence how the server responds:

- **Content negotiation** — client says "I want HTML" via `Accept`, server obliges; says "I want JSON", server sends JSON.
- **Caching control** — server uses `Cache-Control` or `Expires` to dictate how long the client may reuse a resource.
- **Authentication** — `Authorization` drives access-control decisions on the server.

---

## 6. HTTP methods and idempotency

Methods exist so that requests can express **intent**. Without them, every request would be semantically identical and you'd have to encode meaning in the URL or body.

| Method | Intent | Has body? |
|---|---|---|
| `GET` | Fetch data. Must not modify server state. | No |
| `POST` | Create a new resource. | Yes |
| `PATCH` | Partially update a resource — append or selective replacement. | Yes |
| `PUT` | Update by **complete replacement** of the resource. | Yes |
| `DELETE` | Remove a resource. | Usually no |

### PUT vs PATCH

Both update, but they differ in semantics. `PATCH` modifies part of a resource; `PUT` replaces it wholesale with whatever the body contains.

In practice developers frequently reach for `PUT` when they mean `PATCH`, violating the semantics. The rule of thumb from the video:

> **Default to `PATCH` unless you specifically need full replacement.**

Example: a user-profile edit form where someone changes only their display name is a `PATCH`. Sending the entire user object to overwrite the stored one is a `PUT`.

### Idempotency

An **idempotent** method can be called multiple times and produce the same resulting state.

**Idempotent:**

- `GET` — fetching data any number of times returns the same data and changes nothing.
- `PUT` — replacing a resource with identical data N times leaves it in the same state as doing it once.
- `DELETE` — a resource can only be deleted once; repeating it doesn't delete it "more".

**Non-idempotent:**

- `POST` — each call creates a new resource. If a user submits a "create note" request twice, you get two notes. Same request, different outcomes.

This matters practically: idempotent operations are safe to retry automatically after a network failure. `POST` retries risk duplicate records, which is why real systems add idempotency keys.

### The OPTIONS method

You will rarely write an `OPTIONS` request yourself, but you'll see them constantly in the browser's Network tab. `OPTIONS` asks the server to describe its **capabilities** for a given resource — which methods and headers it permits for a cross-origin request. It exists almost entirely to serve the CORS preflight flow.

---

## 7. CORS and the OPTIONS method

### The problem CORS solves

Browsers enforce the **same-origin policy**: by default, a web page may only make requests to the origin that served it. A page on `example.com` cannot freely call `api.anotherexample.com`.

An *origin* is scheme + domain + port. Even `localhost:5173` and `localhost:3000` are different origins.

**CORS (Cross-Origin Resource Sharing)** is the browser-enforced mechanism that lets servers opt in — specifying *who* may access their resources and *how*. Without it, all cross-origin requests from web applications would be blocked.

Two things worth being precise about:

- CORS is enforced **by the browser**, not the server. The server just declares policy via headers; the browser decides whether to hand the response to your JavaScript.
- The request often *does* reach the server and *is* processed. A "CORS error" usually means the response was blocked on arrival, not that the request never went out.

### Flow A — Simple requests

Setup: frontend at `example.com`, API at `api.example.com`, making a `GET`.

1. The client sends the request. The browser **automatically adds an `Origin` header** identifying where the request came from.
2. The method is a simple one (`GET`, `POST`, or `HEAD`), so no preflight happens.
3. The server receives it and checks `Origin` against its CORS policy.
4. If allowed, the server includes `Access-Control-Allow-Origin` in the response.
5. The browser inspects the response. It notices `Origin` (`example.com`) differs from `Host` (`api.example.com`), so it looks for `Access-Control-Allow-Origin`.
6. If that header contains the client's origin — or `*`, meaning all origins — the browser passes the response to the JavaScript that requested it.

**Failure case:** if the server omits the header, or doesn't allow this particular origin, the header is absent from the response. The browser sees the absence and **blocks the response**. You get a CORS error in the console and the request shows as failed in the Network tab — even though the server may have processed it perfectly well.

### Flow B — Preflighted requests

The browser sends a preliminary `OPTIONS` request before the real one when the request is cross-origin **and any one** of these is true:

1. The method is **not** `GET`, `POST`, or `HEAD` — e.g. `PUT` or `DELETE`.
2. The request includes **non-simple headers** — anything beyond the basic general/request headers, e.g. `Authorization` or custom headers.
3. The `Content-Type` is something other than `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`.

Condition 3 is why this matters so much in practice: **any JSON API request triggers a preflight**, because `application/json` isn't a simple content type. Most real API traffic is preflighted.

**The preflight request:**

```http
OPTIONS /api/users/42 HTTP/1.1
Host: api.example.com
Origin: https://example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: authorization, content-type
```

It carries **no body**. It's purely an inquiry: *"I'm making a cross-origin request. Do you support `PUT` on this route? Do you accept these headers?"*

**The preflight response** from a properly configured server:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: content-type, authorization
Access-Control-Max-Age: 86400
```

Reading it line by line:

- **204 No Content** — success, nothing in the body, because this was only an inquiry.
- **`Access-Control-Allow-Origin`** — yes, your origin is permitted (or `*` for all).
- **`Access-Control-Allow-Methods`** — these methods are allowed on this route, including the one you asked about.
- **`Access-Control-Allow-Headers`** — these non-simple headers are permitted.
- **`Access-Control-Max-Age`** — cache this policy for N seconds and stop preflighting every request. Setting 86400 means "don't ask again for 24 hours". This saves meaningful bandwidth and latency for both sides.

If the server doesn't handle CORS, it won't respond with these headers, and the browser blocks the flow before the real request is ever sent.

Once the browser has ticked off every condition, it sends the **original request** (`PUT`, with body, `Authorization` header, etc.), and the server responds normally.

### What the live demo showed

Using an HTTP interception tool against a local frontend (`localhost:5173`) and API (`localhost:3000`):

- Different ports on `localhost` are treated as different origins, so everything was cross-origin.
- In the simple-request case, `Origin` and `Host` differed, and `Access-Control-Allow-Origin: http://localhost:5173` in the response let it through.
- Removing that one header from the server caused the browser to block the response with a CORS error — the request still went out and returned, the browser just refused to hand it over.
- The preflight case fired an `OPTIONS` request first, got `204` back, then fired the real `PUT`. All three preflight conditions were satisfied simultaneously: non-simple method (`PUT`), `Authorization` header, and `Content-Type: application/json` — though any one alone would have triggered it.
- `Access-Control-Max-Age` was set to `0` during the demo specifically so a preflight would fire on every request and be observable. In production you'd set it high.

---

## 8. Status codes

### Why they exist

Status codes communicate the result of a request in a **standardised** way. Without them, the client would have to infer success or failure by inspecting the body — "if it succeeded I'd expect this shape, if it failed I'd expect that shape, if the server crashed maybe a null object." That's fragile and inconsistent.

With status codes you get:

- **Instant outcome signalling** — read one number, know what happened, no body parsing needed.
- **Structured error handling** — a `401` means "log the user out and show the login screen"; a `400` means "show the user a form validation error and let them resubmit."
- **Cross-platform standardisation** — Python, Go, Rust, JavaScript, Ruby servers all follow the same convention. Success is `200`. Resource created is `201`. Universal language between any client and any server.

### The five classes

| Range | Class |
|---|---|
| `1xx` | Informational |
| `2xx` | Success |
| `3xx` | Redirection |
| `4xx` | Client errors |
| `5xx` | Server errors |

Day to day, you'll live in `2xx`, `4xx`, and `5xx`.

### 1xx — Informational

- **`100 Continue`** — the server has received the headers and the client may proceed to send the body. Used for large uploads: send headers first, get the go-ahead, then commit to sending megabytes.
- **`101 Switching Protocols`** — the server is switching protocols as requested, e.g. upgrading from HTTP to WebSocket.

You'll rarely handle these explicitly.

### 2xx — Success

- **`200 OK`** — the most common code. Request succeeded and the server is returning the resource or has performed the action. Typical for a successful `GET`.
- **`201 Created`** — the request succeeded *and* created a new resource. Use for `POST` / form submissions that create something.
- **`204 No Content`** — succeeded, but there's nothing to return. Used for CORS preflight responses, and commonly for `DELETE`: "I deleted it, there's no body to send back."

### 3xx — Redirection

- **`301 Moved Permanently`** — the resource has permanently moved; future requests should use the new URL. Example: you rename a route from `/user` to `/person`. Adding a 301 on `/user` keeps old clients and bookmarks working instead of breaking them.
- **`302 Found` (temporary redirect)** — the resource is *temporarily* elsewhere, but the client should keep using the original URL in future. Example: for a few hours during a campaign you redirect a route to a special landing page, then revert.
- **`304 Not Modified`** — the resource hasn't changed since the client last fetched it, so reuse the cached copy. Used with conditional `GET` requests and `ETag`s. Covered in detail in the caching section.

The 301-vs-302 distinction matters because browsers and search engines cache 301s aggressively. Using 301 for something temporary is hard to undo.

### 4xx — Client errors

These are the ones you'll return most often as a backend engineer.

| Code | Name | When to return it |
|---|---|---|
| `400` | Bad Request | The client sent invalid or illogical data. You expected a number and got an array; you expected an email and got a phone number. Signals "fix your request format and try again." |
| `401` | Unauthorized | Authentication required but missing or invalid — no JWT sent, or the token has expired. |
| `403` | Forbidden | The server understood the request and the client *is* authenticated, but refuses to authorise it. User A trying to delete User B's resource. |
| `404` | Not Found | The requested resource doesn't exist — wrong URL, or it was deleted. |
| `405` | Method Not Allowed | Wrong HTTP method for this route — `PUT` to something that only accepts `GET` or `POST`. Often the result of a typo on the frontend. |
| `409` | Conflict | The request conflicts with current state. Classic case: unique-name constraints — a user tries to create a folder whose name already exists, so you return 409 and they pick a different name. |
| `429` | Too Many Requests | Rate limiting. If your server permits 60 requests per second per client and one exceeds it, return 429. |

**401 vs 403** is the distinction people get wrong most often: 401 is "I don't know who you are"; 403 is "I know exactly who you are, and you're not allowed."

### 5xx — Server errors

| Code | Name | Meaning |
|---|---|---|
| `500` | Internal Server Error | Something unexpected broke — an unhandled exception. Rather than hanging or returning an empty response, return 500 so the client knows the failure is on the server side. Keep the message vague for security reasons; don't leak stack traces. |
| `501` | Not Implemented | The server doesn't support the requested method or functionality *yet*, but intends to. Communicates intent, not permanent refusal. |
| `502` | Bad Gateway | A proxy or reverse proxy (Nginx, a load balancer) received an invalid response from the upstream server. You don't usually return this yourself — infrastructure does. |
| `503` | Service Unavailable | Temporarily unable to handle the request — heavy traffic or maintenance. Tells the client to retry later. |
| `504` | Gateway Timeout | Like 502, but specifically: the upstream server didn't respond within the timeout window. Nginx couldn't get a reply from your app server in time. |

502 and 504 both point at proxy-layer problems, and the distinction is diagnostically useful: 502 means the upstream *answered badly*, 504 means it *didn't answer at all*.

---

## 9. HTTP caching

### The idea

HTTP caching stores copies of responses for reuse, avoiding repeated round trips to the server. The payoff is threefold: faster load times, less bandwidth consumed, and reduced server load. If the data hasn't changed, there's no reason for the client to download it again or the server to send it.

### The three headers that drive it

Set by the server on the initial response:

- **`Cache-Control`** — e.g. `max-age=10`, meaning "keep this cached for at most 10 seconds."
- **`ETag`** — a unique identifier for this version of the resource. Normally a hash computed from the response body. (In the video's demo it was a random number for simplicity.)
- **`Last-Modified`** — the timestamp of the last change to the resource.

Sent by the client on subsequent requests:

- **`If-None-Match`** — carries the `ETag` the client already holds.
- **`If-Modified-Since`** — carries the `Last-Modified` timestamp the client already holds.

Together these say: *"If the ETag no longer matches, or the resource has changed since this timestamp, send me the new version. Otherwise I'll use my cached copy."*

### The full cycle, as demonstrated

**Step 1 — initial fetch**

```http
GET /api/resource HTTP/1.1
```
```http
HTTP/1.1 200 OK
Cache-Control: max-age=10
ETag: "3141"
Last-Modified: Mon, 01 Jan 2024 10:00:00 GMT

{...body...}
```

Client stores the body along with the ETag and timestamp.

**Step 2 — refetch, nothing changed**

```http
GET /api/resource HTTP/1.1
If-None-Match: "3141"
If-Modified-Since: Mon, 01 Jan 2024 10:00:00 GMT
```
```http
HTTP/1.1 304 Not Modified
```

The server compares the incoming ETag and timestamp against current state. They match, so it returns **304 with no body**. The client reuses its cached copy. This is where the bandwidth saving lives — the response is a few hundred bytes of headers instead of the full payload.

**Step 3 — resource is updated**

An update request comes in and the server responds `200 OK` with a **new ETag** (`"2943"`) and a new `Last-Modified`.

**Step 4 — refetch with a stale ETag**

The client still holds `"3141"` and sends it. The server sees the mismatch and responds **`200 OK`** with the full body, the new ETag `"2943"`, and the updated timestamp.

**Step 5 — refetch again**

Now the client holds `"2943"`. It matches, so the server responds **304** again.

### Practical caveats

In production this gets complicated, because the server must generate and manage ETags correctly. If you forget to update an ETag after a change, clients will keep serving stale data indefinitely — a genuinely nasty class of bug.

The video's view is that modern client-side caching layers (React Query and similar) are often the better solution: the client gets explicit control over when to use a cached value, when to refetch, and at what interval. HTTP-level caching remains worth understanding and is perfectly adequate when the use case is simple.

---

## 10. Content negotiation and compression

### Content negotiation

A mechanism by which client and server **agree on the best format** for exchanging data. The client states its preferences; the server tries to honour them, falling back to something else if it can't.

Three types:

| Type | Client header | Example values |
|---|---|---|
| **Media type** | `Accept` | `application/json`, `application/xml`, `text/html` |
| **Language** | `Accept-Language` | `en`, `es` |
| **Encoding** | `Accept-Encoding` | `gzip`, `deflate`, `br`, `zstd` |

**What the demo showed:** a single endpoint served materially different responses purely based on headers.

- Default request: `Accept-Language: en`, `Accept: application/json` → JSON response in English.
- Change to `Accept-Language: es`, everything else identical → the *same endpoint* returned Spanish content.
- Change to `Accept: application/xml` → the same endpoint returned XML, still in Spanish.

The endpoint and the URL never changed. Only headers did. This is the "headers as remote control" idea made concrete, and it lets one route serve many client types cleanly.

### HTTP compression

Falls under the same umbrella, driven by `Accept-Encoding` (client) and `Content-Encoding` (server).

The demo used a JSON file containing 11,000 entries:

| Compression | Transferred size |
|---|---|
| gzip enabled | **3.8 MB** |
| gzip disabled | **26 MB** |

Same file, roughly a 7× difference. The client advertises `Accept-Encoding: gzip, deflate, br, zstd`; the server compresses and marks the response `Content-Encoding: gzip`; the browser transparently decompresses it. The application code on both ends is unaware any of this happened.

For any sizeable response, this is close to free performance — and multiplied across every client, the bandwidth savings are enormous.

---

## 11. Persistent connections and keep-alive

In HTTP/1.0, every request–response cycle needed its own TCP connection. Since establishing and tearing down TCP connections is resource-intensive and slow, this was a major inefficiency.

HTTP/1.1 introduced **persistent connections**: a single TCP connection is reused across multiple requests and responses, avoiding the open/close overhead each time.

Key points:

1. **Persistent by default in HTTP/1.1.** Connections stay open for further requests unless explicitly closed. You don't need to configure anything. Multiple request/response pairs flow over one connection, cutting latency and conserving resources.

2. **The `Connection: keep-alive` header.** Although persistence is the default, this header can still be used to explicitly ask the server to hold the connection open. It can carry options like `timeout` (how long to stay open) and `max` (how many requests before closing).

3. **`Connection: close`.** Closes the connection after the response is sent. This was HTTP/1.0's default behaviour and can still be explicitly requested in HTTP/1.1.

In practice the defaults work fine and you'll rarely touch these — but keep-alive reappears meaningfully in streaming, covered next.

---

## 12. Large requests and responses

Normal JSON payloads are small. Files — video, audio, images, large datasets — need different mechanisms in each direction.

### Client → server: multipart requests

`multipart/form-data` is used for sending files. Rather than one contiguous blob, the binary data is transferred **in parts** — hence "multipart".

What the request looks like:

```http
POST /api/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryAbC123
Content-Length: 284910

------WebKitFormBoundaryAbC123
Content-Disposition: form-data; name="file"; filename="photo.jpg"
Content-Type: image/jpeg

<binary data>
------WebKitFormBoundaryAbC123--
```

The critical piece is the **boundary** parameter. Because the binary data is transferred in parts, something must delimit where each part starts and ends. The boundary string is that delimiter — you'll find it immediately before the binary data begins and again after it ends. The server uses it to split the stream back into discrete parts.

**Rule:** whenever you're transferring files to a server, use multipart.

### Server → client: streaming in chunks

For large responses, the server streams data to the client in chunks rather than buffering the whole thing.

The request is unremarkable — a plain `GET`. The response is where the interesting headers live:

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Connection: keep-alive
```

- **`Content-Type: text/event-stream`** — this is not a static text response; data will arrive as a stream of events.
- **`Connection: keep-alive`** — hold the connection open until all data has been sent.

In the demo, chunks arrived continuously and the client appended each one, progressively reconstructing the complete file while it was still downloading. Notice there's no `Content-Length` — the server doesn't know the total size upfront, which is precisely the point.

---

## 13. SSL, TLS, HTTPS

You won't work with these directly, but you should know what the words mean.

**SSL (Secure Sockets Layer)** was the original protocol for securing communication between client and server. It encrypts data so sensitive information — passwords, credit card numbers — can't be intercepted by attackers. **SSL is now outdated** due to known security vulnerabilities and has been replaced by TLS. (The name persists everywhere out of habit — "SSL certificate" almost always means a TLS certificate.)

**TLS (Transport Layer Security)** is the modern, more secure successor. It encrypts data **in transit**, protecting it from interception and tampering. TLS uses **certificates** to authenticate the server and establish the encrypted connection, preventing eavesdropping and data breaches. It's continuously updated; TLS 1.3 is the current recommended version.

**HTTPS** is simply HTTP with the security features provided by TLS underneath. When you visit an HTTPS site, TLS encrypts the communication between browser and server, protecting credentials and other sensitive data from interception.

That's the appropriate depth for application-level work. Certificate chains, cipher suites, and handshake internals belong to the network/security domain.

---

## 14. Quick reference

### Method semantics

| Method | Idempotent | Body | Typical success code |
|---|---|---|---|
| `GET` | Yes | No | 200 |
| `POST` | **No** | Yes | 201 |
| `PUT` | Yes | Yes | 200 |
| `PATCH` | Not guaranteed | Yes | 200 |
| `DELETE` | Yes | No | 204 |
| `OPTIONS` | Yes | No | 204 |

### CORS headers cheat sheet

| Header | Direction | Purpose |
|---|---|---|
| `Origin` | Request | Where the request came from (browser sets automatically) |
| `Access-Control-Request-Method` | Preflight request | "Do you allow this method?" |
| `Access-Control-Request-Headers` | Preflight request | "Do you allow these headers?" |
| `Access-Control-Allow-Origin` | Response | Permitted origin, or `*` |
| `Access-Control-Allow-Methods` | Preflight response | Permitted methods |
| `Access-Control-Allow-Headers` | Preflight response | Permitted non-simple headers |
| `Access-Control-Max-Age` | Preflight response | Seconds to cache this policy |

### Preflight trigger conditions

Cross-origin **AND any one** of:
1. Method is not `GET` / `POST` / `HEAD`
2. Request includes non-simple headers (e.g. `Authorization`, custom headers)
3. `Content-Type` is not `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`

→ **Any JSON API call triggers a preflight.**

### Caching headers cheat sheet

| Header | Direction | Purpose |
|---|---|---|
| `Cache-Control` | Response | How long the resource may be cached |
| `ETag` | Response | Version identifier (usually a hash) |
| `Last-Modified` | Response | Timestamp of last change |
| `If-None-Match` | Request | Client's held ETag |
| `If-Modified-Since` | Request | Client's held timestamp |
| → `304 Not Modified` | Response | Nothing changed, reuse your cache |

### Status codes at a glance

| Code | Meaning | Code | Meaning |
|---|---|---|---|
| 100 | Continue | 403 | Forbidden |
| 101 | Switching Protocols | 404 | Not Found |
| 200 | OK | 405 | Method Not Allowed |
| 201 | Created | 409 | Conflict |
| 204 | No Content | 429 | Too Many Requests |
| 301 | Moved Permanently | 500 | Internal Server Error |
| 302 | Found (temporary) | 501 | Not Implemented |
| 304 | Not Modified | 502 | Bad Gateway |
| 400 | Bad Request | 503 | Service Unavailable |
| 401 | Unauthorized | 504 | Gateway Timeout |

---

## Closing thought

The closing argument of the video is worth keeping: this is roughly the complete set of HTTP knowledge needed to build and debug backend systems. There's always more depth available — TLS internals, TCP behaviour, HTTP/3 and QUIC — but if you can visualise the full flow of the components above and understand *why* each exists, you can reason about and debug most of what you'll meet in production.

The recurring theme: HTTP is deliberately simple at its core (stateless, client-initiated, request/response), and nearly all of its sophistication is layered on through **headers**. Caching, CORS, content negotiation, compression, authentication, streaming, security policy — all of it is metadata negotiation on top of an unchanged base protocol. That's what makes HTTP extensible, and it's why understanding headers well gets you most of the way to understanding HTTP.
