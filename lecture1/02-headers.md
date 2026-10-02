### Security Headers

- `Strict-Transport-Security` (HSTS) => used for downgrade attacks
  - If a client sends a request to the server using `http`, the server
    sends a header called the `Strict-Transport-Security` to let the browser
    know that the subsequent requests will only come via `https.`

  - The browser then remembers it and even if i type `http://example.com`,
    the browser will make a request to `https://example.com`.

  - If not for this header, a man in the middle attack is so prevelant as the
    data passed via http is not encrypted.

  - But now there is a catch, what if on the very first request that is made via
    `http`, the hacker intercepts it and never lets the servers `Strict-Transport-Security`
    header reach the client which means the browser will keep making requests using http
    and thus the attacker can steal credentials.

  - On the very first request, the browser doesn't know that the server is going to send this
    header, so **How will we prevent a downgrade attack in such case?**
  - HSTS Preload solves that: `Browsers maintain a built-in list of domains that should always use HTTPS.`

- `Content-Security-Policy` (CSP) =>  Cross-site scripting (XSS) — restricts where scripts, CSS, and images may load from.
  - Suppose our application has a compromised endpoint
    ```
    app.get("/hello", (req, res) => {
    const name = req.query.name;

    res.send(`
        <html>
            <body>
                <h1>Hello ${name}</h1>
            </body>
        </html>
    `);
    });
    ```

  - Normally you visit: `https://example.com/hello?name=Syed`
  - The browser generates
    ```html
      <h1>Hello Syed</h1>
    ```
  - The attacker notices name is directly inserted into the HTML.
  - So the attacker instead of name sends a malicous script and when the client side
    code executes it, this is called `cross site scripting.`

- `X-Frame-Options` => Clickjacking — prevents the page being embedded in an iframe
  - X-Frame-Options: DENY/SAMEORIGIN
    - DENY: Never allow the websit to be embedded as an iframe.
    - SAMEORIGIN: Allow framing only by pages from the same origin.

  ```
    evil.com
  ┌──────────────────────────────────────┐
  │                                      │
  │     🎁 YOU WON A FREE iPHONE!        │
  │                                      │
  │           [ CLAIM PRIZE ]            │
  │                                      │
  │                                      │
  │   ┌──────────────────────────────┐   │
  │   │ bank.com/transfer             │   │
  │   │                              │   │
  │   │      [ TRANSFER ]            │   │
  │   │                              │   │
  │   └──────────────────────────────┘   │
  │                                      │
  └──────────────────────────────────────┘
  ```

  - The attacker can make the websit position over the claim prize button and the client can think
    it clicked claim prize where as it clicked on `bank.com/transfer`.


- `X-Content-Type-Options` => MIME-type sniffing — stops the browser guessing content types.
  - **Why is it dangerous?**
  - Consider an attacker managing to upload something malicous on a compromised route on a server.
  - Now when the server serves that file(assuming the server saying that it is text/plain) but infact
    it is a malicous HTML page.
  - When the client's browser recieves it, it says that although the server says it plain text but it looks
    like HTML, so i will go ahead and parse and execute it as HTMl which is an attack vector.
  - Adding the header `X-Content-Type-Options: nosniff` prevents the browser from guessing the MIME type of the
    content being served to it.

- `Set-Cookie` with `HttpOnly` / `Secure` => Cookie theft — blocks JavaScript access, forces HTTPS-only transmission.
  - `Set-Cookie` is a servers response header. `HttpOnly` and `Secure` are the attributes.
  - `Set-Cookie: sessionId=abc123; HttpOnly`: HttpOnly means js running in the page cannot access the cookie.
    Useful as it prevents the attacker from stealing the cookie if your site has an XSS vulnerablity.
  - `Set-Cookie: sessionId=abc123; Secure`. The cookie would be sent over https only. So a man in the middle
    intercepting the traffic cannot steal the cookie. But if your site has an XSS vulnerability, then the attacker
    can steal the cookie by executing malicous script on ur website.
  - `Set-Cookie: sessionId=abc123; HttpOnly; Secure`. So neither the js can read it and the cookie will always be
    sent over https.

- Now even if this header is there and the attributes are there, if your site has an XSS vulnerablitiy, the attacker
  doesn't need to steal the cookie but instead can inject a malicous script on your website and makes a request to the
  server that sent the cookie, so the browser attaches the cookie and therefore the attacker can do dangerous operations.
  `Like changing password or email.`
- Now imagine if your website doesn't have an XSS vulnerability and you visit a website `evil.com`.
- The attacker manages to make a request to lets say `bank.com` and the browser attaches the cookie
  automatically and thus initiates a transfer of money on ur behalf.

#### Important distinction between XSS and CSRF

**XSS**
`The attacker gets JavaScript executing inside bank.com:`

```
bank.com
  └── attacker's JavaScript
          ↓
       fetch()
          ↓
       Cookie attached
```

**CSRF**
`The attacker controls a different website:`


```
evil.com
  └── malicious request
          ↓
       bank.com
          ↓
       Cookie may be attached
```

The attacker doesn't have JavaScript running as bank.com.
That's why SameSite cookies and CSRF tokens exist.

Now, CSRF tokens help in the way that when the user tries to make a request to transfer money.
The bank sends a form with csrf token attached into an input value and when the user makes a
request, the browser attaches the cookie and there is a csrf token also which the server
authenticates. The attacker doesn't have the csrf token even though he can make a request with
the cookie attached.
