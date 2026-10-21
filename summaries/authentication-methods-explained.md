# Authentication Methods, Explained in Detail

A line-by-line companion to [authentication-methods-deep-dive.md](authentication-methods-deep-dive.md). The deep-dive is a dense **map**: every method fits into a table row or a short paragraph. This file slows down and explains **every idea, term, diagram and code snippet** in it: what it means, why it exists, how it works step by step, and what goes wrong.

**How to read it:** the section numbers here match the deep-dive (§5 here explains §5 there). Read the deep-dive section first, then the matching section here. If a term confuses you, it's probably explained in **Part 0**, which covers the building blocks everything else depends on.

---

## Table of contents

0. [Foundations: the building blocks you need first](#0-foundations-the-building-blocks-you-need-first)
1. [The map: three layers of authentication](#1-the-map-three-layers-of-authentication)
2. [Password-based methods](#2-password-based-methods)
3. [One-time passwords (OTP)](#3-one-time-passwords-otp)
4. [Magic links](#4-magic-links)
5. [Passkeys, WebAuthn and FIDO2](#5-passkeys-webauthn-and-fido2)
6. [Biometrics and Apple Face ID](#6-biometrics-and-apple-face-id)
7. [Certificate and key-based authentication](#7-certificate-and-key-based-authentication)
8. [Single Sign-On (SSO)](#8-single-sign-on-sso)
9. [Enterprise and network protocols](#9-enterprise-and-network-protocols)
10. [Machine-to-machine beyond API keys](#10-machine-to-machine-beyond-api-keys)
11. [HTTP authentication schemes](#11-http-authentication-schemes)
12. [Patterns layered on top](#12-patterns-layered-on-top)
13. [Emerging methods](#13-emerging-methods)
14. [Reading the comparison table](#14-reading-the-comparison-table)
15. [The reasoning behind "what to use when"](#15-the-reasoning-behind-what-to-use-when)
16. [The closing thought, unpacked](#16-the-closing-thought-unpacked)
17. [Check your understanding](#17-check-your-understanding)

---

## 0. Foundations: the building blocks you need first

Every authentication method in the deep-dive is assembled from a small set of primitives. If you understand these ten ideas, every later section becomes "oh, it's *that* idea again, arranged differently".

### 0.1 Authentication vs authorization, and the three factors

- **Authentication (authn)** answers *"who are you?"*. It turns an anonymous request into "this is user 42".
- **Authorization (authz)** answers *"what are you allowed to do?"*. It happens *after* authentication: "user 42 is an editor, so they may edit this note".

A **factor** is a category of proof:

| Factor | Meaning | Examples |
|---|---|---|
| **Something you know** | A secret in your head | Password, PIN, security-question answer |
| **Something you have** | A physical or digital object you possess | Phone with an authenticator app, YubiKey, smart card, a private key on a device |
| **Something you are** | A property of your body | Face, fingerprint, iris, voice |

**Multi-factor authentication (MFA)** means combining **different categories**. A password plus a second password is still one factor (both are "know"). A password plus a code from your phone is two (know + have).

Why combine categories? Because each category fails in a *different* way. Passwords leak in database breaches and phishing; phones get stolen; faces can be photographed. An attacker who has one kind of proof rarely has the other at the same time.

### 0.2 Encoding, hashing, encryption and signing are four different things

Beginners often mix these up. The deep-dive relies on the difference constantly.

| Operation | Reversible? | Needs a key? | Purpose | Example |
|---|---|---|---|---|
| **Encoding** | Yes, by anyone | No | Represent bytes in a different alphabet so they survive transport | Base64, base32, hex, URL encoding |
| **Hashing** | **No** (one-way) | No | Make a fixed-size fingerprint of data | SHA-256, SHA-1 |
| **Encryption** | Yes, with the key | Yes | Keep data **secret** | AES, RSA encryption |
| **Signing / MAC** | Not applicable | Yes | Prove **who produced** the data and that it **wasn't changed** | HMAC, ECDSA, RSA signatures |

- **Encoding is not security.** Base64 of `alice:s3cr3t` is `YWxpY2U6czNjcjN0`, and anyone can decode it instantly. This matters in §2.1 (HTTP Basic).
- **A hash** is a one-way function: easy to compute `SHA-256("hello")`, practically impossible to go backwards. The same input always gives the same output, and a tiny change to the input changes the output completely.
- **Encryption** is reversible by whoever holds the key. It gives *confidentiality* (secrecy), but on its own it doesn't tell you who sent the message.
- **Signing** gives *authenticity* (who made it) and *integrity* (it wasn't modified). It doesn't hide anything.

### 0.3 Password hashing: salt and slowness

Servers should never store passwords, only **hashes** of them. But a plain fast hash like SHA-256 isn't enough:

- **Problem 1: identical passwords give identical hashes.** If a million users chose `123456`, they'd all have the same hash, and attackers have precomputed tables ("rainbow tables") of hashes for common passwords.
  **Fix: a salt**, a random value generated per user and stored next to the hash. `hash(salt + password)` makes every user's hash unique, so precomputed tables are useless.
- **Problem 2: fast hashes are fast to guess.** A GPU can compute billions of SHA-256 hashes per second, so a leaked table of fast hashes can be cracked by trying every common password.
  **Fix: a deliberately slow hash** (Argon2id, bcrypt, scrypt) tuned to take, say, 100 ms and lots of memory. That's unnoticeable for one login but makes guessing billions of passwords infeasible.

Keep this in mind: later sections contrast "things we can hash" (passwords, API keys, magic-link tokens) with "things we **can't** hash because the server needs the original value" (TOTP secrets, HMAC keys).

> **Why random tokens don't need slow hashes:** a 32-byte random token has 2²⁵⁶ possible values. Nobody can guess it, fast hash or not. Slow hashing only matters for *low-entropy* human-chosen secrets like passwords. So magic-link tokens and API keys are stored as plain SHA-256 hashes.

### 0.4 Entropy and randomness

**Entropy** measures how unpredictable a secret is, in bits. Each bit doubles the number of possibilities an attacker must try.

- A 6-digit code: 10⁶ = 1,000,000 possibilities (about 20 bits). Guessable in seconds without rate limiting.
- A random 32-byte token: 2²⁵⁶ possibilities. More than the number of atoms in the observable universe.

Security tokens must come from a **cryptographically secure random number generator (CSPRNG)**: `crypto.randomBytes()` in Node, `secrets` in Python, `crypto/rand` in Go. Ordinary `Math.random()` is predictable and must never be used for secrets.

### 0.5 Symmetric vs asymmetric cryptography

This is the single most important idea in the whole topic.

**Symmetric (shared secret):** both sides hold **the same key**.
- Used by: passwords, HMAC, TOTP, API keys, Kerberos.
- Weakness: the server must hold the secret (or something derived from it), so **the secret can leak from either side**. Anyone who learns it can impersonate you.

**Asymmetric (key pair):** you generate a **private key** and a matching **public key**.
- The **private key** stays with you. It can **sign** data.
- The **public key** can be shared with anyone. It can **verify** signatures made by the private key, but it **cannot create** them.
- Used by: passkeys, SSH keys, TLS certificates, smart cards, JWTs signed with RS256/ES256.
- Strength: the server stores only the public key. **If the server's database leaks, the attacker can't log in as you**, because a public key can only check signatures, not produce them.

Analogy: a public key is like a **wax-seal pattern** everyone knows. Anyone can look at a letter and check "yes, that seal matches Alice's ring". Only Alice has the ring (the private key) that can press the seal.

### 0.6 Digital signatures and MACs

- **Digital signature** (asymmetric): `signature = Sign(privateKey, message)`; anyone can run `Verify(publicKey, message, signature) → true/false`. Algorithms: ECDSA (e.g. P-256), Ed25519, RSA.
- **MAC, Message Authentication Code** (symmetric): `mac = HMAC(sharedKey, message)`. Only someone with the same key can compute or check it.

**HMAC** (Hash-based MAC) mixes a secret key into a hash function in a specific, proven-safe way. `HMAC-SHA256(key, message)` produces 32 bytes that:
- change completely if the message changes by one bit, and
- can't be computed without the key.

HMAC shows up in TOTP (§3), webhook signatures (§10.2), AWS request signing (§10.1) and HS256 JWTs.

### 0.7 Challenge–response, nonces and replay attacks

A **replay attack** is when an attacker records a valid authentication message and simply sends it again later.

If you prove your identity by sending the same thing every time (like a password), a recorded copy works forever. The fix is **challenge–response**:

1. The server sends a fresh random value, a **challenge** (also called a **nonce**: "number used once").
2. The client proves its identity *about that specific challenge*, e.g. by signing it.
3. The server checks the proof and **throws the challenge away**.

A recorded response is useless next time, because the next login uses a different challenge. Timestamps play a similar role: "this proof is only valid for the next 5 minutes". Passkeys, SSH, Kerberos, mTLS, HMAC signing and Digest auth all use this idea.

### 0.8 Constant-time comparison (timing attacks)

When code compares two strings with `===`, it usually stops at the **first different character**. Comparing a wrong guess that matches the first 5 characters takes slightly longer than one matching 0 characters. By measuring response times over many requests, an attacker can recover a secret one character at a time.

`crypto.timingSafeEqual(a, b)` always compares every byte, so the time doesn't depend on where the difference is. **Use it whenever you compare secrets, codes, MACs or signatures.** (The first file covers this in its timing-attack section.)

### 0.9 Certificates, Certificate Authorities and PKI

A public key alone is just a number. How do you know *whose* it is? A **certificate** answers that.

- A **certificate** (format: **X.509**) is a document saying *"this public key belongs to `notes.com`"* (or to `payments-service`, or to Alice), **signed by a Certificate Authority (CA)**.
- A **CA** is an entity everyone has agreed to trust. Your browser and OS ship with a list of trusted **root CAs**.
- **Chain of trust:** root CA → signs an intermediate CA → signs the website's certificate. Your browser checks each signature up to a root it already trusts.
- **PKI (Public Key Infrastructure)** is the whole system: CAs, certificates, issuance, expiry and **revocation** (cancelling a certificate before it expires, via CRLs or OCSP).

Companies can run their **own private CA** for internal services and devices. That's what makes mTLS (§7.1) possible.

### 0.10 Origins, phishing and man-in-the-middle

- An **origin** is scheme + host + port: `https://notes.com` and `https://notes-login.evil.com` are different origins. Browsers know the true origin of every page; users often don't look carefully.
- **Phishing:** tricking a user into entering credentials on a fake site.
- **Real-time phishing / Adversary-in-the-Middle (AiTM):** the fake site acts as a live proxy to the real one. It forwards your password *and* your MFA code to the real site within seconds, then steals the resulting session cookie. Kits like Evilginx automate this.
- **Phishing-resistant** methods are those where the **browser or device, not the human**, checks the origin, so a fake site gets nothing usable. This is the property that makes passkeys special (§5.5).

---

## 1. The map: three layers of authentication

### 1.1 Why layers?

The deep-dive's first move is to stop you comparing apples to oranges. "Authentication methods" is a loose phrase covering three very different kinds of thing:

**Layer 1: Credential. *What* proves it's you?**
The raw evidence: a password, a code, a private key, a fingerprint. This is the only layer that actually contains "proof".

**Layer 2: Protocol. *How* does that proof travel, and in what format?**
A credential is useless unless there's an agreed way to deliver it. A password can travel inside an HTML form POST, inside an HTTP `Authorization: Basic` header, or via an LDAP bind. A signature can travel inside WebAuthn JSON or a TLS handshake. SAML and OIDC are protocols too: they carry the *result* of an authentication ("this is Alice") from one system to another.

**Layer 3: Architecture. *How many systems* trust one login, and under what policy?**
Decisions about the overall system: "one login for 50 apps" (SSO), "require two factors" (MFA), "ask again before deleting the repo" (step-up), "trust nothing because of network location" (zero trust).

### 1.2 The Slack example, step by step

*"Sign in to your company's Slack"*:

1. You open Slack. Slack doesn't check any password itself. It's configured to **trust your company's Identity Provider** (Okta). That decision is **Layer 3 (SSO architecture)**.
2. Slack redirects you to Okta. When Okta is done, it needs to tell Slack "this is alice@company.com" in a format Slack understands, securely. It uses **SAML or OIDC**: **Layer 2 (protocol)**.
3. Okta itself must decide whether you really are Alice. It asks for a **passkey**; your Mac asks for **Touch ID** to unlock that passkey. That's **Layer 1 (credential)**.

So "SSO vs Face ID" is a meaningless comparison: SSO *uses* some credential (possibly Face ID-unlocked) at its centre.

### 1.3 Things often confused with authentication

**OAuth 2.0.** Designed for **delegated authorization**: "let this calendar app read my Google Calendar" without giving it your password. The calendar app gets an **access token** that says what it may do, but OAuth itself doesn't define a standard way to say *who the user is*. Apps that misused raw OAuth access tokens as proof of identity created real vulnerabilities. **OpenID Connect (OIDC)** fixed that by adding an **ID token** (a signed JWT stating who the user is). So: OAuth = authorization, OIDC = authentication built on top.

**Face ID / Touch ID.** It feels like "logging in with your face", but your face is checked **on your phone** and never sent anywhere. What the server receives is a cryptographic signature produced by a key that Face ID unlocked. §6 explains this in depth.

**SSO.** An **architecture or experience** ("log in once, use many apps"), not a protocol. It's implemented *using* protocols like SAML, OIDC or Kerberos. Asking "should I use SSO or SAML?" is like asking "should I use online shopping or HTTPS?".

**TLS / HTTPS.** The padlock means the **server** proved its identity to **you** (via its certificate), and the connection is encrypted. It says nothing about who *you* are; you still need to log in. **Mutual TLS** (§7.1) is the variant where the client also presents a certificate.

**CAPTCHA.** Proves you're probably **a human** rather than a bot. It doesn't identify *which* human, so it's a bot-defence tool that sits *next to* authentication (e.g. on login forms, to slow down automated guessing).

**Encryption.** Keeps data secret. But if Mallory encrypts a message with a key that Bob uses too, Bob can decrypt it without learning who sent it. Authentication needs **signatures or MACs** (§0.6), or a protocol that ties keys to identities.

---

## 2. Password-based methods

### 2.1 HTTP Basic authentication

**What it is:** a scheme built into the HTTP standard itself (RFC 7617), from the early web. No login page, no session, no cookies: the client attaches the username and password to **every single request**.

**The request, piece by piece:**

```
GET /admin/metrics HTTP/1.1
Authorization: Basic YWxpY2U6czNjcjN0
```

- `Authorization` is the standard header for sending credentials.
- `Basic` names the scheme.
- `YWxpY2U6czNjcjN0` is **base64** of the string `alice:s3cr3t` (username, a colon, password). Decode it in any terminal with `echo YWxpY2U6czNjcjN0 | base64 -d`. That's why it's **not encryption**: base64 just makes arbitrary bytes safe to put in a header.

**The challenge:**

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="admin area"
```

- `401 Unauthorized` (which really means "unauthenticated") tells the client "you need to prove who you are".
- `WWW-Authenticate` says **how**: using the Basic scheme.
- `realm` is a label for a protected area. Browsers show it in the popup and remember credentials per realm, so a site can have several areas with different logins.

When a browser sees this response, it shows its **native username/password dialog**. After you enter credentials, the browser **caches them and re-sends them automatically** on every request to that realm.

**Each pro and con explained:**

- *Built into every client:* `curl -u alice:s3cr3t https://…` just works. No login endpoint to write.
- *Plaintext without HTTPS:* anyone on the network path (public Wi-Fi, a compromised router) can read and decode the header. **With HTTPS**, the whole request including headers is encrypted, which is why Basic is acceptable only over HTTPS.
- *Password on every request:* each request is another chance for it to end up in a log file, a proxy, an error tracker, or a debugging dump. Compare with sessions, where the password is sent **once** and later requests carry a revocable session ID.
- *Expensive to verify:* if passwords are stored with a slow hash (as they should be, §0.3), the server must run that slow hash on **every request**. Servers often add a cache to compensate.
- *No logout:* the browser keeps re-sending the cached credentials until it's closed. There's no standard "forget these" command.
- *No MFA, ugly UI:* the native popup can't ask for a second factor or show your branding.

**When it's fine:** an internal metrics page, a staging site you want to hide from search engines, a simple script calling an internal service. All over HTTPS.

### 2.2 HTTP Digest authentication

**The idea:** Basic sends the password. Digest instead proves you know the password by sending a **hash** that includes a fresh **nonce** from the server (challenge–response, §0.7), so a recorded request can't be replayed.

**How it works (simplified):**

1. Server: `401` with `WWW-Authenticate: Digest realm="admin", nonce="a8f2…"`.
2. Client computes:
   - `HA1 = MD5(username:realm:password)`
   - `HA2 = MD5(method:uri)`, e.g. `MD5(GET:/admin)`
   - `response = MD5(HA1:nonce:…:HA2)`
3. Client sends `response`. The server computes the same thing and compares.

**Why it failed:**

- **MD5 is broken** as a cryptographic hash, and Digest was designed around it (a later revision allows SHA-256, but support is patchy).
- **The server must store `HA1`**, because it needs `HA1` to compute the expected response. But look at step 2: the client only needs `HA1`, not the password, to produce a valid response. So **`HA1` is "password-equivalent"**: anyone who steals the `HA1` table can log in as every user without cracking anything.
- It can't use salted slow hashes like bcrypt, because the formula fixes exactly which hash is stored.

HTTPS solved Digest's main goal (not exposing the password on the wire) more cleanly, so the world moved to **HTTPS + Basic** or **HTTPS + login form + session**.

### 2.3 Modern password rules (NIST)

**NIST** (the US National Institute of Standards and Technology) publishes **SP 800-63B**, the most widely cited guideline for authentication. Its modern guidance reversed many rules that companies enforced for decades. Each row of the deep-dive's table, explained:

**"Don't impose composition rules."**
Rules like "must contain uppercase, number and symbol" were meant to increase randomness. In practice humans satisfy them predictably: capital first letter, `1` or the current year at the end, `!` as the symbol. `Password1!` passes every rule and is in every attacker's wordlist. The rules annoy users without adding much real unpredictability.

**"Length matters most."**
Every extra character multiplies the possibilities. Compare:
- 8 characters from all 94 printable symbols: 94⁸ ≈ 6 × 10¹⁵ combinations.
- 15 lowercase letters only: 26¹⁵ ≈ 1.7 × 10²¹ combinations, about **275,000 times more**.

Long passphrases (`correct horse battery staple`-style) are also easier to remember. NIST's latest revision asks for at least **15 characters** when the password is the only factor (8 when it's combined with MFA), and asks systems to accept **at least 64**, so people can use passphrases and password-manager output. Very low maximums (like "max 16 characters") are a red flag: they often mean the site stores passwords in a fixed-size column, possibly unhashed.

**"Don't force periodic rotation."**
Forced 90-day changes don't make passwords stronger; they make people choose *predictable sequences*: `Summer2026!` → `Autumn2026!`. An attacker who learns one can guess the next. Change a password only when there's **evidence it's compromised** (found in a breach, suspicious logins).

**"Allow paste and password managers."**
Blocking paste breaks password managers, pushing users towards short, memorable, reused passwords. Password managers generate long random unique passwords: exactly what you want.

**"Don't use security questions."**
"Mother's maiden name", "first pet", "school" are either public (social media, public records) or guessable from a small set of answers. They're effectively a weak second password that's often used for **account recovery**, the weakest path (§12.5).

**"Check against breached-password lists."**
Billions of real passwords have leaked over the years. Attackers try those first. If a user picks a password that's already in a breach list, reject it with a helpful message, at registration and when it's changed.

#### Breached-password checks without leaking the password (k-anonymity)

The problem: you want to ask a service "has this password leaked?" without sending it the password. *Have I Been Pwned*'s solution:

1. Compute `SHA-1(password)`. For `password` that's `5BAA61E4C9B93F3F0682250B6CF8331B7EE68FD8`.
2. Send only the **first 5 hex characters**: `GET https://api.pwnedpasswords.com/range/5BAA6`.
3. The API returns every leaked hash **suffix** that starts with that prefix, each followed by how many times it appeared in breaches, e.g. `1E4C9B93F3F0682250B6CF8331B7EE68FD8:<count>` plus hundreds of other lines.
4. Your server checks locally whether your remaining 35 characters are in the list.

**Why this is private:** 5 hex characters have 16⁵ ≈ 1 million possible values, and each prefix is shared by hundreds of leaked passwords (and countless non-leaked ones). The API can't tell which of them you were checking. That's **k-anonymity**: your query is hidden among at least *k* others.

*Why SHA-1 is fine here:* it's being used as a lookup key into a public dataset, not to protect stored passwords. Its weaknesses don't matter for this use.

### 2.4 Password-authenticated key exchange (PAKE)

**The problem:** in a normal login, your password arrives at the server (inside HTTPS) and exists in the server's memory for a moment before being hashed. A compromised server, a logging bug, or a misconfigured proxy could capture it.

**The PAKE idea:** run a cryptographic conversation in which **both sides prove they know matching password-derived values without ever transmitting the password**. At the end, they share a session key.

- **SRP (Secure Remote Password):** at signup the client computes a **verifier** from the password and a salt and sends only that. At login, client and server exchange random values; a correct password lets the client derive the same session key as the server. 1Password uses SRP.
- **OPAQUE:** a newer design that also hides the salt from attackers, so they can't even precompute guesses before stealing the server's database.

You rarely implement this in typical web apps (HTTPS + slow hashing is the norm), but it's the "password done perfectly" end of the spectrum.

### 2.5 The main threats to passwords

**Brute force:** try many passwords against **one** account (`aaaa`, `aaab`, …, or a wordlist).
*Defences:* **rate limiting** (max N attempts per minute), **exponential backoff** (each failure doubles the wait), slow hashing. Be careful with hard **lockouts** ("locked after 5 failures"): attackers can use them to deliberately lock out real users, so backoff or CAPTCHA is often better than a full lock.

**Credential stuffing:** take username/password pairs leaked from **another** site and try them on yours. It works because people reuse passwords. Often run from botnets with millions of IPs, so per-IP limits alone don't stop it.
*Defences:* breached-password checks (§2.3), MFA, bot detection, alerting on spikes of failed logins.

**Password spraying:** the reverse of brute force: try **one** very common password (`Winter2026!`) against **many** accounts. Each account sees only one or two failures, so per-account lockout never triggers.
*Defences:* rate limits **per IP and globally**, banning common passwords, monitoring for "many accounts, same failure pattern".

**Phishing:** a fake login page collects the password. MFA helps, but code-based MFA can be relayed in real time (§0.10). Passkeys stop it outright (§5.5).

**Database breach:** the attacker gets your users table. If passwords are stored with salted Argon2id or bcrypt, cracking strong passwords is infeasible; weak ones still fall, which is another reason for breached-password checks.

---

## 3. One-time passwords (OTP)

An **OTP** is a code that works **once** or for a **short time**. Because it expires, stealing it is far less valuable than stealing a password. OTPs are usually the **second factor** ("something you have": the phone that receives or generates the code), and sometimes the only factor in passwordless logins.

### 3.1 HOTP and TOTP: authenticator apps

#### The big idea

Server and app share one secret. Each independently computes `code = f(secret, moving factor)`. Nothing is sent between them at login time; they agree because they run the same function on the same inputs. The **moving factor** makes the code change:
- **HOTP** (HMAC-based OTP, RFC 4226): the moving factor is a **counter** that goes up by one each time a code is used.
- **TOTP** (Time-based OTP, RFC 6238): the moving factor is **the current time**, divided into 30-second steps.

#### Setup, step by step

1. **The server generates a random shared secret**, e.g. 20 random bytes from a CSPRNG.
2. **It shows a QR code.** The QR code just encodes a URI:
   ```
   otpauth://totp/Notes:alice@example.com?secret=JBSWY3DPEHPK3PXP&issuer=Notes
   ```
   - `otpauth://` tells the phone "this is for an authenticator app".
   - `totp` says time-based (the alternative is `hotp`).
   - `Notes:alice@example.com` is the **label** the app displays, so you know which account this code is for.
   - `secret=JBSWY3DPEHPK3PXP` is the shared secret in **base32** (letters A–Z and digits 2–7: easy to type by hand if the QR code can't be scanned). This example decodes to the bytes `Hello!` followed by `0xDEADBEEF`, a classic test value. Real secrets are random.
   - `issuer=Notes` is the service name. Optional parameters (`algorithm=SHA1`, `digits=6`, `period=30`) default to these values.
3. **The app scans and stores the secret.** From now on, server and app both know it, and it never travels again.
4. **The user types one code to confirm.** This proves the app scanned the right secret and the clocks agree. Only now does the server switch 2FA on. Without this step, a bad scan would lock the user out.

#### Every login

The app shows `code = TOTP(secret, now)`. The user types it. The server computes `TOTP(secret, now)` itself and compares.

#### HOTP vs TOTP

- **HOTP's weakness (counter drift):** if you press the button on a hardware token five times without logging in, the token's counter is 5 ahead of the server's. Servers handle this with a **look-ahead window**: accept any of the next ~10 counter values and then resynchronise.
- **TOTP's weakness (clock drift):** the phone and server clocks must roughly agree. Servers accept the previous and next 30-second step as well (±1 step), giving about 90 seconds of tolerance. Phones sync their clocks automatically, so this is rarely a problem.
- TOTP won because codes **expire on their own**: a code that leaks is useless after a minute, whereas an unused HOTP code stays valid until a later one is used.

#### The code, line by line

```js
function hotp(secret, counter, digits = 6) {
  const msg = Buffer.alloc(8);
  msg.writeBigUInt64BE(BigInt(counter));
```
The counter becomes an **8-byte big-endian integer**, exactly as RFC 4226 specifies. ("Big-endian" means the most significant byte comes first.) Both sides must encode it identically, or they'd compute different HMACs.

```js
  const hmac = crypto.createHmac("sha1", secret).update(msg).digest(); // 20 bytes
```
Compute `HMAC-SHA1(secret, counter)` (§0.6). The output is 20 bytes. SHA-1 is weak for some uses, but as part of HMAC it's still considered safe, and it's what nearly every authenticator app expects by default.

```js
  const offset = hmac[hmac.length - 1] & 0x0f;                         // "dynamic truncation"
  const code = (hmac.readUInt32BE(offset) & 0x7fffffff) % 10 ** digits;
```
This is **dynamic truncation**: turning 20 bytes into a 6-digit number.
- Take the **last byte** and keep its low 4 bits (`& 0x0f`). That gives a number 0–15: the **offset**.
- Read **4 bytes starting at that offset** as a 32-bit integer. (The offset is at most 15, and 15 + 4 = 19, which is still inside the 20 bytes.)
- `& 0x7fffffff` clears the top bit, so the number is always treated as positive (avoids signed-vs-unsigned differences between languages).
- `% 10 ** digits` keeps the last 6 decimal digits.

Why pick the offset from the hash itself? So that *which* 4 bytes are used is unpredictable too, spreading the result evenly.

```js
  return code.toString().padStart(digits, "0");
}
```
Pad with leading zeros: the code `42` must be shown as `000042`.

**Worked example** (the official RFC 4226 test vector, secret = the ASCII bytes `12345678901234567890`, counter 0):
- HMAC = `cc93cf18508d94934c64b65d8ba7667fb7cde4b0`
- Last byte `b0` → low 4 bits = `0` → offset 0
- 4 bytes at offset 0: `cc93cf18`; after `& 0x7fffffff`: `4c93cf18` = 1,284,755,224
- `% 1,000,000` = **755224**

```js
const totp = (secret, timeMs = Date.now()) => hotp(secret, Math.floor(timeMs / 1000 / 30));
```
TOTP is literally HOTP with **counter = number of 30-second periods since 1 January 1970**. At Unix time 1,728,380,000 s, the counter is `floor(1728380000 / 30) = 57,612,666`. Everyone in the world computing a TOTP right now uses the same counter; only the secret differs.

```js
function verifyTotp(secret, input) {
  if (!/^\d{6}$/.test(input)) return false;
```
Reject anything that isn't exactly six digits. Besides basic validation, this guarantees both buffers below have the same length, which `timingSafeEqual` requires (it throws otherwise).

```js
  return [-1, 0, 1].some((w) =>
    crypto.timingSafeEqual(Buffer.from(totp(secret, Date.now() + w * 30_000)), Buffer.from(input))
  );
}
```
Try the **previous, current and next** 30-second window (`w * 30_000` ms) to tolerate clock drift and slow typists. Compare in **constant time** (§0.8). `.some()` returns true if any window matches.

#### Server-side gotchas, explained

- **The secret can't be hashed.** For passwords, the server only needs to *check* a value, so it stores a one-way hash. For TOTP, the server must *compute HMAC(secret, time)*, which needs the actual secret. So **encrypt it at rest**, ideally with a key held in a **KMS** (Key Management Service, e.g. AWS KMS) so a database dump alone doesn't reveal secrets.
- **Block replay.** A code stays valid for its whole window (up to ~90 s with ±1 tolerance). An attacker watching over your shoulder, or a phishing proxy, could reuse it within that time. Store the **last accepted time-step** per user and reject any code from that step or earlier.
- **Rate-limit.** With ±1 window, three codes are valid at any moment, so each random guess has a 3-in-1,000,000 chance. Unlimited guessing would succeed after ~333,000 tries on average, which is trivial for a script. Allow e.g. 5 attempts, then slow down or lock the MFA step.
- **Recovery codes.** If the phone is lost, the user is locked out. At setup, show ~10 random single-use codes to print or save. Store them **hashed** (the server only needs to check them, not compute anything from them), and mark each as used once consumed.

### 3.2 SMS and email OTP

**How it works:** the server generates a random 6-digit code, stores `hash(code)` with an expiry (5–10 minutes) and an attempt counter, and sends the code by SMS or email. The user types it; the server hashes, compares, checks expiry, and deletes it.

(Because a 6-digit code only has a million values, its hash could be brute-forced if the DB leaked. That's acceptable since it expires in minutes, but it's why the **attempt limit** matters most: invalidate the code after ~5 wrong tries.)

**Each SMS weakness explained:**

- **SIM swapping:** the attacker calls your mobile carrier pretending to be you ("I lost my phone, please move my number to this new SIM"), often using personal details found online, or bribes a store employee. Your phone loses signal; theirs now receives your calls and SMS, including every login code. Crypto investors and public figures have been repeatedly targeted this way.
- **SS7 interception:** SS7 is the decades-old signalling network that phone carriers use to route calls and texts between each other. It was built for a small club of trusted operators and has weak authentication, so attackers with access to it have been able to redirect SMS messages.
- **Phishing:** a fake site asks for the code and forwards it to the real site immediately. The code doesn't care where it was typed.
- **Cost and delivery:** each SMS costs money (and "SMS pumping" fraud abuses this by triggering expensive messages to premium numbers). Messages get delayed or lost, especially internationally.

**NIST "restricted"** means: you may use it, but you must offer users an alternative and assess the risk. The balanced takeaway: **SMS 2FA blocks the vast majority of mass, automated attacks** (credential stuffing), so it's far better than nothing. It's weak against *targeted* attackers. **Email OTP** inherits the security of the user's email account, which may itself be protected only by a password.

### 3.3 Push-notification approval

**How it works:** after you enter your password, the IdP sends a push notification to an app registered on your phone. You tap *Approve*; the app (holding a key registered at setup) sends a signed approval back.

**MFA fatigue / push bombing:** the attacker already has your password (from phishing or a leak). They hit "log in" again and again, generating dozens of prompts, often late at night. The victim, annoyed or confused, eventually taps *Approve* just to make it stop. In the **2022 Uber breach**, the attacker spammed an employee with push requests and then messaged them, pretending to be IT support, saying to accept the prompt.

**Number matching:** the login screen shows a number (`47`) and the phone asks you to **type it**. You can't type the right number unless you're looking at the login screen that started this request. A random 2 a.m. prompt is now unanswerable. Adding **context** (the app name, the approximate location of the login: "Sign-in attempt from Moscow?") lets users spot fraud. Microsoft made number matching mandatory for its Authenticator push in 2023.

### 3.4 Hardware OTP tokens

Key-fob devices with a small screen showing a code that changes every 30–60 seconds, computed from a secret (the **seed**) burned in at the factory. The server holds a copy of each token's seed.

Weaknesses: they cost money, need physical distribution and replacement, and the vendor-held seeds are a concentrated target: in 2011, attackers breached RSA and stole SecurID seed-related data, which was then used in attacks on its customers. They're being replaced by authenticator apps and FIDO keys.

### The key limitation of every OTP type

Every OTP is a value **the human reads and types**. The human can be fooled about *where* they're typing it. A real-time phishing proxy (§0.10) collects the password and code and logs in to the real site within seconds, well inside the validity window. That's why OTPs are **not phishing-resistant**. Only methods where a machine checks the website's origin (passkeys, security keys, client certificates) stop this.

---

## 4. Magic links

### 4.1 The flow, step by step

**Step 1: user submits their email.** The server:
- generates a **cryptographically random token** of 32+ bytes (§0.4), e.g. `k3J9…` once base64url-encoded,
- stores `hash(token)`, the user ID, an expiry 10–15 minutes ahead, and `used = false`.

**Step 2: server emails a link** containing the **raw** token: `https://notes.com/auth/verify?token=k3J9…`.

**Step 3: user clicks the link.** The server hashes the token from the URL and looks up the row. It checks: does the row exist? Not expired? Not already used? If all pass, it marks the row used, **creates a normal session** (cookie), and redirects into the app.

The magic link is just a **login method**; after it, everything works with the same sessions or JWTs as the first file describes.

### 4.2 Each implementation rule, explained

**Random, hashed, single-use, short expiry.**
- *Random:* if tokens were predictable (sequential IDs, timestamps), attackers could guess valid links.
- *Hashed:* if your database leaks, an attacker shouldn't be able to turn stored rows into working login links. A plain SHA-256 is enough because the token is high-entropy (§0.3).
- *Single-use:* links end up in browser history, email archives, forwarded messages. Once used, they should be worthless.
- *Short expiry:* limits how long a link sitting in an inbox can be abused.

**Email security scanners.** Corporate mail systems (Microsoft Defender "Safe Links", Proofpoint, Mimecast…) **open every link in incoming email** in a sandbox to check for malware, often before the user sees the email. A `GET` that consumes the token would let the scanner "use up" the link, and the user's click then fails with "link expired".
**Fix:** the `GET` only shows a page with a **"Log in" button**. Clicking it sends a `POST`, which is what consumes the token. Scanners follow links but don't press buttons. (This also follows the HTTP rule that `GET` should not change server state.)

**Bind the link to the browser that requested it.**
At step 1, set a cookie in the requesting browser containing a random value tied to the token. At step 3, require that cookie. Why:
- *Confusion:* you request a link on your laptop, open the email on your phone, and the **phone** gets logged in, not the laptop.
- *Login CSRF attack:* an attacker requests a magic link for **their own** account and sends it to you ("Check out this doc!"). You click and are now silently logged into the **attacker's** account. Anything you type (notes, payment details) goes into an account the attacker controls. With browser binding, the link only works in the browser that requested it, i.e. the attacker's.

Many apps handle the cross-device case by showing "This link was requested from a different browser. Open it there, or enter this code instead."

**Anti-enumeration.** If the response says "No account with that email", attackers can test which emails are registered (useful for targeted phishing, or embarrassing for sensitive services). Always reply: *"If an account exists for that email, we've sent a link."*

### 4.3 Pros and cons, explained

- *No password to remember, leak or reuse:* there's nothing for credential stuffing to attack.
- *Simple to build:* a random token, a table, and an email sender.
- *Doubles as email verification:* clicking proves the user controls the address.
- *Security = email security:* anyone who controls the inbox controls the account. That's equally true of "forgot password" flows, so magic links don't make things worse than a password app with email reset.
- *Friction:* switching to the inbox, waiting for delivery, digging through spam. Fine for apps you log into once a month; annoying for daily use.
- *Phishable in some setups:* attackers can send look-alike emails, and in the "approve this login" variant they can trick you into approving their session.

**Email OTP codes** ("Your code is 381 552") are the same mechanism with a short code instead of a link: the user types it into the tab they're already in. Better on mobile, and naturally bound to the browser that's waiting for it.

---

## 5. Passkeys, WebAuthn and FIDO2

### 5.1 The names, untangled

There are many names because there are several **pieces**, standardised by different groups:

- **FIDO Alliance:** an industry consortium (Apple, Google, Microsoft, Amazon, banks…) founded to "reduce reliance on passwords". It writes the specifications.
- **WebAuthn (Web Authentication):** the **JavaScript API in browsers** (a W3C standard). Websites call `navigator.credentials.create()` to register and `navigator.credentials.get()` to log in. This is the part *you* write code against.
- **CTAP2 (Client to Authenticator Protocol):** how the **browser/OS talks to the authenticator**, e.g. over USB, NFC or Bluetooth to a YubiKey or a phone. You never touch this; the browser handles it.
- **FIDO2:** the umbrella name for WebAuthn + CTAP2.
- **Passkey:** the friendly product name, adopted by Apple, Google and Microsoft in 2022, for a FIDO2 credential, typically one that **syncs** across your devices.
- **Authenticator:** the thing that **holds the private key and signs**.
  - **Platform authenticator:** built into the device you're using: iPhone/Mac (Secure Enclave, unlocked by Face ID/Touch ID), Windows Hello (TPM), Android.
  - **Roaming authenticator:** a separate device you bring along: a YubiKey over USB/NFC, or your phone helping your laptop.
- **Relying Party (RP):** the website that relies on the authentication (`notes.com`). Its **RP ID** is its domain.

### 5.2 The core idea

A password is a **shared secret**: you know it, and the server stores something derived from it. Both sides can leak it.

A passkey replaces that with a **key pair** (§0.5), **generated separately for each website**:

- The **private key** never leaves your authenticator (or is synced only in end-to-end encrypted form between your own devices). There's **nothing to type**, so nothing to phish, and nothing for you to reuse across sites.
- The server stores **only the public key**. A leaked database of public keys is worthless to an attacker: public keys can verify signatures but not create them.
- **Login = challenge–response (§0.7):** server sends a random challenge; your device signs it (after you approve with Face ID, fingerprint or PIN); server checks the signature with your public key.

Because each site gets its own key pair, sites can't correlate you by key either.

### 5.3 Registration, step by step

1. **User clicks "Create passkey"** (usually while already logged in, or during signup).
2. **Server generates a random challenge** (e.g. 32 bytes), stores it in the user's session so it can check it later, and returns **options**:
   - `challenge`: the random bytes.
   - `rp.id = "notes.com"`: which site this credential will belong to.
   - `user.id`: a stable, random **user handle** (not the email; emails change and leak personal data).
   - `pubKeyCredParams`: which signature algorithms the server accepts.
3. **Browser calls `navigator.credentials.create(options)`.** It checks that `rp.id` matches the page's actual domain (a page on `evil.com` can't create credentials for `notes.com`).
4. **The OS prompts the user** (Face ID / Touch ID / PIN / touch the security key). This is **user verification**.
5. **The authenticator creates a brand-new key pair** just for `notes.com` and this user.
6. **It returns** a `credentialId` (an identifier for this key), the **public key**, and signed data containing the challenge and the origin. Optionally it includes **attestation** (proof of the authenticator's make and model, §5.7).
7. **Server verifies:** does the challenge match the one stored in step 2? Is the origin exactly `https://notes.com`? Then it stores `userId, credentialId, publicKey, signCount`.

### 5.4 Authentication, step by step

1. **User clicks "Sign in".** Server creates a **new** random challenge and returns it with `rpId`.
2. **Browser calls `navigator.credentials.get()`.** The OS shows the passkeys it has **for `notes.com` only**.
3. **User approves** with Face ID. The authenticator signs: **`authenticatorData ‖ SHA-256(clientDataJSON)`** (‖ means "concatenated with").
4. **Browser sends** `credentialId`, `signature`, `authenticatorData`, `clientDataJSON` to the server.
5. **Server checks, in order:**
   1. Look up the stored public key by `credentialId` (and check it belongs to the expected user).
   2. Check `clientDataJSON`: correct `type` (`webauthn.get`), the **challenge** equals the one it issued, the **origin** is exactly `https://notes.com`. Check `authenticatorData`'s **rpIdHash** equals `SHA-256("notes.com")`.
   3. Check the **flags**: UP (user present) set; UV (user verified) set if you require it.
   4. **Verify the signature** over `authenticatorData ‖ SHA-256(clientDataJSON)` using the public key.
   5. Delete the used challenge, create a session.

**The two signed structures:**

`clientDataJSON` is created **by the browser**, not by the website's JavaScript. It looks like:
```json
{"type":"webauthn.get","challenge":"q3Jk…","origin":"https://notes.com","crossOrigin":false}
```
Because the browser writes the origin, a malicious page **cannot lie about it**.

`authenticatorData` is created **by the authenticator**, a compact binary structure:
- **rpIdHash** (32 bytes): SHA-256 of the RP ID the key belongs to.
- **flags** (1 byte): bits for **UP** (user present), **UV** (user verified), **BE/BS** (backup eligible / backed up, i.e. is this a synced passkey?), and whether extra data follows.
- **signCount** (4 bytes): the signature counter (§5.7).
- During registration only: the new **credential ID and public key**.

### 5.5 The browser code, explained

```js
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: fromBase64url(opts.challenge),            // random bytes from the server
```
WebAuthn works with raw bytes (`ArrayBuffer`), but JSON can't carry raw bytes, so servers send them as **base64url** text (base64 with URL-safe characters) and the page converts them.

```js
    rp: { id: "notes.com", name: "Notes" },
```
`id` binds the credential to the domain. `name` is just a display label.

```js
    user: { id: fromBase64url(opts.userHandle), name: "alice@example.com", displayName: "Alice" },
```
`id` is the opaque user handle. `name` and `displayName` are shown in the OS passkey picker so the user can tell accounts apart.

```js
    pubKeyCredParams: [{ type: "public-key", alg: -7 },   // ES256
                       { type: "public-key", alg: -257 }], // RS256
```
Accepted algorithms, in preference order, using **COSE** numbers (COSE is a compact binary standard for crypto objects): `-7` = **ES256** (ECDSA on curve P-256 with SHA-256, which almost all authenticators support), `-257` = **RS256** (RSA with SHA-256, needed by some Windows Hello setups). `-8` (EdDSA) is also common.

```js
    authenticatorSelection: { residentKey: "required", userVerification: "required" },
```
- `residentKey: "required"` asks for a **discoverable credential**: the authenticator stores the user handle together with the key, so at login it can offer "Sign in as alice@example.com" **without the user typing a username first**. This is what makes it a true passkey experience.
- `userVerification: "required"`: the authenticator must check *who* (biometric/PIN), not merely that someone touched it.

```js
const assertion = await navigator.credentials.get({
  publicKey: { challenge: fromBase64url(opts.challenge), rpId: "notes.com", userVerification: "required" },
});
```
Login: a fresh challenge, the RP ID, and the same UV requirement. No `allowCredentials` list is given, so the OS offers any discoverable passkey for `notes.com`.

**Why "use a library" on the server:** the authenticator's data is encoded in **CBOR** (a binary JSON-like format), public keys are in **COSE** format, signatures may be in **DER** encoding, and there are dozens of fields and flags to check in the right order. Missing one check (e.g. forgetting the origin check) silently destroys the security. SimpleWebAuthn, go-webauthn, py_webauthn and webauthn4j encode all of this correctly.

### 5.6 Why passkeys are phishing-resistant

Walk through an attack on each method. An attacker builds `https://notes-login.evil.com`, a pixel-perfect copy of the Notes login page, and sends you a link.

- **Password:** you type it. The attacker has it. ❌
- **Password + TOTP:** you type both. The attacker's server relays them to the real site within seconds. ❌
- **Passkey:**
  - Your browser asks the authenticator for passkeys for the current site, `notes-login.evil.com`. **You don't have one**, so there's nothing to select, nothing to sign, nothing to type.
  - Suppose the attacker's page tries to request a credential for `rpId: "notes.com"`: the browser refuses, because the page's origin isn't `notes.com` or a subdomain of it.
  - Even in a hypothetical where a signature was produced, the browser writes `"origin":"https://notes-login.evil.com"` into the signed `clientDataJSON`, and the real server rejects it. ✅

The deep principle: **the decision "is this the real site?" moves from a tired human to the browser**, which never gets it wrong.

### 5.7 Synced vs device-bound passkeys

**Synced passkeys** live in a credential manager (iCloud Keychain, Google Password Manager, 1Password, Bitwarden…). They're copied to all your devices, **end-to-end encrypted**, meaning the sync provider can't read them.
- Pro: lose your phone and your passkeys are still on your laptop, or restorable when you sign into a new phone.
- Con: their security depends partly on your Apple/Google account's security.
- The BE/BS flags in `authenticatorData` tell the server whether a credential is synced.

**Device-bound passkeys** are generated inside a piece of hardware (a YubiKey's secure element, a laptop's TPM) and **cannot be exported**, even by you.
- Pro: the highest assurance. Possession of *that physical object* is proven.
- Con: lose it and the credential is gone. Users need a **second key** registered as a backup.
- Used for admins, high-risk employees and regulated industries.

**Cross-device sign-in ("hybrid"):** you're on a library computer without your passkey. Choose *"Use a phone or tablet"*:
1. The computer shows a QR code (it contains a one-time key for this session).
2. Your phone scans it and connects through an encrypted relay.
3. The phone and computer also do a **Bluetooth proximity check**: the computer must detect a Bluetooth signal from your phone. This proves the phone is physically near the computer, so an attacker can't email you a QR code from across the world and have you approve their login.
4. Your phone asks for Face ID and signs the challenge.

**Conditional UI:** the website calls `navigator.credentials.get({ mediation: "conditional", … })` and marks its username field with `autocomplete="username webauthn"`. When you click the field, the browser's **autofill dropdown** lists your passkeys alongside saved passwords. Users pick one without needing to understand what a passkey is. This is how passkeys sneak into existing login pages gently.

### 5.8 Other details, explained

**UP vs UV.**
- **User Presence (UP):** someone physically interacted (tapped the key, clicked Allow). Stops malware from silently using a plugged-in key, but doesn't say *who*.
- **User Verification (UV):** the authenticator checked **who** via biometric or PIN.
- If the passkey **replaces** the password entirely, require UV, or a thief holding your unlocked phone or YubiKey could log in.

**A passkey with UV is MFA in one step.** You need the device (**have**), and the device demands your face or PIN (**are** / **know**) before signing. One gesture, two factors.

**Signature counter.** Device-bound authenticators increment a counter every time they sign. The server stores the last value. If a new login arrives with a counter **equal or lower** than stored, two copies of the same key may exist, i.e. the key may have been **cloned**. Synced passkeys exist on many devices by design, so they usually report `0` and the counter means nothing.

**Attestation.** During registration, the authenticator can include a certificate chain proving "I'm a genuine YubiKey 5, model X". An enterprise can then enforce "only company-issued keys". Consumer apps usually request `attestation: "none"`: they don't care which device you use, and attestation adds privacy and complexity costs.

**Account recovery** is the hard part. If every passkey is lost, how does the user get back in? Let users register **multiple passkeys** (phone + laptop + security key), rely on synced passkeys for consumers, and keep a well-protected fallback (§12.5). Recovery is where attackers aim once the front door is strong.

---

## 6. Biometrics and Apple Face ID

### 6.1 Your face never goes to the server

The diagram, step by step:

1. **Your face → TrueDepth camera.** The phone captures depth and infrared data of your face.
2. **→ Secure Enclave compares with the enrolled template.** The comparison happens inside isolated hardware on your phone (§6.3).
3. **Match? → unlocks a private key** stored in the Secure Enclave. The biometric check is a **gate** guarding the key.
4. **The key signs the server's challenge.** Exactly the passkey flow from §5.
5. **The server verifies the signature with the public key.**

**What the server learns:** only that a valid signature was produced by a key registered to this user, plus a single **UV flag** meaning "the authenticator verified the user somehow". It cannot tell whether that was Face ID, Touch ID or a device PIN.

**"Face ID is a gate on something you have."** From the server's point of view, the proof is **possession of the device's private key** (have). Face ID is the device's way of making sure only *you* can use that possession. That's why the deep-dive says biometrics are a *local unlock mechanism*.

**Why it's designed this way:**
- **You can change a password after a breach. You can't change your face.** A leaked face template is leaked for life.
- A central server storing millions of face templates would be one of the most valuable breach targets in the world, and every breach would be permanent.
- Keeping each template **on its own device, in tamper-resistant hardware**, means there's no central honeypot. To steal your template, an attacker needs your physical phone *and* a way into the Secure Enclave.

### 6.2 How Face ID works

**1. Sensing (the TrueDepth camera system)**, three components at the top of the phone:
- **Flood illuminator:** shines invisible **infrared** light on your face, so the camera can see you in total darkness.
- **Dot projector:** projects **30,000+ infrared dots** in a known pattern.
- **Infrared camera:** photographs where the dots land. On a curved, 3D face, the dots shift by different amounts depending on depth (nose closer, ears further). From this distortion the phone computes a **3D depth map**, plus a 2D infrared image.

Why depth matters: a **photo or screen is flat**. Its depth map looks nothing like a face's, so it fails immediately. This is the main advantage over simple 2D camera face unlock found on some cheaper devices.

**2. Processing:** an on-device neural network (running on Apple's Neural Engine) converts the depth map and IR image into a **mathematical representation**: a list of numbers capturing your face's geometry, not a picture.

**3. Matching:** that representation is compared to your **enrolled template** inside the Secure Enclave. iOS itself never sees the template; it only receives "match" or "no match".

**4. Attention awareness:** by default, Face ID checks that your **eyes are open and directed at the screen**. Without this, someone could unlock your phone by holding it up to your face while you sleep or are unconscious.

**5. Anti-spoofing:** additional neural networks are trained specifically to reject masks, photos and screens. The depth data makes this far easier than with an ordinary camera.

**6. Adaptation:** faces change (beards grow, glasses, makeup, ageing). After a successful match, Face ID can fold the new data into the template. If Face ID nearly matches and you then enter your passcode, it also uses that as a learning signal.

**Apple's published figures, explained:**
- **False match probability** = the chance that a **random other person's** face or finger unlocks your device.
- **Face ID: below 1 in 1,000,000. Touch ID: about 1 in 50,000.** For comparison, a random 4-digit PIN is guessed with probability 1 in 10,000, and a 6-digit PIN 1 in 1,000,000.
- **Higher for twins, look-alike siblings, and children under 13:** "random person" statistics assume an unrelated face; relatives share facial structure, and children's facial features are still developing.

**Privacy and fallbacks, explained:**
- Face data **stays in the Secure Enclave**, **isn't backed up to iCloud**, **isn't sent to Apple**. There's no server copy to subpoena or steal.
- **Apps never see face data.** The API (`LocalAuthentication`) returns success/failure only.
- **The passcode is the root of trust.** The passcode is always required after a **restart**, after **5 failed Face ID attempts**, after **48 hours** without unlocking, and in a few other situations (e.g. after a remote lock or an Emergency SOS). Why? The passcode is mixed into the keys that **encrypt the phone's data**; after a restart, those keys aren't available until you type it. Face ID is a convenience that releases keys which the passcode already unlocked earlier. This also gives you a legal and physical safety valve: squeezing the side buttons forces passcode entry, so nobody can unlock your phone by pointing it at your face.

### 6.3 The Secure Enclave

**What it is:** a separate small processor inside Apple's chips (A-series, M-series), with:
- its own **secure boot** (it only runs Apple-signed firmware),
- **encrypted memory** that the main processor can't read,
- its own **hardware random number generator** and crypto engine.

**The key property:** even if attackers fully compromise iOS (a jailbreak, malware with root), they **can't extract keys or templates from the Secure Enclave**. The most they can do is *ask* it to use a key, and the Secure Enclave will only do that if its own conditions are met (e.g. "Face ID matched in the last few seconds").

**Equivalents:**
- **TPM (Trusted Platform Module):** a security chip in Windows PCs. Windows Hello stores keys there.
- **Titan M / StrongBox:** Google's security chip in Pixel phones; StrongBox is Android's API name for a hardware-isolated keystore.
- **Secure elements:** the tamper-resistant chips inside YubiKeys, smart cards, and payment cards.

### 6.4 How an app should use Face ID

#### ❌ The boolean gate

```swift
let context = LAContext()
context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                       localizedReason: "Log in to Notes") { success, error in
    if success { showNotes() }
}
```

- `LAContext` is the LocalAuthentication framework's entry point.
- `evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, …)` shows the Face ID prompt with your `localizedReason` text.
- The closure receives `success: Bool`.

**Why it's weak:** the security decision is a **single `if` on a boolean inside your app's own process**. On a jailbroken phone, tools like Frida can hook `evaluatePolicy` and make it call the closure with `success = true` immediately, without any face. Also, nothing is proven to your **server**: the server just has to trust whatever the app claims. Fine for "hide the screen from a nosy friend", not for real authentication.

#### ✅ The key gate

The idea: put the "did Face ID pass?" decision **inside the Secure Enclave**, and make the result a **cryptographic signature the server checks**. No signature → no login, and there's no boolean for an attacker to flip.

```swift
let access = SecAccessControlCreateWithFlags(nil,
    kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
    [.privateKeyUsage, .biometryCurrentSet],
    nil)!
```
Creates the **rules** for the key:
- `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`: usable only while the device is unlocked, and **never** included in backups or migrated to another device.
- `.privateKeyUsage`: required for Secure Enclave keys; it means "the private key may be used for signing".
- `.biometryCurrentSet`: using the key requires a Face ID/Touch ID match **with the faces/fingers enrolled right now**. If anyone adds or removes a face or fingerprint, the key becomes permanently unusable.

```swift
let attributes: [String: Any] = [
    kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
    kSecAttrKeySizeInBits as String: 256,
    kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
    kSecPrivateKeyAttrs as String: [kSecAttrIsPermanent as String: true,
                                    kSecAttrAccessControl as String: access],
]
```
- `kSecAttrKeyTypeECSECPrimeRandom` + `256`: an elliptic-curve key on curve **P-256**, the key type the Secure Enclave supports for this API.
- `kSecAttrTokenIDSecureEnclave`: **generate the key inside the Secure Enclave**. The private key is never in normal memory, not even at creation.
- `kSecAttrIsPermanent: true`: save it in the Keychain so it survives app restarts. (Real code also sets a `kSecAttrApplicationTag` so it can find the key again later.)
- `kSecAttrAccessControl: access`: attach the rules from above.

```swift
let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, nil)!
```
Create the key pair. `privateKey` is only a **reference** (a handle) to a key that lives in the Secure Enclave, not the key bytes.

```swift
// Once: send SecKeyCopyPublicKey(privateKey) to your server.
```
**Enrolment:** extract the **public** key (that's allowed) and register it with your server for this user, over an already-authenticated session.

```swift
let signature = SecKeyCreateSignature(privateKey, .ecdsaSignatureMessageX962SHA256,
                                      serverChallenge as CFData, nil)
```
**Login:** the app fetches a random `serverChallenge` from the server and asks for a signature. iOS shows the Face ID prompt automatically, because the key's access control demands it. Only on a match does the Secure Enclave compute **ECDSA-P256(SHA-256(challenge))** (`X962` refers to the standard encoding of the signature). The app sends the signature to the server, which verifies it against the stored public key and checks the challenge is fresh and unused.

(The `!` force-unwraps keep the example short; production code must handle errors such as "the user cancelled" or "biometry changed".)

**Why `.biometryCurrentSet` matters:** imagine a thief watches you type your passcode, then steals the phone. With the passcode they can go to Settings and **enrol their own face**. Without `.biometryCurrentSet`, their face would now unlock your banking key. With it, enrolment changes destroy the key, and the app must make you log in properly again.

**Equivalents:**
- **Android:** generate a key in the Android Keystore with `setUserAuthenticationRequired(true)` (and optionally `setInvalidatedByBiometricEnrollment(true)`, Android's version of `.biometryCurrentSet`), then use `BiometricPrompt` with a `CryptoObject` wrapping a `Signature` using that key. The signature only works after the prompt succeeds.
- **The web:** you don't write any of this. WebAuthn **is** the key-gate pattern: the browser uses Face ID, Touch ID or Windows Hello as the platform authenticator.

**The common mobile pattern, explained:** after a full first login, the app stores its **refresh token** in the Keychain with an access control that requires biometrics. On the next launch, the app requests the token; iOS shows Face ID; on success the Keychain releases the token; the app exchanges it with the server for a fresh short-lived access token. The user experiences "open app, glance at phone, you're in".

### 6.5 Other biometrics

- **Fingerprint:** *capacitive* sensors measure tiny electrical differences between ridges (touching) and valleys (not touching); *optical* sensors photograph the finger (common under phone screens); *ultrasonic* sensors bounce sound waves to build a 3D ridge map, which is harder to fool with a printed fingerprint.
- **Windows Hello face:** requires an **infrared** camera (not the normal webcam), which works in the dark and resists photos (a printed photo looks different under IR). Keys live in the TPM; same key-gate design.
- **Iris:** the coloured ring around your pupil has highly distinctive, stable patterns. Captured under near-infrared light. Very accurate; used at some borders and secure facilities.
- **Voice:** a "voiceprint" of how you speak, used by some phone banking systems ("my voice is my password"). **AI voice cloning** can now imitate a person from a few seconds of audio, which makes voice alone increasingly unsafe.
- **Palm / vein:** infrared light reveals the vein pattern under the skin, which is hidden and hard to copy. Used in Amazon One payments and Japanese ATMs.
- **Behavioural:** not a body part but *how you behave*: typing rhythm, mouse movement, swipe pressure, how you hold the phone, gait. Banks run this **passively** in the background to flag sessions where "the person using this account suddenly behaves differently", e.g. a fraudster after takeover. This leads to **continuous authentication**: confidence is re-scored throughout the session instead of only at login.

### 6.6 Biometric concepts worth knowing

**FAR and FRR.** A biometric match is never exact; the system computes a similarity score and compares it to a **threshold**.
- **FAR (False Acceptance Rate):** how often an impostor is accepted. Security failure.
- **FRR (False Rejection Rate):** how often the real user is rejected. Usability failure.
- Raise the threshold: FAR falls, FRR rises (fewer impostors, but you get rejected more with wet fingers or bad lighting). Lower it: the reverse. Every product chooses a trade-off. (The point where FAR = FRR is called the **EER, equal error rate**, used to compare systems.)

**Liveness / Presentation Attack Detection (PAD).** A "presentation attack" means presenting a fake: a photo, a video replay, a silicone mask, a gummy fingerprint, or now an AI-generated **deepfake** video fed to a camera. Liveness detection checks for signs of a real, live person: depth, IR reflectance of skin, blinking, micro-movements, blood flow. It's especially critical for **remote identity verification** (e.g. "take a selfie to open a bank account"), where the system uses an ordinary phone camera without depth sensors.

**Template protection.** Systems don't store your raw face photo or fingerprint image. They store a **derived template** (the mathematical representation), ideally in secure hardware, so even a leaked template is hard to turn back into a usable face.

**Biometrics are identifiers, not secrets.** A secret is something only you know. Your face appears in every photo you've posted; you leave fingerprints on every glass you touch. Anyone motivated can obtain a copy. So a biometric can't be a *secret* like a password; its value comes from **liveness checks** (proving a real body is present *right now*) combined with **device possession** (it must be *your* phone's sensor). That's exactly the local-unlock design.

**Legal risk.**
- **GDPR** classifies biometric data used for identification as **special category data**: processing it needs explicit consent or another narrow legal basis, plus stronger safeguards.
- **Illinois BIPA** (Biometric Information Privacy Act) requires written consent before collecting biometrics and lets individuals sue directly. It has produced very large settlements, including **$650 million from Facebook** over face-tagging.
- Lesson: if your **server** collects biometric data, you take on serious legal liability. Letting the device handle biometrics (WebAuthn, the key gate) means you never touch biometric data at all.

---

## 7. Certificate and key-based authentication

### 7.1 Mutual TLS (mTLS)

#### First, normal TLS

When you visit `https://bank.com`:
1. Your browser says hello and lists the encryption it supports.
2. The server sends its **certificate** ("this public key belongs to bank.com, signed by a trusted CA") and proves it holds the matching private key by signing part of the handshake.
3. Your browser checks the certificate chain up to a trusted root CA (§0.9) and that the name matches `bank.com`.
4. Both sides derive shared encryption keys. Everything afterwards is encrypted.

The server is authenticated. **You are not**; you're anonymous at the TLS level until you log in inside the app.

#### Mutual TLS adds the client side

The diagram, step by step:

1. **ClientHello:** the client starts the handshake.
2. **ServerHello + server certificate + `CertificateRequest`:** the server presents its certificate *and* says "send me yours too". It may list which CAs it accepts.
3. **Client verifies the server's certificate**, as in normal TLS.
4. **Client sends its own certificate + `CertificateVerify`.** The certificate says "this public key belongs to `payments-service`". `CertificateVerify` is a **signature over the handshake transcript** made with the client's private key. This is challenge–response (§0.7): it proves the client actually **holds the private key**, not just a copy of the (public) certificate.
5. **Server verifies the client certificate** against the CA(s) it trusts for clients (usually the company's own private CA), checks it isn't expired or revoked, and verifies the signature.
6. **Encrypted channel, both sides authenticated.**

**Identity = the certificate's subject.** The server reads who the client is from the certificate: traditionally the **CN (Common Name)**, e.g. `CN=payments-service`, or more modern **SAN (Subject Alternative Name)** fields, such as a SPIFFE ID URI `spiffe://prod/ns/payments/sa/api` (namespace `payments`, service account `api`). Authorization can then say "only `payments-service` may call `POST /refunds`".

**Strengths, explained:**
- *Before any HTTP request is processed:* a client without a valid certificate can't even complete the connection. Your application code never sees unauthenticated traffic, which shrinks the attack surface a lot.
- *No shared secret to leak:* the server holds only CA certificates; each client's private key stays with the client.
- *Phishing-resistant:* certificates are presented automatically by software, not typed by humans.

**Costs, explained:** you need a **CA** to issue certificates; every client needs a certificate **issued, installed, rotated before expiry, and revoked** if compromised. Doing this manually for hundreds of services is painful and error-prone (expired certificates are a classic cause of outages). Modern platforms **automate** it with **short-lived certificates** (hours or days), which also makes revocation less necessary: a stolen certificate soon expires anyway.

**A practical gotcha:** if a load balancer terminates TLS in front of your app, the mTLS handshake happens with the load balancer, not your app. The load balancer must pass the verified identity along (e.g. in a header like `X-Forwarded-Client-Cert`), and your app must only accept that header from the load balancer.

**Where it's used, explained:**
- **Service meshes** (Istio, Linkerd) inject a proxy next to every service. The proxies get short-lived certificates automatically and do mTLS between each other, so every service-to-service call is encrypted and authenticated without any application code changes.
- **Open banking:** regulators require banks' APIs to authenticate third-party fintech apps with certificates.
- **IoT:** each device gets a unique certificate at manufacture, so a cloud backend can tell devices apart and revoke a single compromised device.
- **Zero trust** (§12.7): service identity instead of network location.

### 7.2 SSH keys

The same challenge–response idea as passkeys, and older (SSH dates from 1995):

1. `ssh-keygen -t ed25519` creates a key pair: `~/.ssh/id_ed25519` (private, protected by an optional passphrase) and `~/.ssh/id_ed25519.pub` (public). You put the public key in the server's `~/.ssh/authorized_keys` file, or in GitHub's settings.
2. When you connect, the client and server first set up an encrypted connection, which produces a unique **session identifier**. The server checks your public key is authorised, and the client **signs data containing that session identifier** with the private key.
3. The server verifies the signature with the stored public key. Because each session's identifier is different, a recorded signature can't be replayed.

(SSH also authenticates the **server** to you: its **host key** is saved in `~/.ssh/known_hosts` on first connect. That's the *"The authenticity of host … can't be established"* prompt.)

**SSH certificates:** with 500 servers and 200 engineers, copying public keys into every `authorized_keys` file and removing them when people leave is unmanageable. Instead, run an **SSH CA**: servers trust the CA's public key (`TrustedUserCAKeys` in the server config), and engineers get **short-lived certificates** (e.g. valid for 8 hours) after logging into SSO. Offboarding becomes automatic: no new certificates for leavers.

**`ed25519-sk` keys:** the `-sk` means "security key". The private key is protected by a FIDO hardware key, so SSH logins require you to **physically touch** the YubiKey. Malware on your laptop can't use the key silently.

### 7.3 Smart cards and PIV/CAC

A plastic card with a **secure element** chip holding a private key and its certificate.
- You insert or tap it and enter a **PIN**. The PIN unlocks the chip, which then signs challenges for you. The private key never leaves the chip.
- Too many wrong PINs and the chip **locks itself**, so stealing the card and guessing is impractical.
- **PIV** (Personal Identity Verification) is the US federal standard; **CAC** (Common Access Card) is the US military's version. Many countries' national ID cards work the same way.
- **Two factors:** the card (have) + the PIN (know).
- Uses: logging into Windows, VPNs, signing documents and email legally, opening doors.

### 7.4 Hardware security keys

Devices like **YubiKey** or **Google Titan**, plugged in via USB or tapped via NFC.
- **FIDO2/WebAuthn:** stores passkeys (device-bound, §5.7).
- **FIDO U2F:** the earlier standard (2014), designed as a **second factor** only (password + tap the key). Still phishing-resistant, because it also checks the origin.
- **Extra modes on many models:** OTP generation, smart card (PIV) mode, OpenPGP for email and Git commit signing.

**Why they're the strongest widely available factor:** non-exportable keys in tamper-resistant hardware, origin binding (phishing-resistant), and physical touch required. A widely cited data point: after Google required security keys for all employees (2017), it reported **no successful phishing-based account takeovers** of its employees.

Recommended for: admins, developers with production access, finance staff, executives, journalists, activists. Anyone whose account is worth a targeted attack.

---

## 8. Single Sign-On (SSO)

### 8.1 What it is

**SSO = log in once at a central Identity Provider, then use many applications without logging in again.**

- **IdP (Identity Provider):** the one system that actually checks your credentials (password, passkey, MFA) and then **vouches** for you to other apps. Examples: Okta, Microsoft Entra ID, Google Workspace, Keycloak (open source, self-hosted), Auth0, Ping, JumpCloud.
- **SP (Service Provider, SAML term) / RP (Relying Party, OIDC term):** each app that **trusts** the IdP's word instead of checking passwords itself: Slack, GitHub, Salesforce, your SaaS.
- **Federation:** the configured trust relationship: "Slack trusts assertions signed by Acme's Okta". Federation is often **across organisations** (Acme's IdP and Slack's servers belong to different companies), set up by exchanging metadata, certificates and URLs.

### 8.2 Why companies want it

- **One identity per employee:** remembering one strong password + MFA is realistic. Remembering 50 unique ones isn't; people reuse them instead.
- **Instant offboarding:** when an employee leaves, IT disables **one** IdP account and access to every connected app is gone. Without SSO, someone must remove them from 50 apps by hand, and the ones they forget (an old marketing tool, a forgotten admin panel) become backdoors for a disgruntled ex-employee or an attacker who later steals their credentials.
- **Central policy:** "everyone needs MFA", "admins need hardware keys", "only from company laptops", "sessions last 12 hours" are set **once at the IdP** and apply everywhere.
- **Audit:** security teams can answer "who accessed what, when" from one log.
- **Fewer passwords:** fewer to phish, fewer reset tickets for the help desk.

### 8.3 How it works under the hood

**Important misconception:** there's no magic shared session across apps. Browsers keep cookies **per domain**, so `notes.com` can't read `login.company.com`'s cookie. Instead there are **separate sessions**:
- the **IdP's session**: a cookie on `login.company.com`,
- and **each app's own session**: a cookie on `notes.com`, another on `tasks.com`.

The diagram, step by step:

1. **Browser → App A (`notes.com`):** no `a_sid` cookie, so not logged in.
2. **App A redirects to the IdP** with "please authenticate this user for App A" (an OIDC `/authorize` request or a SAML `AuthnRequest`).
3. **Browser → IdP:** the IdP has no session cookie for this browser, so it shows its **login page**.
4. **User submits password + MFA.** The IdP verifies them, creates **its own session**, and sets `idp_sid` on `login.company.com`.
5. **IdP redirects back to App A** with an authorization code (OIDC) or a signed assertion (SAML).
6. **App A verifies** the code/assertion and creates **its own session**, setting `a_sid` on `notes.com`. The user is in App A.
7. **Later, browser → App B (`tasks.com`):** no `b_sid`, so not logged in to App B.
8. **App B redirects to the IdP.** This time the browser **automatically sends `idp_sid`**, because the request goes to `login.company.com`.
9. **The IdP sees a valid session: no prompt.** It immediately redirects back to App B with a code/assertion for App B.
10. **App B verifies and sets `b_sid`.** The user sees, at most, a brief flicker of redirects.

That's the whole trick: the "single" in single sign-on is **the IdP's session being reused** by every app's redirect.

### 8.4 SSO protocols

- **SAML 2.0 (2005):** XML-based. Built when enterprise web apps were server-rendered pages. Still the dominant way enterprises connect their IdP to SaaS products.
- **OpenID Connect (2014):** JSON and JWT-based, built on OAuth 2.0. The modern default: social login, mobile apps, single-page apps. The first file walks through its flow.
- **Kerberos (1980s, MIT):** binary tickets. SSO **inside** a corporate network, e.g. Windows logons and file shares (§9.1).
- **CAS (Central Authentication Service):** a simple ticket-based protocol from Yale, still common at universities.
- **WS-Federation:** Microsoft's older XML protocol (used by ADFS). Legacy; you'll only meet it in older Microsoft environments.

### 8.5 SAML 2.0 in depth

#### The SP-initiated flow, step by step

1. **User visits `app.com/login?org=acme`.** Your app looks up Acme's SAML configuration (their IdP's URL and certificate).
2. **Your app builds an `<AuthnRequest>`**: a small XML message saying "please authenticate a user for SP `https://app.com`, request ID `_req123`, send the answer to `https://app.com/saml/acs`".
3. **Redirect to the IdP** using the **HTTP-Redirect binding**: the XML is compressed (deflate), base64-encoded and URL-encoded into `?SAMLRequest=…`. `RelayState=/dashboard` rides along so you know where to send the user afterwards.
4. **The IdP authenticates the user**, or reuses an existing IdP session (that's the SSO part).
5. **The IdP returns an HTML page with an auto-submitting form** (the **HTTP-POST binding**): `<form method="POST" action="https://app.com/saml/acs">` with a hidden `SAMLResponse` field, plus a tiny script that submits it immediately. Why POST? The signed XML response is far too large for a URL.
6. **The browser POSTs to your ACS URL.** Your app then validates, in order:
   1. **The XML signature** using the IdP's certificate (from their metadata). Proves the assertion came from Acme's IdP and wasn't modified.
   2. **`Issuer`** is Acme's IdP entity ID (not some other customer's IdP). **`Audience`** is *your* SP entity ID (an assertion meant for another app can't be replayed at yours). **`Destination`/`Recipient`** is your ACS URL. **`InResponseTo`** matches the request ID `_req123` you issued (proves this is an answer to *your* request, in *this* browser).
   3. **Time conditions:** `NotBefore` ≤ now < `NotOnOrAfter`, allowing a minute or two of clock skew. Also remember used assertion IDs until they expire, so the same assertion can't be posted twice.
   4. **Read the identity:** `NameID` (e.g. `alice@acme.com`) and attributes (groups, name). Find or create the user (JIT, §8.9) and create a session.
7. **Set your session cookie and redirect to `RelayState`** (`/dashboard`). Validate `RelayState` is a path on your own site, or it becomes an open redirect.

#### The assertion, element by element

- `<saml:Assertion ID="_a1b2c3" IssueInstant="…">`: the assertion's unique ID (for replay detection) and when it was created.
- `<saml:Issuer>https://acme.okta.com</saml:Issuer>`: who made it.
- `<ds:Signature>`: an XML digital signature over the assertion, made with the IdP's private key.
- `<saml:Subject>`: **who** the assertion is about.
  - `<saml:NameID>alice@acme.com</saml:NameID>`: the user's identifier.
  - `<saml:SubjectConfirmation Method="…:bearer">`: "**whoever presents** this assertion may use it" (just like a bearer token), within the limits that follow.
  - `<saml:SubjectConfirmationData InResponseTo="_req123" Recipient="https://app.com/saml/acs" NotOnOrAfter="…10:05:00Z"/>`: only valid as a reply to request `_req123`, only at this ACS URL, only until 10:05.
- `<saml:Conditions NotBefore="…09:59:00Z" NotOnOrAfter="…10:05:00Z">`: the overall validity window, **five minutes**. Assertions are meant to be consumed immediately.
  - `<saml:AudienceRestriction><saml:Audience>https://app.com</saml:Audience>`: **only for `app.com`**.
- `<saml:AuthnStatement AuthnInstant="…">`: when and (in the omitted part) how the user authenticated at the IdP, e.g. password vs MFA.
- `<saml:AttributeStatement>`: extra facts about the user: here, `groups = engineering`. Apps often map groups to roles (RBAC).

#### Key SAML terms, explained

- **ACS URL (Assertion Consumer Service):** the endpoint on *your* app that receives POSTed assertions (`/saml/acs`). The IdP is configured with it.
- **Entity ID:** a globally unique name for each party, usually a URL like `https://app.com`, but it's just an identifier, not necessarily a reachable page.
- **Metadata:** an XML file each side publishes containing its entity ID, endpoint URLs, and **signing certificate**. When a customer "connects their Okta" to your app, they're really exchanging metadata: they upload yours to Okta, and you import theirs.
- **Bindings:** *how* SAML messages travel over HTTP. **HTTP-Redirect** puts the (small) request in the URL; **HTTP-POST** puts the (large) response in an auto-submitted form.
- **RelayState:** an opaque value that survives the round trip, typically the page the user originally wanted.
- **IdP-initiated flow:** the user starts at their Okta dashboard and clicks your app's tile. Okta POSTs an assertion your app **never requested**, so there's no `InResponseTo` to check. That weakens replay protection and enables **login CSRF** (an attacker could get a victim's browser to POST the *attacker's* valid assertion). Many security teams prefer to convert IdP-initiated logins into SP-initiated ones: on receiving one, redirect back to the IdP with a fresh request.

#### The security warning, explained

SAML signatures use **XML-DSig**, which signs one *element* within a larger document, identified by an `ID` attribute. The signature check and the code that reads the data can disagree about **which element they're looking at**:

- **XML Signature Wrapping (XSW):** the attacker takes a legitimately signed assertion (for their own account), moves it somewhere harmless in the document, and inserts a new, **unsigned** assertion saying `NameID = admin@victim.com` where the application expects it. The signature validator finds the signed element by ID and says "valid ✔". The application code reads the *first* assertion it sees, which is the forged one.
- **Comment and parser tricks:** in 2018 several libraries were found to read `alice@acme.com<!---->.evil.com` as just `alice@acme.com`, because of how XML parsers split text around comments. More parser-differential bugs were found in widely used SAML libraries (e.g. Ruby's `ruby-saml`) as recently as 2024–2025.

Hence: **never write your own SAML validation.** Use a mature, maintained library, keep it updated, or use a provider (WorkOS, Auth0, Okta, Keycloak, BoxyHQ) that turns enterprise SAML into a simpler OIDC-style integration for you.

### 8.6 SAML vs OIDC, row by row

- **Data format:** SAML uses verbose XML assertions; OIDC uses compact JSON in a JWT ID token.
- **Signing:** XML-DSig requires canonicalising XML (normalising whitespace, namespaces, attribute order) before signing, the source of many bugs. JWS (JWT signing) just signs base64url bytes: far simpler.
- **Transport:** SAML pushes the whole assertion through the **browser** via a form POST. OIDC's code flow sends only a short **code** through the browser; the app then fetches tokens **directly from the IdP** over a server-to-server (back-channel) call, so tokens never pass through the browser's URL bar or history.
- **Mobile and SPAs:** auto-submitting HTML forms and huge XML are awkward in native apps; OIDC was designed with mobile and SPAs in mind (e.g. PKCE).
- **API access:** SAML says who the user is but gives no token for calling APIs. OIDC is built on OAuth, so you also get **access tokens**.
- **Where you'll see it:** enterprise IdPs and established SaaS speak SAML; everything new prefers OIDC.
- **For B2B SaaS:** support **both**, because enterprise customers' IT teams will ask for whichever their IdP is set up for.

### 8.7 Social login and "Sign in with Apple"

"Sign in with Google / GitHub / Microsoft" is **federated login over OIDC**: your app is the RP, Google is the IdP. It's technically SSO too: one Google session signs you into many unrelated sites.

**Sign in with Apple**, explained:
- **Hide My Email:** Apple generates a random address like `x7k2p9@privaterelay.appleid.com` unique to your app, which forwards to the user's real inbox. Your app never learns the real address, two apps can't correlate the user by email, and the user can turn forwarding off at any moment. To send email through the relay, you must register your sending domains with Apple, or your messages are dropped.
- **Face ID / Touch ID approval:** on Apple devices the sign-in is approved biometrically, using the Secure Enclave machinery from §6.
- **Name only on first login:** Apple includes the user's name in the **first** authorization response only. If your code doesn't save it then, you can't get it again (without the user revoking and re-authorising). A classic integration bug.
- **The App Store rule:** Apple's guidelines have required apps that offer third-party social login to also offer Sign in with Apple; since a 2024 change, an equivalent privacy-focused login option can satisfy the rule instead.
- **Identify users by `sub`,** the stable user ID in the ID token, never by email (which may be a relay address and can change).

### 8.8 Single Logout (SLO) is hard

When you click "Log out" in App A, App A deletes **its** session. But:
- the IdP session still exists, so App A's next redirect logs you straight back in, and
- Apps B, C and D still have their own sessions.

**Protocols for SLO:**
- **Front-channel logout:** the IdP loads each app's logout URL in hidden iframes or a chain of redirects in your browser. Fragile: modern browsers block third-party cookies in iframes, and one broken app stops the chain.
- **Back-channel logout:** the IdP calls each app server-to-server ("end session X"). More reliable, but every app must store a mapping from IdP session to local sessions and implement the endpoint.

**What most systems do in practice:** keep **app sessions short** (so they re-check with the IdP often), make logout at the IdP end the IdP session, and rely on **deprovisioning** (§8.9) to cut access for people who leave.

### 8.9 Provisioning: SCIM and JIT

SSO proves *who is logging in*. But your app also needs a **user record** (with a role, a team, settings), and that record should be **removed** when the person leaves.

**JIT (Just-In-Time) provisioning:** on the first SSO login, read the assertion's `NameID` and attributes and create the user automatically.
- Simple: no extra integration.
- The gap: nothing happens when someone **leaves**. Their account stays in your app (still visible in member lists, still counted for billing, possibly with active API keys) until they try to log in, which they never will.

**SCIM 2.0 (System for Cross-domain Identity Management):** a standard REST API that **you implement**, and the **IdP calls**:
- `POST /scim/v2/Users`: a new employee was assigned your app → create them.
- `PATCH /scim/v2/Users/{id}`: their name or department changed → update.
- `PATCH … {"active": false}` or `DELETE`: they left → **deactivate immediately**, revoking sessions and tokens.
- `/scim/v2/Groups`: keep group memberships (and therefore roles) in sync.

The IdP authenticates to your SCIM endpoint with a bearer token you generate for that customer. When HR offboards Alice in Okta, Okta calls your endpoint within seconds.

**Directories:** the IdP itself gets its list of employees from somewhere: an **LDAP / Active Directory** directory (§9.2) or an HR system (Workday etc.). The chain is often: HR system → directory → IdP → (SSO + SCIM) → every app.

### 8.10 The "SSO tax"

Many SaaS vendors put SAML SSO only in their most expensive "Enterprise" plan, sometimes multiplying the price. Critics call this the **SSO tax**, since it makes a basic security feature unaffordable for small companies. As a builder, the lesson is practical: **enterprise buyers will require SAML/OIDC SSO and SCIM** in security reviews before signing a contract, so design your user model for organisations, external identities and deprovisioning from the start.

---

## 9. Enterprise and network protocols

### 9.1 Kerberos

#### What problem it solves

In a company network in the 1980s (MIT's Project Athena), users needed to access many services (file servers, printers, mail) over a network that **anyone could eavesdrop on**. Sending passwords to every service was out of the question. Kerberos lets you **prove your identity to any service without sending your password anywhere**, and without typing it more than once a day.

Kerberos uses only **symmetric** keys (§0.5). The trick is that a trusted third party, the **KDC**, knows everyone's key:
- **your key** is derived from your password,
- **each service's key** is configured when the service is registered,
- the KDC has its own secret key (on Windows, the key of the special `krbtgt` account).

Analogy: a fairground. You show ID once at the main booth (**AS**) and get a **wristband** (the TGT). At each ride's ticket booth (**TGS**), you show the wristband and get a **ride ticket**. The ride operator (**service**) only checks the ticket, never your ID.

#### The flow, step by step

**Step 1: AS-REQ ("I'm alice").** Your PC derives a key from your password and uses it to **encrypt the current timestamp**. It sends "I'm alice" + that encrypted timestamp. This is **pre-authentication**: the KDC decrypts it with the copy of Alice's key it stores; if that yields a current timestamp, the sender must know Alice's password. The password itself never crosses the network.

**Step 2: AS-REP (the TGT).** The KDC creates a random **session key** (call it S1) and replies with two things:
- the **TGT (Ticket-Granting Ticket)**: contains "alice", S1 and an expiry, **encrypted with the KDC's own key**. Alice can't read or modify it; she can only hand it back.
- **S1 encrypted with Alice's key**: only someone knowing her password can open it.

Your PC decrypts S1. You now hold a TGT (opaque to you) and S1 (known to you and the KDC).

**Step 3: TGS-REQ ("I want the fileserver").** Your PC sends the TGT, the name of the service it wants, and an **authenticator** (your name + current timestamp, encrypted with S1).

**Step 4: TGS-REP (the service ticket).** The TGS decrypts the TGT with its own key, finds S1 inside, uses S1 to decrypt the authenticator, and checks the name and timestamp match. It creates a new session key S2 and returns:
- a **service ticket**: "alice", S2, expiry, **encrypted with the fileserver's key**,
- S2 encrypted with S1 (so your PC can read it).

**Step 5: AP-REQ (to the service).** Your PC sends the service ticket plus a new authenticator encrypted with S2. The fileserver decrypts the ticket **with its own key**, which only it and the KDC know, so the ticket must have come from the KDC. Inside it finds "alice" and S2, decrypts the authenticator with S2, and checks the timestamp. **"This really is Alice."**

**Step 6 (optional): mutual authentication.** The server replies with something encrypted with S2 (based on your timestamp), proving it was able to open the ticket, i.e. that it's the real fileserver.

#### The bullet points, explained

- **The TGT is a day pass** (often ~10 hours). As long as it's valid, steps 3–5 happen silently for every new service. That's why you only type your password when you log into Windows in the morning.
- **The service never contacts the KDC during login.** It trusts the ticket purely because it decrypts correctly with a key only the KDC and the service share. This scales well: the KDC isn't involved in every access.
- **Clocks must be synchronised** (default tolerance about 5 minutes). Timestamps inside authenticators are what stop an eavesdropper replaying them later; if clocks disagree by more than the tolerance, logins fail with "clock skew too great". Domain-joined machines sync time with the domain controller.
- **Kerberos on the web (SPNEGO):** an intranet site replies `401` with `WWW-Authenticate: Negotiate`. The browser (if the site is on its trusted intranet list) obtains a service ticket for `HTTP/intranet.company.com` and sends it in `Authorization: Negotiate <ticket>`. The user is logged in with no prompt: "**Integrated Windows Authentication**".

#### The attacks, explained

- **Pass-the-ticket:** malware with admin rights on a PC can copy Kerberos tickets out of memory and reuse them from another machine, impersonating the user until the tickets expire. No password needed.
- **Kerberoasting:** **any** logged-in domain user can request a service ticket for **any** service. That ticket is encrypted with the service account's key, which is derived from the service account's **password**. The attacker takes the ticket offline and tries billions of password guesses until one decrypts it. Service accounts often have old, human-chosen passwords that never change. Defence: long random service-account passwords (or managed service accounts that rotate automatically) and AES-only encryption.
- **Golden ticket:** if attackers steal the KDC's own key (the `krbtgt` account's key, after compromising a domain controller), they can **forge TGTs** for any user, including domain admins, with any lifetime. It's total domain compromise; recovery requires resetting the `krbtgt` key (twice).

### 9.2 LDAP and Active Directory

**LDAP (Lightweight Directory Access Protocol)** is a protocol for reading and writing a **directory**: a hierarchical database of people, groups, computers and their attributes (email, phone, manager, group memberships).

Entries are identified by a **DN (Distinguished Name)**, read right to left like an address:
```
cn=alice,ou=engineering,dc=company,dc=com
```
- `dc=company,dc=com`: **domain components**: the company's domain, `company.com`.
- `ou=engineering`: an **organisational unit** (like a folder).
- `cn=alice`: the **common name** of the entry itself.

**Active Directory (AD)** is Microsoft's directory service and the backbone of most corporate Windows networks. It uses **LDAP** for lookups and **Kerberos** for authentication (§9.1). Open-source alternatives: **OpenLDAP** (directory only) and **FreeIPA** (directory + Kerberos, Linux-focused).

**"LDAP authentication" = an LDAP *bind*.** Typical flow inside an app like Jenkins or Grafana:
1. The app connects to the directory (often using a service account) and **searches** for the user's DN, e.g. "find the entry where `mail=alice@company.com`".
2. The app then **binds** (logs in to the directory) **as that DN with the password the user typed**.
3. Bind succeeds → password correct → user authenticated. The app may also read the user's groups to assign roles.

**The downsides, explained:**
- **The app sees the password.** Every app with LDAP login receives users' real directory passwords: the same password that opens their email and laptop. One compromised internal tool leaks the company-wide password. This is why modern setups put an **IdP** (SAML/OIDC) in front of the directory: only the IdP ever sees passwords.
- **Plain LDAP on port 389 is unencrypted**, so the password crosses the network in the clear. Use **LDAPS** (LDAP over TLS, port 636) or upgrade the connection with **StartTLS**.
- **Classic bug: the empty-password bind.** Many LDAP servers treat a bind with a username and an **empty password** as an "anonymous" bind, which **succeeds**. An app that only checks "did the bind succeed?" lets anyone log in as anyone by leaving the password blank. Always reject empty passwords before binding.
- **LDAP injection:** building search filters by string concatenation (`(mail=` + input + `)`) lets attackers inject filter syntax like `*)(…`, similar to SQL injection. Escape user input.

### 9.3 NTLM

Microsoft's older challenge–response protocol (from the 1990s), still present for compatibility when Kerberos can't be used (e.g. connecting by IP address, machines outside the domain).

**How it works:** the server sends a random challenge; the client responds with a value computed from the challenge and the **NT hash** of the password (an unsalted MD4 hash).

**Why it's dangerous:**
- **Pass-the-hash:** the response is computed from the **hash**, not the password. So the hash is **password-equivalent** (the same flaw as Digest's HA1, §2.2). Attackers who dump hashes from one machine's memory can authenticate as that user elsewhere without ever cracking the password.
- **Relay attacks:** an attacker tricks a machine into authenticating to *them*, then forwards each challenge and response to a **real** server in real time, logging in as the victim.

Microsoft has **deprecated** NTLM and is phasing it out of Windows. If a system you own still depends on it, plan the migration to Kerberos or modern protocols.

### 9.4 RADIUS, TACACS+ and 802.1X

These authenticate access to **the network itself** rather than to an application. They follow the **AAA** model: **Authentication** (who are you?), **Authorization** (what may you access?), **Accounting** (log what you did and for how long).

**802.1X / "WPA2/WPA3-Enterprise" Wi-Fi.** Home Wi-Fi uses one shared password for everyone ("WPA2-Personal"); anyone who knows it is in, and you can't revoke one person. Enterprise Wi-Fi authenticates **each user or device individually**:
1. Your laptop (the **supplicant**) connects to the access point (the **authenticator**), which initially allows only authentication traffic.
2. Your laptop sends credentials using **EAP** (Extensible Authentication Protocol), e.g. **EAP-TLS** (a device certificate, the strongest) or **PEAP** (a username/password inside a TLS tunnel).
3. The access point relays them to a **RADIUS server** (the **authentication server**), which checks them against the directory and replies "accept" (maybe with a VLAN assignment) or "reject".
4. On accept, the access point opens the network to your device. When an employee leaves, disabling their account cuts their Wi-Fi access too.

**RADIUS** is the same yes/no protocol used by many **VPNs**: the VPN server asks RADIUS whether these credentials are valid.

**TACACS+** is used for **administrators logging into network devices** (routers, switches, firewalls). Compared with RADIUS it encrypts the entire message (RADIUS only hides the password), runs over TCP, and separates authorization so it can approve or deny **each command** ("this engineer may run `show` commands but not `configure`"), logging everything for audit.

---

## 10. Machine-to-machine beyond API keys

**The problem with API keys:** an API key is sent **as-is** on every request. Anyone who sees one request (a log line, a proxy, a debugging screenshot) has the key forever, and the key proves nothing about *what* the request said. The methods here fix that.

### 10.1 HMAC request signing

**The idea:** keep the secret on the client and send a **proof** instead: an HMAC (§0.6) computed over the request.

```
signature = HMAC-SHA256(secret, method + path + selected headers + hash(body) + timestamp)
```

Why each part is included:
- **method + path:** a signature for `GET /orders` can't be reused for `DELETE /orders`.
- **selected headers** (e.g. `Host`, `Content-Type`): can't be redirected to a different host or reinterpreted.
- **hash(body):** changing a single byte of the body (`"amount": 10` → `10000`) breaks the signature.
- **timestamp:** the server rejects requests older than a few minutes, so a captured request can't be replayed tomorrow.

The client also sends an identifier for *which* secret was used (a key ID), so the server knows which secret to check with.

**The server** looks up the secret by key ID, rebuilds exactly the same string from the received request, computes the HMAC, and compares in **constant time** (§0.8). Both sides must build the string **byte-for-byte identically** (same header order, same encoding), so these schemes define a strict **canonical request** format.

**Benefits, explained:**
- **Integrity:** any modification in transit, by a proxy or an attacker, is detected.
- **Replay protection:** a signed timestamp plus a short acceptance window.
- **No secret on the wire:** logging a full request leaks a signature that's useless for any other request.

**The trade-off:** the server must be able to **retrieve the secret itself** in order to recompute HMACs, so it can't store just a hash (unlike API keys). Secrets must be encrypted at rest, like TOTP secrets.

**AWS Signature Version 4**, the header explained:
```
Authorization: AWS4-HMAC-SHA256 Credential=AKIA…/20261008/eu-west-1/s3/aws4_request,
               SignedHeaders=host;x-amz-content-sha256;x-amz-date, Signature=fe5f80f7…
```
- `AWS4-HMAC-SHA256`: the algorithm.
- `Credential=AKIA…/20261008/eu-west-1/s3/aws4_request`: the **access key ID** (public identifier, `AKIA…`), followed by the **scope**: date, region, service. AWS derives a **signing key** by HMAC-chaining the secret through the date, region and service. So a signing key leaked for S3 in `eu-west-1` today is useless for another service, region or day.
- `SignedHeaders=…`: which headers were included in the signature.
- `Signature=…`: the HMAC result.

The AWS SDKs and CLI do all of this for you on every call.

### 10.2 Webhook signatures

**The problem:** your webhook endpoint (`POST /webhooks/stripe`) is a public URL. Anyone who discovers it can POST a fake `payment_intent.succeeded` event and get goods for free, unless you verify the sender.

**The fix:** when you register the webhook, the provider gives you a **signing secret**. Every webhook includes an HMAC of the payload made with that secret. Only Stripe and you know it.

- **Stripe:** `Stripe-Signature: t=1728380000,v1=5257a869…`. `t` is the Unix timestamp; `v1 = HMAC-SHA256(secret, t + "." + rawBody)`. Including `t` in the signed string means the timestamp can't be changed to replay old events.
- **GitHub:** `X-Hub-Signature-256: sha256=<HMAC-SHA256(secret, rawBody)>`. No timestamp in the signature, so deduplicate deliveries by their delivery ID.

**The code, line by line:**

```js
function verifyWebhook(rawBody, header, secret, toleranceSec = 300) {
```
`rawBody` must be the **exact bytes** received, not a parsed object (see bug 1 below). `toleranceSec = 300` is a 5-minute replay window.

```js
  const parts = Object.fromEntries(header.split(",").map((kv) => kv.split("=")));
```
Turns `"t=1728380000,v1=5257a869…"` into `{ t: "1728380000", v1: "5257a869…" }`. (Simplified: Stripe can send **several** `v1` values while you're rotating secrets, and this one-liner keeps only the last. The official SDK handles that.)

```js
  const timestamp = Number(parts.t);
  if (!timestamp || Math.abs(Date.now() / 1000 - timestamp) > toleranceSec) return false;
```
If `t` is missing, `Number(undefined)` is `NaN`, which is falsy, so we reject. Otherwise reject if the event is more than 5 minutes from now in either direction. An attacker who recorded a genuine webhook can't replay it later.

```js
  const expected = crypto.createHmac("sha256", secret).update(`${timestamp}.${rawBody}`).digest("hex");
```
Recompute what the signature *should* be, using the same recipe Stripe used.

```js
  const a = Buffer.from(expected);
  const b = Buffer.from(parts.v1 ?? "");
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```
`timingSafeEqual` throws if lengths differ, so check length first (length isn't secret: every valid signature is 64 hex characters). Then compare in constant time.

**The two classic bugs, explained:**

1. **Verifying re-serialised JSON.** Frameworks like Express usually parse the body into an object. If you then call `JSON.stringify(req.body)` to verify, you get *different bytes*: spacing, key order, number formatting or Unicode escaping may differ from what Stripe signed. The HMAC never matches, and developers sometimes "fix" this by disabling verification. **Capture the raw body** for the webhook route (e.g. `express.raw({ type: "application/json" })`) and parse it only *after* verifying.
2. **Comparing with `===`.** Leaks timing information that lets an attacker forge a signature byte by byte (§0.8).

Good webhook hygiene also includes: process each event ID **once** (providers retry, so duplicates happen), respond `2xx` quickly and do heavy work asynchronously, and support **rotating** the secret.

### 10.3 Mutual TLS

Covered in §7.1. For machine-to-machine traffic it's the strongest common option: the identity check happens in the connection itself, and a service mesh can automate certificates so developers don't handle any secrets.

### 10.4 Workload identity: no long-lived secrets at all

**The problem with any secret** (API key, HMAC key, database password): it has to be **stored somewhere** the workload can read it: an environment variable, a config file, a CI secret. Secrets get committed to Git, printed in logs, copied to laptops, and live for years. Most cloud breaches involve a leaked long-lived credential.

**The idea of workload identity:** the **platform** running your code already knows what it is ("this is the VM `i-0abc` in account 1234 with role `orders-api`"). Let the platform **vouch** for the workload and hand out **short-lived, automatically rotated** credentials. There's nothing long-lived to leak.

**Cloud IAM roles.** You attach a role to an EC2 instance, Lambda function or GCP VM. The code (via the SDK) asks the cloud's **instance metadata service**, a special local address (`169.254.169.254`) reachable only from inside that machine, for credentials. It receives temporary credentials valid for about an hour, which the SDK refreshes automatically. No access keys in env vars.
*Gotcha:* if your app has an **SSRF** bug (an attacker can make your server fetch arbitrary URLs), the attacker can ask it to fetch the metadata service and steal the role credentials. This was central to the **2019 Capital One breach**. AWS's **IMDSv2** requires a session token obtained via a `PUT` request, which blocks most SSRF attacks. Enforce it.

**Kubernetes service account tokens.** Each pod runs as a **service account**. Kubernetes mounts a **projected token** into the pod: a JWT that names the service account, is **short-lived** (rotated automatically), **audience-bound** (it's only valid for the service it was minted for), and becomes invalid when the pod is deleted. Other services, or cloud providers, can verify it with the cluster's public keys.

**SPIFFE / SPIRE.** **SPIFFE** is an open standard for workload identity that works across clouds, Kubernetes and bare metal:
- Each workload gets an ID: `spiffe://company.com/payments/api`.
- It proves that ID with an **SVID** (SPIFFE Verifiable Identity Document): a short-lived X.509 certificate (used for mTLS) or a JWT.
- **SPIRE**, the reference implementation, issues SVIDs only after **attesting** the workload: checking what node it's running on and what process it is (its Kubernetes namespace and service account, its Unix user, its container image).

**OIDC federation for CI/CD**, the diagram step by step:
1. A GitHub Actions job (with the `id-token: write` permission) asks GitHub's OIDC provider for a token.
2. GitHub returns a **signed JWT** describing the job:
   - `iss: token.actions.githubusercontent.com`: issued by GitHub.
   - `sub: "repo:acme/api:ref:refs/heads/main"`: this job runs in repo `acme/api`, on branch `main`.
   - `aud: "sts.amazonaws.com"`: it's intended for AWS.
3. The job calls **AWS STS `AssumeRoleWithWebIdentity`** with the JWT.
4. AWS fetches GitHub's **public keys** (from its OIDC discovery endpoint), verifies the signature, and checks the role's **trust policy**, e.g. "only tokens whose `sub` is `repo:acme/api:ref:refs/heads/main`".
5. AWS returns **temporary credentials** (~1 hour). The deploy runs. Nothing permanent was ever stored in GitHub.

**Critical gotcha:** the trust policy **must restrict `sub`** (repo and branch, or environment). GitHub signs tokens for **every** repository on GitHub. A policy that only checks the issuer and audience lets *anyone's* GitHub workflow assume your role.

This is the same OIDC from the first file, with **machines** as the users: GitHub is the IdP, AWS is the relying party.

### 10.5 Sender-constrained tokens

**The bearer problem:** a bearer token (§11) is like cash. Whoever holds it can spend it. A token stolen from logs, a compromised proxy or a malicious browser extension works for the thief exactly as for the owner.

**The fix:** make the token like a **cheque made out to a specific person**: bind it to a key that only the legitimate client holds, and require proof of that key on every use.

**DPoP (Demonstrating Proof-of-Possession):**
1. The client generates a key pair.
2. When requesting tokens, it includes a **DPoP proof**: a small JWT signed with its private key, containing its public key.
3. The authorization server issues an access token containing a **thumbprint** (hash) of that public key (the `cnf` / "confirmation" claim).
4. On **every API request**, the client sends the token **plus a new DPoP proof** signed with the private key, covering the HTTP method, the URL, a timestamp, a unique ID, and a hash of the access token.
5. The API checks that the proof's key matches the thumbprint inside the token.

A thief who steals the token but not the private key can't create valid proofs, so the token is useless. Because each proof covers the method, URL and time, captured proofs can't be reused for other requests either. The HTTP header becomes `Authorization: DPoP <token>` plus a `DPoP: <proof>` header.

**mTLS-bound tokens:** the same idea using the client's **TLS certificate** as the key: the token contains a hash of the client certificate, and the API only accepts it over an mTLS connection made with that certificate.

Both appear in high-security OAuth profiles such as **FAPI** (Financial-grade API), which open-banking regulations build on.

---

## 11. HTTP authentication schemes

### 11.1 The common framework

HTTP defines one generic **challenge–response framework**, and the "schemes" are plug-ins to it:

1. Client requests a protected resource without credentials.
2. Server replies **`401 Unauthorized`** with **`WWW-Authenticate: <scheme> <parameters>`**, saying "authenticate using this scheme".
3. Client retries with **`Authorization: <scheme> <credentials>`**.

Two related status codes worth getting right:
- **401 Unauthorized** really means **unauthenticated**: "I don't know who you are" (or your credentials are invalid). Must include `WWW-Authenticate`.
- **403 Forbidden** means **unauthorized**: "I know who you are, and you're not allowed".

(There's also a proxy version, `407 Proxy Authentication Required` with `Proxy-Authenticate` / `Proxy-Authorization`, for logging into a corporate proxy.)

### 11.2 Each scheme in the table

- **`Basic`:** base64 of `user:password` (§2.1). HTTPS only.
- **`Digest`:** hash-based challenge–response (§2.2). Obsolete.
- **`Bearer`:** "**whoever bears (holds) this token gets access**". The token may be a JWT, an opaque random string, or an OAuth access token; the scheme doesn't care. This is the standard for APIs. On failure the server can explain why: `WWW-Authenticate: Bearer error="invalid_token", error_description="expired"`.
- **`Negotiate`:** carries Kerberos (or, as a fallback, NTLM) tokens via SPNEGO. Gives silent single sign-on on Windows intranets (§9.1).
- **`DPoP`:** a sender-constrained access token, always accompanied by a `DPoP` proof header (§10.5).
- **`AWS4-HMAC-SHA256`:** a vendor-specific scheme name for AWS's signed requests (§10.1). Any service can define its own scheme name this way.
- **Custom headers like `X-API-Key`:** very common for API keys, but not part of the standard framework (no `WWW-Authenticate` challenge). Fine in practice; just be consistent and keep them out of logs.

### 11.3 Cookies sit outside this

Browser sessions don't use `Authorization` at all. After a login form, the server sets `Set-Cookie: sid=…`, and the browser automatically sends `Cookie: sid=…` with every request to that site. That automatic sending is what makes cookies convenient for browsers, and also what creates **CSRF** risk (covered in the first file's cookie section). The `Authorization` header, by contrast, is only sent when your code explicitly adds it.

---

## 12. Patterns layered on top

These aren't credentials or protocols. They're **policies** (Layer 3 in §1) that decide *when* and *how strongly* to authenticate.

### 12.1 MFA and assurance levels

**MFA** combines factors from **different categories** (§0.1).

NIST's **Authenticator Assurance Levels (AALs)** grade how confident you can be that the person logging in is the account owner:

- **AAL1, single factor:** e.g. a password alone. Some assurance; falls to any single leak or phish.
- **AAL2, two factors:** e.g. password + TOTP, or a passkey unlocked with biometric/PIN (one device, two factors: §5.8). High confidence against remote attackers, but OTP-based options can still be phished in real time.
- **AAL3, hardware-backed and phishing-resistant:** e.g. a hardware security key with a PIN, or a smart card. The key must live in hardware that resists extraction, and the method must be bound to the site (§5.6). The highest level, used for government and high-privilege access.

**The ranking, explained:** **SMS < email < TOTP app < push with number matching < passkey / security key**.
- **SMS** is weakest: vulnerable to SIM swap and interception, plus phishing (§3.2).
- **Email** depends on the email account's security, and is still phishable.
- **TOTP** can't be SIM-swapped or intercepted, since the secret never travels after setup, but codes can be phished in real time.
- **Push with number matching** defeats push bombing and requires the user to look at the real login screen, though a sophisticated real-time phishing proxy can still display the number to the victim.
- **Passkeys / security keys** are origin-bound: phishing fails by construction.

(The ordering of email vs SMS is debatable and depends on how well the email account is protected. The important split is between **phishable** factors and **phishing-resistant** ones.)

### 12.2 Step-up authentication

**The idea:** a session cookie proves you logged in **at some point**, maybe weeks ago on a laptop that's since been left unlocked. For dangerous actions, demand **fresh** or **stronger** proof right now.

- **GitHub's sudo mode:** before deleting a repository, adding SSH keys or changing 2FA settings, GitHub asks for your password or passkey again. After you confirm, you get a few hours of "sudo" before it asks again.
- **Banks:** viewing your balance needs a normal login; sending money to a **new** recipient needs an extra confirmation in the app.

**Implementation, explained:**
- In the session, store **`auth_time`** (when the user last actively authenticated) and **how** (e.g. `["pwd", "otp"]` or `["hwk"]`). OIDC standardises these as claims in the ID token: `auth_time`, **`amr`** (Authentication Methods References: which methods were used) and **`acr`** (Authentication Context Class Reference: what assurance level was achieved).
- A middleware on sensitive endpoints checks: "was `auth_time` within the last N minutes, and was the method strong enough?" If not, respond with a "re-authentication required" error; the frontend shows a password/passkey prompt, then retries.
- With an external IdP, the app can force a fresh login with OIDC's `max_age` parameter (or `prompt=login`) and verify the returned `auth_time`.

### 12.3 Adaptive or risk-based authentication

**The idea:** don't apply the same friction to every login. Score each login attempt's **risk** and respond proportionally.

**Risk signals, explained:**
- **New device or browser:** no "remember this device" cookie (§12.4), unfamiliar browser fingerprint.
- **Unusual country:** the user always logs in from one city, and this attempt comes from another continent.
- **Impossible travel:** a login from London at 10:00 and from Sydney at 10:20. Nobody can travel 17,000 km in 20 minutes, so one of them is likely an attacker (or a VPN).
- **IP reputation:** the IP belongs to a hosting provider (bots run in datacentres, real users don't), a Tor exit node, or appears on abuse lists.
- **Behaviour:** many failed attempts recently, or the password just appeared in a breach list.

**Responses:** low risk → log in silently (maybe skip MFA). Medium → require MFA. High → block, and email the user "we blocked a suspicious sign-in". The benefit: legitimate users get little friction; attackers hit walls. Auth providers (Auth0, Okta, Cognito, Entra ID) offer this out of the box.

### 12.4 "Remember me" and device trust

Two different features that are often confused:

**"Remember this device" (skip MFA here for 30 days):**
- After a successful MFA login, set a long-lived cookie containing a random device token: `HttpOnly` (JavaScript can't steal it), `Secure`, and ideally signed.
- Store a **hash** of the token server-side, linked to the user, so it can be **revoked** (the user's "Devices" page, or automatically after a password reset).
- On the next login, a valid device cookie means the password is still required, but MFA can be skipped.

**Persistent login ("keep me signed in"):**
- A long-lived **refresh token** (or remember-me token) that can repeatedly obtain new **short** sessions or access tokens.
- **Rotation with reuse detection**, step by step:
  1. Refresh token R1 is used → the server issues a new access token **and a new refresh token R2**, and marks R1 as used.
  2. Normally the client next uses R2, gets R3, and so on.
  3. If R1 is **ever presented again**, two parties hold copies of the chain: the legitimate client and a thief. The server can't tell which is which, so it **revokes the entire family** (R1, R2, R3…). Both must log in again; the thief can't, so the theft is contained.

### 12.5 Account recovery: the weakest link

Attackers don't attack your strongest door; they look for the weakest one. With passkeys and MFA in place, that's usually **"Forgot password?"** or a **human support agent**.

- **Reset links follow magic-link rules** (§4): random, hashed, single-use, short-lived.
- **After a reset, invalidate all existing sessions** (and refresh tokens). If an attacker was logged in, the reset kicks them out. **Notify the user** by email ("your password was changed; if this wasn't you…"), so a takeover doesn't go unnoticed.
- **Recovery must not bypass MFA.** If "forgot password" only needs email access and then lets you log straight in, your MFA is decorative: an attacker who owns the email owns the account. Require the second factor (or a recovery code) even after a reset, or add a delay with notifications for MFA removal.
- **Recovery codes and multiple passkeys/keys** give users a way back in without involving support at all.
- **Social engineering of support:** attackers call or chat with support pretending to be the user ("I lost my phone, please disable my 2FA"). Several major breaches started exactly this way. Give support staff strict verification procedures and tooling, and treat "remove MFA" as a high-risk action with delays and alerts.

### 12.6 Impersonation ("log in as user")

Support staff often need to see exactly what a customer sees in order to debug. Doing it carelessly creates a powerful backdoor; doing it well looks like this:

- **A separate, audited flow:** not "know the user's password". An explicit "impersonate" action requiring an **admin role** and **step-up authentication** (§12.2), ideally with a reason/ticket number.
- **Two identities in the session:** `actor: admin_7` (who is actually acting) and `subject: user_42` (whose view they're seeing). Every action is logged with **both**, so "user 42 deleted a note" can be distinguished from "admin 7, impersonating user 42, deleted a note". (OAuth's Token Exchange standard has an `act` claim for exactly this.)
- **Safety rails:** a visible banner ("You are viewing as Alice"), automatic expiry (e.g. 30 minutes), and blocking destructive or sensitive actions such as changing the password, viewing full payment details, or deleting the account. Some companies also require the customer's consent or notify them.

### 12.7 Zero trust

**The old model, "castle and moat":** a strong firewall/VPN at the network edge; anything inside the corporate network is trusted. Problems:
- Once an attacker gets **one** foothold inside (a phished laptop, a vulnerable server), they can move freely to everything else ("lateral movement").
- Modern work doesn't fit it: employees work from home, apps are SaaS in the cloud, contractors need partial access.

**Zero trust** (popularised by Google's **BeyondCorp** project after a sophisticated 2009 attack, and formalised in NIST SP 800-207):
- **Network location grants nothing.** Being on the office Wi-Fi or the VPN doesn't make a request trusted. Every request is authenticated and authorized as if it came from the public internet.
- **Decisions combine three things:** **user identity** (SSO + strong, preferably phishing-resistant MFA), **device identity and health** (is this a company-managed laptop? OS patched? disk encrypted? EDR running?) and **context** (time, location, sensitivity of the resource).
- **Services authenticate each other too:** mTLS and workload identity (§7.1, §10.4), so a compromised service can't freely call others.
- **Implementation:** an **identity-aware proxy** (e.g. Google IAP, Cloudflare Access, Pomerium) sits in front of each internal app and checks identity and device posture on every request, replacing the VPN. Credentials are **short-lived** everywhere.

---

## 13. Emerging methods

### Decentralized identifiers (DIDs)
Today your online identifiers are issued by someone else: your Google account, your email address at a provider, your phone number from a carrier. They can be suspended or taken away. A **DID** is an identifier **you** create and control, such as `did:web:alice.example` or `did:key:z6Mk…`. It resolves to a **DID document** listing your public keys, published on your own web domain or a distributed ledger. You prove control by signing with the matching private key. It's a W3C standard, with limited real-world adoption so far.

### Verifiable Credentials (VCs)
The digital equivalent of a physical card in your wallet. Three roles:
- **Issuer** (a university, a government) signs a credential: "Alice holds a BSc from X", "Alice was born on 1995-03-02".
- **Holder** (Alice) keeps it in a digital wallet app.
- **Verifier** (an employer, a bar, a website) checks the issuer's signature **without contacting the issuer**: the issuer never learns where you used it.

**Selective disclosure:** prove only what's needed. To show you're "over 18", the wallet can present a signed claim of just that fact, without revealing your birthdate, name or address. (Formats like SD-JWT make this possible.) Growing in government and education.

### Digital ID wallets / mobile driver's licences
Government ID on your phone, following the **ISO 18013-5 mDL** standard (designed for in-person checks, e.g. tapping your phone at airport security) and related standards for online use. Apple Wallet and Google Wallet support IDs in some regions. In the EU, **eIDAS 2.0** requires member states to offer every citizen an **EU Digital Identity Wallet**, which can hold ID, driving licences, diplomas and more, and can be used to log into public and private services.

### Sign-In with Ethereum (SIWE)
For web3 apps. The site presents a standard human-readable message (defined in **EIP-4361**) including its domain, your wallet address, a **nonce** and an expiry. Your crypto wallet signs it with your account's private key. The server recovers the signing address from the signature and checks it matches, along with the domain and nonce. It's challenge–response (§0.7) with your blockchain key as the credential. Phishing risk remains if users sign messages for look-alike domains, so wallets display the domain prominently.

### Behavioural / continuous authentication
Instead of a single yes/no at login, keep **re-scoring** throughout the session how likely it is that the same person is still there, from typing rhythm, mouse dynamics, how the phone is held, navigation patterns. If the score drops sharply (e.g. after a remote-access-trojan takeover in a banking session), require step-up auth or block the transaction. Used quietly in bank fraud systems today.

### Post-quantum cryptography
Today's public-key algorithms (RSA, elliptic curves like P-256 and Ed25519) rely on maths problems that a large enough **quantum computer** could solve efficiently (Shor's algorithm). No such computer exists yet, but:
- **"Harvest now, decrypt later":** attackers can record encrypted traffic today and decrypt it once quantum computers arrive. That's why **key exchange** (protecting secrecy) is being upgraded first.
- NIST published standards in 2024: **ML-KEM** (key exchange) and **ML-DSA** (signatures), plus SLH-DSA (hash-based signatures).
- Browsers and major sites already use **hybrid** key exchange (classical X25519 combined with ML-KEM) in TLS. Signatures, and therefore certificates, passkeys and JWT signing, will migrate more slowly, because signatures can't be forged retroactively: they only need to be safe once quantum computers actually exist.

---

## 14. Reading the comparison table

The deep-dive's big table compares every method on four columns. Here's how to read each column, and the rows that are less obvious.

**"Factor(s)":** which category of proof (§0.1). Notes:
- **TOTP app = "have"**: the code proves you possess the phone with the secret, even though you *type* it.
- **Face ID = "are (unlocks a have)"**: the biometric only gates the device key (§6.1).
- **Kerberos = "know (then tickets)"**: the password is used once; afterwards the tickets are bearer-like possessions.
- **SAML/OIDC SSO = "whatever the IdP uses"**: SSO protocols carry the *result* of authentication, so their strength is exactly the strength of the login at the IdP.

**"Phishing-resistant?":**
- ❌ for anything a **human types or reads** (passwords, all OTPs), because the human can be fooled about where they're typing.
- ⚠️ **push** (number matching helps a lot but a live proxy can show the number) and **magic links** (link and approval flows can be abused).
- ✅ for anything where **software checks the origin or presents a key automatically**: passkeys, security keys, smart cards, mTLS, SSH keys.
- ✅ **Kerberos within the domain**: tickets are issued for specific named services and protected by keys, so there's no prompt to fake. (It has other attacks, §9.1.)
- **n/a** for machine credentials (API keys, HMAC, workload identity): no human to trick. Their risk is **leakage** instead.

**"What the server stores":** this column shows what a **database breach** gives away:
- **Hash** (passwords, API keys, magic links, OTP codes): the attacker must crack it, and for random tokens that's impossible.
- **Encrypted, retrievable secret** (TOTP seeds, HMAC keys): must be decryptable by the server, so a full compromise (DB + encryption key) leaks working secrets.
- **Public key** (passkeys, security keys, SSH): **worthless to the attacker**. This is the core advantage of asymmetric methods.
- **Nothing** (Face ID): the biometric never leaves the device.
- **CA certificate / trust policy** (mTLS, smart cards, workload identity): the server only stores *who it trusts to vouch*, not per-user secrets.

**"Typical use":** where you'll actually meet each method; §15 explains the reasoning.

---

## 15. The reasoning behind "what to use when"

- **Consumer web app:** offer **passkeys** first (phishing-resistant, no password to leak, great UX via conditional UI), keep **email + password with TOTP** as a fallback for users and devices that aren't ready (or magic links for low-frequency apps), and offer **social login** because many users prefer one click. Store sessions in `HttpOnly` cookies so JavaScript (and XSS) can't read them. "Consider an auth provider" because recovery, MFA, rate limiting, breach checks and passkeys add up to a lot of security-critical code.

- **Consumer mobile app:** use OIDC/social login or passkeys (both work natively on iOS/Android). Keep a **short-lived access token** for API calls and a **refresh token in the Keychain/Keystore** gated by Face ID/fingerprint (§6.4), so re-opening the app is one glance, not a login.

- **B2B SaaS:** everything above, **plus** what enterprise IT departments require before buying: **SAML and OIDC SSO** (their employees log in with the company IdP), **SCIM** (automatic onboarding and, crucially, offboarding), **per-organisation policies** (e.g. "Acme requires SSO for all members"), and **audit logs**.

- **Internal company tools:** never create yet another password database. Put tools behind the company IdP via OIDC/SAML, or an identity-aware proxy (§12.7), so offboarding and MFA policies apply automatically.

- **Public developer API:** **API keys** are the simplest thing for developers calling your API for themselves: hash them, scope them (read-only vs write), and **prefix** them (e.g. `sk_live_…`) so secret scanners on GitHub can detect leaks. Use **OAuth 2.0** when **third-party apps** need to act on behalf of *your users*, so users never hand their password to those apps.

- **Webhooks you send:** HMAC-sign every payload, including a timestamp (§10.2), and publish clear verification docs with code samples, because your customers' security depends on them verifying correctly.

- **Microservices:** **mTLS via a service mesh** (identity and encryption with no app code), or **short-lived JWTs from workload identity** (§10.4). Avoid shared static secrets between services.

- **CI/CD deploying to the cloud:** **OIDC federation** (§10.4). No cloud keys stored in CI secrets, and a correctly scoped trust policy limits deploys to the right repo and branch.

- **A CLI tool:** the **OAuth device code flow**: the CLI shows a short code and a URL (`github.com/login/device`); you approve in your browser (with all its passkeys and SSO); the CLI polls until approved and receives tokens. This is how `gh auth login` works. Alternatively, the CLI opens a browser and receives the redirect on a temporary **`localhost`** port. Either way, the user never types a password into the terminal.

- **IoT devices:** devices have no user to type anything and may be physically exposed, so give each device its own **X.509 certificate** (ideally with the key in a secure element) and use **mTLS**. A compromised device can be revoked individually.

- **High-security / admin access:** **hardware security keys** or device-bound passkeys (AAL3, §12.1), **step-up** for dangerous actions, **short sessions**, and often device-posture checks. These accounts are what attackers target first.

---

## 16. The closing thought, unpacked

The deep-dive ends by reducing everything to three ideas. Here's why that classification is so useful:

**1. A shared secret** (passwords, OTP seeds, API keys, HMAC keys).
Both sides know the secret, so it can leak from **either** side: a phished user or a breached server. It must also be stored somewhere retrievable or verifiable, and humans can be tricked into revealing typed secrets. Most security engineering around these methods (hashing, salting, rate limits, rotation, breach checks) is *damage control* for that fundamental weakness.

**2. A key pair** (passkeys, security keys, SSH, mTLS, smart cards, Face ID-protected keys).
Only you hold the private key, often inside hardware that won't release it. The server stores a public key that's useless to a thief. Proofs are challenge–response, so recordings can't be replayed, and origin binding makes phishing fail. This is what "phishing-resistant" and "breach-resistant" ultimately mean.

**3. A trusted third party vouching for you** (SSO, SAML, OIDC, Kerberos, workload identity).
Instead of every app verifying credentials, one specialised system does it well, and others trust its signed statements. That centralises policy (MFA, offboarding, device checks), and lets apps avoid handling credentials at all. Note that the third party itself still needs idea 1 or 2 at its core.

**The direction of travel:** from (1) to (2) and (3). From secrets that people type and servers store, to **keys that never leave hardware**; from **many passwords** to **one strong identity, federated everywhere**. And biometrics like Face ID fit into this picture not as a replacement for keys, but as the **convenient, local way to unlock them**.

---

## 17. Check your understanding

Try answering before expanding each answer.

<details>
<summary>1. Why can't a server store a TOTP secret as a bcrypt hash, the way it stores passwords?</summary>

Because the server must **compute** `HMAC(secret, time)` to know what the current code should be. A hash is one-way, so the original secret would be unrecoverable. TOTP secrets must be **encrypted** (reversibly) at rest instead.
</details>

<details>
<summary>2. An attacker clones your bank's login page at a look-alike domain. Which of these stop them: SMS OTP, TOTP, push with number matching, passkey?</summary>

Only the **passkey** stops it by construction. The browser has no passkey for the fake domain, and it writes the real origin into the signed data. All code-based methods can be relayed in real time; number matching makes it harder but a live proxy can show the number to the victim.
</details>

<details>
<summary>3. Does the server learn anything about your face when you use Face ID with a passkey?</summary>

No. It receives a signature and a UV ("user verified") flag. It can't tell whether you used Face ID, Touch ID or a PIN. The face template never leaves the Secure Enclave.
</details>

<details>
<summary>4. Why is the "boolean gate" use of Face ID weak, and what replaces it?</summary>

The app trusts a `true/false` result inside its own process, which can be hooked on a compromised device, and the server gets no proof. The **key gate** generates a Secure Enclave key that can only sign after Face ID; the server verifies the signature over its own fresh challenge.
</details>

<details>
<summary>5. You log into App A via SSO, then log out of App A. Why does clicking "Log in" on App A log you straight back in without a password?</summary>

Logging out of App A deleted only **App A's** session. The **IdP's** session cookie is still valid, so the IdP skips the prompt when App A redirects to it. Logging out properly requires ending the IdP session too (and Single Logout to reach other apps).
</details>

<details>
<summary>6. What does SCIM give you that JIT provisioning doesn't?</summary>

**Deprovisioning.** JIT creates accounts on first login but never learns when someone leaves. With SCIM the IdP actively calls your API to deactivate users (and update groups) the moment they're offboarded.
</details>

<details>
<summary>7. In Kerberos, how does the fileserver know a ticket is genuine without contacting the KDC?</summary>

The service ticket is encrypted with the **fileserver's own key**, which only the fileserver and the KDC know. If it decrypts correctly and contains a valid structure, only the KDC could have made it.
</details>

<details>
<summary>8. Your webhook signature check always fails, although the secret is right. What's the most likely cause?</summary>

You're computing the HMAC over **re-serialised JSON** (`JSON.stringify(req.body)`) instead of the **raw request bytes**. Any difference in whitespace or key order changes the HMAC.
</details>

<details>
<summary>9. Why must a GitHub-to-AWS OIDC trust policy restrict the `sub` claim?</summary>

GitHub signs tokens for **every** repository's workflows. If the policy only checks the issuer and audience, any GitHub user's workflow could assume your AWS role. Restricting `sub` to `repo:acme/api:ref:refs/heads/main` (or an environment) limits it to your repo and branch.
</details>

<details>
<summary>10. What happens in refresh-token rotation when an old, already-used refresh token is presented again, and why?</summary>

The server **revokes the whole token family**. A reused old token means two parties have copies of the chain (the real client and a thief); the server can't tell which is legitimate, so it forces both to log in again, which only the real user can do.
</details>

<details>
<summary>11. Why is a leaked database of passkey public keys harmless, while a leaked database of HMAC secrets is a disaster?</summary>

Public keys can only **verify** signatures; creating one requires the private key, which never left users' devices. HMAC secrets are **symmetric**: whoever holds the secret can produce valid signatures exactly as the real client does.
</details>

<details>
<summary>12. Which layer (credential, protocol or architecture) is each of these: SAML, Touch ID, step-up authentication, a YubiKey, SSO, mTLS?</summary>

SAML: **protocol**. Touch ID: **credential** (a local unlock of a key credential). Step-up authentication: **architecture/policy**. YubiKey: **credential** (an authenticator holding keys). SSO: **architecture**. mTLS: **protocol** (with a certificate + private key as the credential).
</details>
