## Headers and Idempotency

**Idempotency**
- An idempotent method can be called multiple times and produce the same resulting state.
- GET/PUT/DELETE => Idempotent methods
- PATCH/POST => Non-Idempotent methods

**The Options Method**
- The Options request asks the server to describe the capabilities for
  a given resource => The methods and headers it allows for a given resource for a
  cross origin request. This request is used in the `CORS preflight flow.`

**The problem CORS solves**
- The browser inherently allows the client to make requests to the same origin.
- Origin comprises of scheme, domain and port.
- Naturally, a client cannot make requests to other origins.
- So, in order for a client to make cross origin requests, CORS comes in the picture.
- In order for a browser to make a preflight request, the browser needs to figure out
  whether the request is a simple or a non-simpke request.
- `Simple requests: ` A request is considered simple if it follows certain rules.
    - Http method must be one of `GET/HEAD/POST`.
    - The headers must stay within CORS satisfied headers
        - `Accept`
        - `Accept-Language`
        - `Content-Language`
        - `Content-Type` For content-type, there are certain that are allowed
          such as `application/x-www-form-urlencoded`
                  `multipart/form-data`
                  `text/plain`
- Non-Simple request is simply which doesn't satisfy the above constraints and thus
  the browser makes a pre-flight request before the actual request.

Example

```javascript
fetch("https://api.example.com/users", {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify({
    name: "John"
  })
});
```

`application/json` is not a simple header

Thus, the browser performs a pre-flight

```
Browser
   │
   │ OPTIONS
   │
   │ "Can I send POST with
   │  Content-Type: application/json?"
   ▼
Server
   │
   │ Access-Control-Allow-...
   ▼
Browser
   │
   │ POST
   ▼
Serve
```

The server must respond with these headers

```
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Headers: Content-Type
```

Certain headers to keep in mind which the server responds with when handling a preflight request

- `Access-Control-Allow-Origin` — yes, your origin is permitted (or * for all).
- `Access-Control-Allow-Methods` — these methods are allowed on this route, including the one you asked about.
- `Access-Control-Allow-Headers` — these non-simple headers are permitted.
- `Access-Control-Max-Age` — cache this policy for N seconds and stop preflighting every request.
  Setting 86400 means "don't ask again for 24 hours". This saves meaningful bandwidth and
  latency for both sides.

## Status codes

- These exist to give the result in a standardized way.
- Certain status codes to keep in mind.
- `201 Created` - the request succeeded and made a new resource.
- `204 No Content` - succeeded but there is nothing to return. Used for preflight requests and `DELETE`
- `301 Moved permanently` - the resource has permanently moved, future requests should use the new URL.
- `302 Found(temporary redirect)` - the resource is temporarily elsewhere but the client should keep using the same URL.
- `304 Not modified` - the resource hasn't changed since the client last fetched it.
- `400` Bad request => invalid data sent.
- `401` Unauthorised
- `403` Forbidden => user A tryng to delete user B's data.
- `404 Not found`
- `405 Method not allowed`
- `409 Conflict` => trying to create a resource which already exists.
- `429 Too many requests`
- `500 Internal server error`
- `501 Not implemented` => the server doesn't support the requested method or functionality yet.
- `502 Bad Gateway` => the reverse proxy recieved an invalid response from the upstream server.
- `503 Service unavailable` => Temporarily unavailable due to too much traffic
- `504 Request timeout` => upstream server never repsonded within the time limit.