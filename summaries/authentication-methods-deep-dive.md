# Authentication Methods: The Full Landscape

A companion to [authentication-and-authorization.md](authentication-and-authorization.md). That file covers the four methods from the video: **stateful sessions, stateless JWTs, API keys, and OAuth 2.0 / OpenID Connect**. This file covers **everything else** you'll meet as a backend engineer: passwords done right, one-time codes, magic links, passkeys, biometrics (including how Apple's Face ID actually works), certificates, Single Sign-On, SAML, Kerberos, LDAP, machine-to-machine signing, workload identity, and the patterns layered on top.

> This file is not based on a video. It's written from first principles to fill in the rest of the topic.

---

## Table of contents

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
14. [Comparison of every method](#14-comparison-of-every-method)
15. [What to use when](#15-what-to-use-when)
16. [Glossary](#16-glossary)

---

## 1. The map: three layers of authentication

"Authentication methods" mixes together things that live at **different layers**. Questions like "is SSO better than Face ID?" or "should I use OAuth or passkeys?" are confusing because they compare things on different layers. Separate them first:

```
┌────────────────────────────────────────────────────────────────────────────────────┐
│ 3. ARCHITECTURE: how many systems trust one login?                                 │
│    SSO, federation, MFA, step-up auth, risk-based auth, zero trust                 │
├────────────────────────────────────────────────────────────────────────────────────┤
│ 2. PROTOCOL: how does the proof travel to the server, and in what format?          │
│    HTML login form, HTTP Basic, Bearer, SAML, OIDC, Kerberos, LDAP, WebAuthn, mTLS │
├────────────────────────────────────────────────────────────────────────────────────┤
│ 1. CREDENTIAL: what proves it's you?                                               │
│    password, OTP code, private key, certificate, face, fingerprint, security key   │
└────────────────────────────────────────────────────────────────────────────────────┘
```

A real login uses all three. Example: *"Sign in to your company's Slack"*:
- **Layer 3**: SSO. Slack trusts your company's identity provider (Okta).
- **Layer 2**: SAML or OIDC carries "this is alice@company.com" from Okta to Slack.
- **Layer 1**: at Okta you used a **passkey**, unlocked on your Mac by **Touch ID**.

### Things that are often confused with authentication

| Thing | What it actually does |
|---|---|
| **OAuth 2.0** | *Authorization* (delegated access). OIDC is the authentication layer on top. |
| **Face ID / Touch ID** | **Local** verification on your device. Your face never reaches the server; it unlocks a key that does. (§6) |
| **SSO** | An *architecture* (log in once, use many apps), not a protocol. It's built *with* SAML, OIDC, Kerberos… (§8) |
| **TLS / HTTPS** | Authenticates the **server** to you (the padlock), not you to the server. *Mutual* TLS adds the client side. (§7) |
| **CAPTCHA** | Proves you're **a human**, not **which** human. |
| **Encryption** | Keeps data secret. Doesn't by itself say who sent it. |

---

## 2. Password-based methods

The first file covered the classic **HTML form login → session/JWT** flow. Here are the other password-based methods and the modern rules for handling passwords.

### 2.1 HTTP Basic authentication

The oldest built-in HTTP scheme. The client sends the username and password **on every request**:

```
GET /admin/metrics HTTP/1.1
Authorization: Basic YWxpY2U6czNjcjN0      ← base64("alice:s3cr3t")
```

If credentials are missing, the server replies with a **challenge**, and browsers show their native login popup:

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="admin area"
```

| Pros | Cons |
|---|---|
| Built into every HTTP client and browser; zero code on the client | Base64 is **not encryption**, so it's plaintext without HTTPS |
| Fine for internal tools, quick admin endpoints, simple service calls | Password sent on **every** request (more chances to leak) |
| | No logout (the browser caches it), no MFA, ugly UI |

**Use it only over HTTPS**, and only for internal or low-stakes endpoints (e.g. protecting a metrics page or a staging site).

### 2.2 HTTP Digest authentication

An attempt to fix Basic: the server sends a random **nonce**, and the client sends `hash(username:realm:password + nonce + …)` instead of the password.

It's mostly **obsolete**. It traditionally relies on MD5, and the server must store a **password-equivalent** value (`MD5(user:realm:password)`). That means it can't use salted slow hashes, and a DB leak is as bad as a plaintext leak. HTTPS + Basic, or a proper login flow, replaced it.

### 2.3 Modern password rules (what NIST recommends)

NIST SP 800-63B is the reference most security teams follow. Its guidance overturned many "traditional" password rules:

| Old practice | Current guidance | Why |
|---|---|---|
| Require uppercase + number + symbol | **Don't impose composition rules** | People produce `Password1!`, which is predictable |
| Short minimum (6–8 chars) | **Length matters most.** At least 8; NIST's latest revision recommends **15** when the password is the only factor. Allow at least 64. | Length adds far more entropy than symbols |
| Force a change every 90 days | **Don't force periodic rotation.** Only force a change on evidence of compromise. | Forced rotation produces `Summer2026!` → `Autumn2026!` |
| Block paste | **Allow paste** and password managers | Password managers generate strong, unique passwords |
| Security questions ("first pet?") | **Don't use them** | Answers are guessable or findable on social media |
| (nothing) | **Check against breached-password lists** | Stops reuse of passwords already in attackers' wordlists |

#### Breached-password checks without leaking the password

The *Have I Been Pwned* "Pwned Passwords" API uses **k-anonymity**:

1. Compute `SHA-1(password)`, e.g. `5BAA61E4C9B93F3F0682250B6CF8331B7EE68FD8`.
2. Send only the **first 5 characters** (`5BAA6`) to the API.
3. The API returns every leaked hash suffix that starts with that prefix (hundreds of them).
4. Check locally whether your suffix is in the list.

The API never learns your password or even its full hash.

### 2.4 Password-authenticated key exchange (PAKE)

In a normal login, the **server sees the plaintext password** for a moment (before hashing it). A **PAKE** protocol such as **SRP** (used by e.g. 1Password) or the newer **OPAQUE** proves knowledge of the password through a cryptographic exchange, so the server **never** sees it, even at login. It's rare in typical web apps but worth knowing exists.

### 2.5 The main threats to passwords

| Attack | What it is | Defence |
|---|---|---|
| **Brute force** | Try many passwords against one account | Rate limiting, lockout/backoff, slow hashes |
| **Credential stuffing** | Replay username/password pairs leaked from *other* sites | Breached-password checks, MFA, bot detection |
| **Password spraying** | Try *one* common password (`Winter2026!`) against *many* accounts, which avoids per-account lockouts | Rate limit per IP and globally, ban common passwords |
| **Phishing** | A fake login page collects the password | MFA, ideally **phishing-resistant** (passkeys, §5) |
| **Database breach** | Attacker gets the password table | Salted Argon2id/bcrypt (covered in the first file) |

---

## 3. One-time passwords (OTP)

A code valid for **one use** or a **short time**. Usually a **second factor**, sometimes the only one (passwordless).

### 3.1 HOTP and TOTP: authenticator apps

The 6-digit codes in Google Authenticator, Authy, 1Password and similar apps.

**Setup (once):**
1. The server generates a random **shared secret** (e.g. 20 bytes).
2. It shows a **QR code** encoding `otpauth://totp/Notes:alice@example.com?secret=JBSWY3DPEHPK3PXP&issuer=Notes`.
3. The app scans it and stores the secret. **Now both sides have the same secret.**
4. The user types one code to confirm setup worked.

**Every login:** both sides compute the code independently from the secret plus a moving factor, and the server compares them.

| | HOTP (RFC 4226) | TOTP (RFC 6238) |
|---|---|---|
| Moving factor | A **counter** incremented on each use | **Time**: `floor(unix_time / 30)` |
| Weakness | Counter can drift out of sync | Needs reasonably synced clocks (servers accept ±1 step) |
| Used by | Some hardware tokens | Almost every authenticator app |

The algorithm is short. It's essentially "HMAC the time, keep 6 digits":

```js
import crypto from "node:crypto";

function hotp(secret, counter, digits = 6) {
  const msg = Buffer.alloc(8);
  msg.writeBigUInt64BE(BigInt(counter));                              // counter as 8-byte big-endian
  const hmac = crypto.createHmac("sha1", secret).update(msg).digest(); // 20 bytes
  const offset = hmac[hmac.length - 1] & 0x0f;                         // "dynamic truncation"
  const code = (hmac.readUInt32BE(offset) & 0x7fffffff) % 10 ** digits;
  return code.toString().padStart(digits, "0");
}

const totp = (secret, timeMs = Date.now()) => hotp(secret, Math.floor(timeMs / 1000 / 30));

// Accept the current 30s window plus one either side, to allow for clock drift
function verifyTotp(secret, input) {
  if (!/^\d{6}$/.test(input)) return false;
  return [-1, 0, 1].some((w) =>
    crypto.timingSafeEqual(Buffer.from(totp(secret, Date.now() + w * 30_000)), Buffer.from(input))
  );
}
```

(`secret` here is the raw bytes, i.e. the base32 secret from the QR code, decoded.)

**Server-side gotchas:**
- **The TOTP secret can't be hashed.** The server must compute codes from it, so **encrypt it at rest** (e.g. with a key in a KMS). That's different from passwords.
- **Block replay:** remember the last accepted time-step per user and reject the same code twice.
- **Rate-limit** code attempts. 6 digits is only 1,000,000 possibilities.
- Issue **backup / recovery codes** at setup: about 10 single-use random codes, stored **hashed**, for when the phone is lost.

### 3.2 SMS and email OTP

The server generates a random code, stores its hash with a short expiry (5–10 min), and sends it by SMS or email.

| SMS OTP weaknesses | |
|---|---|
| **SIM swapping** | Attacker convinces your carrier to move your number to their SIM, then receives your codes. This has been behind several high-profile account takeovers. |
| **Network interception** | Weaknesses in the telephone signalling network (SS7) allow interception |
| **Phishing** | A fake site asks for the code and replays it immediately |
| **Cost and delivery** | SMS costs money per message and sometimes doesn't arrive |

NIST classes SMS as a **"restricted"** authenticator: allowed, but discouraged. **SMS 2FA is far better than no 2FA**, but authenticator apps or passkeys are better still. Email OTP is only as secure as the user's email account.

### 3.3 Push-notification approval

Apps like Duo or Microsoft Authenticator show *"Are you trying to sign in? Approve / Deny"*. It's convenient, but vulnerable to **MFA fatigue (push bombing)**: the attacker, who already has the password, triggers dozens of prompts at 2 a.m. until the victim taps *Approve* to make it stop. This was a key step in Uber's 2022 breach.

**Fix: number matching.** The login screen shows a number (e.g. `47`) that the user must type into the app, which proves they're actually looking at the login screen. Add context too (location, app name).

### 3.4 Hardware OTP tokens

Small key-fob devices (e.g. RSA SecurID) that display a TOTP-style code. They're still found in banks and older enterprises, and are mostly being replaced by authenticator apps and FIDO security keys.

> **The key limitation of every OTP type:** a code can be **phished in real time**. A fake site collects the password *and* the code and immediately replays both to the real site. Only methods bound to the website's origin (passkeys, security keys, client certificates) stop this. See §5.

---

## 4. Magic links

**Passwordless via email.** The user enters their email and gets a link like `https://notes.com/auth/verify?token=k3J9…`; clicking it logs them in. Used by Slack, Notion, Medium and many others.

```
1. User submits email ─────────────────▶ Server: generate random token (32+ bytes)
                                          store hash(token), userId, expiresAt (10–15 min), used=false
2. Server emails link containing token
3. User clicks link ───────────────────▶ Server: hash(token) → look up → not expired? not used?
                                          mark used, create session, redirect to app
```

**Implementation rules:**
- **Cryptographically random** token, **hashed** in the DB (like an API key), **single-use**, **short expiry**.
- **Email security scanners** (in corporate mail systems) often *visit links automatically* to check for malware, which would "use up" a single-use token. Fix: the link opens a page with a **"Log in" button** that does a `POST`; only the POST consumes the token.
- Consider binding the link to the **browser that requested it** (e.g. a cookie set at step 1). Otherwise, opening the email on your phone logs in the *phone*, which can be confusing and is exploitable by an attacker who triggers the email.
- Same anti-enumeration rule as before: always respond *"If an account exists, we've sent a link."*

| Pros | Cons |
|---|---|
| No password to remember, leak, or reuse | Security = the security of the user's **email account** |
| Simple to build | Context switch to the inbox, delays, spam folders |
| Doubles as email verification | Phishable in some setups; not great for frequent logins |

**Email OTP codes** (e.g. *"Your code is 381 552"*) are the same idea without a link. They're friendlier on mobile, where switching apps to click a link is awkward.

---

## 5. Passkeys, WebAuthn and FIDO2

This is the most important modern method and the one most likely to replace passwords for consumers. It builds directly on the asymmetric cryptography from the first file.

### 5.1 The names

| Term | What it is |
|---|---|
| **FIDO Alliance** | Industry body (Apple, Google, Microsoft…) that writes the standards |
| **WebAuthn** | The **W3C browser API** (`navigator.credentials.create/get`) that websites call |
| **CTAP2** | The protocol between the browser/OS and an **authenticator** (phone, security key) |
| **FIDO2** | WebAuthn + CTAP2 together |
| **Passkey** | The consumer-friendly name for a FIDO2 credential, usually **synced** across your devices |
| **Authenticator** | The thing holding the private key. **Platform**: built into your device (Face ID, Touch ID, Windows Hello, Android). **Roaming**: a separate device (YubiKey over USB/NFC/Bluetooth, or your phone). |

### 5.2 The core idea

Instead of a **shared secret** (a password that both you and the server know), you have a **key pair**, created **separately for each website**:

- The **private key** stays on your device (or in your encrypted password manager or iCloud Keychain). **It is never sent anywhere.**
- The server stores only the **public key**. If the database leaks, attackers get public keys, which are useless to them.
- To log in, the server sends a random **challenge**; your device **signs** it with the private key after you approve with Face ID, a fingerprint or a PIN; the server **verifies** the signature with the public key.

### 5.3 Registration ("create a passkey")

```
 Browser + authenticator                                 Server (relying party, "RP")
   │  "Create passkey"  ───────────────────────────────────▶ │ generate random challenge, store in session
   │ ◀─────── options: challenge, rp.id="notes.com", user.id, allowed algorithms
   │ navigator.credentials.create(options)
   │   └─▶ OS prompt: Face ID / Touch ID / PIN               │
   │   └─▶ authenticator creates NEW key pair for notes.com  │
   │       signs {challenge, origin} (+ optional attestation)│
   │  credentialId + publicKey + clientDataJSON ───────────▶ │ verify: challenge matches? origin == https://notes.com?
   │                                                         │ store: userId, credentialId, publicKey, signCount
```

### 5.4 Authentication ("sign in with passkey")

```
 Browser + authenticator                                 Server
   │  "Sign in" ───────────────────────────────────────────▶ │ new random challenge
   │ ◀─────────────────────────────────── challenge, rpId    │
   │ navigator.credentials.get(options)                      │
   │   └─▶ OS shows matching passkeys for notes.com          │
   │   └─▶ Face ID ✔ → private key signs:                    │
   │       authenticatorData ‖ SHA-256(clientDataJSON)       │
   │       (clientDataJSON contains the challenge + ORIGIN)  │
   │  credentialId, signature, authenticatorData, clientDataJSON ─▶ │
   │                                                         │ 1. look up public key by credentialId
   │                                                         │ 2. check challenge, origin, rpIdHash
   │                                                         │ 3. check flags: UP (user present), UV (user verified)
   │                                                         │ 4. verify signature with public key
   │                                                         │ 5. create session ✔
```

The browser side:

```js
// Registration (helper functions to convert base64url ↔ ArrayBuffer omitted)
const credential = await navigator.credentials.create({
  publicKey: {
    challenge: fromBase64url(opts.challenge),            // random bytes from the server
    rp: { id: "notes.com", name: "Notes" },
    user: { id: fromBase64url(opts.userHandle), name: "alice@example.com", displayName: "Alice" },
    pubKeyCredParams: [{ type: "public-key", alg: -7 },   // ES256
                       { type: "public-key", alg: -257 }], // RS256
    authenticatorSelection: { residentKey: "required", userVerification: "required" },
  },
});

// Login
const assertion = await navigator.credentials.get({
  publicKey: { challenge: fromBase64url(opts.challenge), rpId: "notes.com", userVerification: "required" },
});
// POST credential / assertion to the server for verification
```

On the server, **use a library**: SimpleWebAuthn (Node), go-webauthn (Go), py_webauthn (Python), webauthn4j (Java). Parsing CBOR/COSE structures and checking every field correctly is easy to get wrong.

### 5.5 Why passkeys are phishing-resistant

This is their key property. The **browser** (not the user) puts the real **origin** (`https://notes.com`) into the signed data, and the authenticator only offers keys registered for that domain.

- On `https://notes-login.evil.com`, your device has **no passkey** for that domain, so there's nothing to steal or sign.
- Even if an attacker relayed a challenge, the signature would contain `origin: evil.com`, and the real server rejects it.

A password or OTP is something the *user* types, and users can be tricked into typing it anywhere. A passkey can't be typed into the wrong site.

### 5.6 Synced vs device-bound passkeys

| | Synced passkeys | Device-bound passkeys |
|---|---|---|
| Where the private key lives | Password manager / iCloud Keychain / Google Password Manager, **end-to-end encrypted** and synced | Inside one piece of hardware (YubiKey, TPM), **can't be exported** |
| Lose your phone? | Still on your other devices / restore from account | Gone; you need a backup key |
| Security | Very high (depends on your cloud account security) | Highest |
| Use | Consumers | Admins, high-security roles, regulated industries |

**Cross-device sign-in:** on a laptop without your passkey, choose *"Use a phone"*. A QR code appears; your phone scans it, a **Bluetooth proximity check** confirms the phone is physically nearby (so a remote attacker can't relay it), and the phone signs the challenge.

**Conditional UI:** passkeys can appear in the browser's **autofill dropdown** on the username field, so users don't need to know what a passkey is.

### 5.7 Other details

- **User presence (UP)** = someone touched or approved. **User verification (UV)** = the authenticator verified *who* (biometric or PIN). Require UV if the passkey is the only factor.
- A passkey with UV is effectively **MFA in one step**: *something you have* (the device holding the key) + *something you are/know* (Face ID or PIN to unlock it).
- **Signature counter:** device-bound keys increment a counter. If the server sees it go *backwards*, the key may have been cloned. Synced passkeys usually report `0`, so don't rely on it.
- **Attestation:** optionally, the authenticator proves its make and model. Enterprises use it ("only company-issued YubiKeys"); consumer apps usually skip it.
- **Account recovery** is the hard part: let users register multiple passkeys, and keep a secure fallback (§12.5).

---

## 6. Biometrics and Apple Face ID

### 6.1 The most important fact: your face never goes to the server

In modern consumer systems, **biometrics are a local unlock mechanism**, not something sent to a server:

```
 Your face ──▶ TrueDepth camera ──▶ Secure Enclave compares with enrolled template
                                         │ match?
                                         ▼
                              unlocks a PRIVATE KEY stored in the Secure Enclave
                                         │
                                         ▼
                       key signs the server's challenge  ──▶  server verifies signature
                                                              with the PUBLIC KEY
```

The server only ever sees **a signature**. It doesn't know whether you used Face ID, Touch ID, or a PIN (the WebAuthn `UV` flag just says "the user was verified"). So from the server's point of view, Face ID is not really "something you are". It's a gate on "something you have".

This is deliberate. **A password can be changed after a breach; your face can't.** A central database of face templates would be a permanent, catastrophic breach target. Keeping templates in tamper-resistant hardware on each device avoids that.

### 6.2 How Face ID works

1. **Sensing (TrueDepth camera system):**
   - A **flood illuminator** lights your face with invisible **infrared** light, so it works in the dark.
   - A **dot projector** projects **more than 30,000 invisible infrared dots** onto your face.
   - An **infrared camera** reads how the dot pattern lands on your face, producing a **3D depth map** plus a 2D infrared image.
2. **Processing:** a neural network running on the device converts the depth map and IR image into a **mathematical representation** of your face.
3. **Matching:** that representation is compared with your **enrolled** template **inside the Secure Enclave**. The main processor and iOS never see the raw template.
4. **Attention awareness:** by default, Face ID requires your **eyes open and looking at the device**, so someone can't unlock your phone while you sleep.
5. **Anti-spoofing:** separate neural networks are trained to detect photos, screens and masks. Because it uses *depth*, a flat photo doesn't work.
6. **Adaptation:** it gradually updates the template as your appearance changes (beard, glasses, makeup).

**Apple's published figures:**

| | False match probability |
|---|---|
| **Face ID** | Less than **1 in 1,000,000** for a random person (single enrolled appearance) |
| **Touch ID** | About **1 in 50,000** for a random fingerprint |

The probability is higher for **twins, siblings who look alike, and children under 13**; Apple suggests using a passcode in those cases.

**Privacy and fallbacks:**
- Face data **stays in the Secure Enclave**, is **not backed up to iCloud**, and **is never sent to Apple**.
- Apps **never get access to face data**. They only learn "success" or "failure".
- The **passcode is always required** in certain cases, e.g. after a **restart**, after **5 failed** Face ID attempts, or if the device hasn't been unlocked for **48 hours**. The passcode is the root of trust; Face ID is a convenience layer over it.

### 6.3 The Secure Enclave

A **separate security coprocessor** inside Apple chips with its own boot process and encrypted memory, isolated from the main processor. Even if iOS is compromised, it can't read keys or biometric templates out of the Secure Enclave. It can only ask the Secure Enclave to *use* a key, and only after the required check (Face ID / passcode) passes.

Equivalents elsewhere: **TPM** chips (Windows/PCs), **Titan M** / **StrongBox** (Android), **secure elements** in YubiKeys and smart cards.

### 6.4 How an app should use Face ID: two approaches

**❌ The boolean gate (weak).** Ask iOS "did Face ID pass?" and trust the answer:

```swift
let context = LAContext()
context.evaluatePolicy(.deviceOwnerAuthenticationWithBiometrics,
                       localizedReason: "Log in to Notes") { success, error in
    if success { showNotes() }   // on a jailbroken device, an attacker can hook this to always return true
}
```

**✅ The key gate (strong).** Create a key that can **only be used after Face ID**, register its public key with your server, and sign the server's challenge at login. If Face ID didn't pass, the signature doesn't exist, and there's no boolean to fake:

```swift
let access = SecAccessControlCreateWithFlags(nil,
    kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
    [.privateKeyUsage, .biometryCurrentSet],   // invalidated if Face ID enrolment changes
    nil)!

let attributes: [String: Any] = [
    kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,   // P-256, the only type the Secure Enclave supports
    kSecAttrKeySizeInBits as String: 256,
    kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,      // key is generated INSIDE the Secure Enclave
    kSecPrivateKeyAttrs as String: [kSecAttrIsPermanent as String: true,
                                    kSecAttrAccessControl as String: access],
]
let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, nil)!
// Once: send SecKeyCopyPublicKey(privateKey) to your server.
// At login: iOS shows Face ID, and the signature is only produced if it passes.
let signature = SecKeyCreateSignature(privateKey, .ecdsaSignatureMessageX962SHA256,
                                      serverChallenge as CFData, nil)
```

`.biometryCurrentSet` means that if someone adds a new face or fingerprint, the key becomes unusable, so a thief who learns the passcode can't enrol their own face to unlock it.

Equivalents: **Android** `BiometricPrompt` with a `CryptoObject` and a Keystore key created with `setUserAuthenticationRequired(true)`. **On the web**, you don't write any of this: **WebAuthn/passkeys** use Face ID, Touch ID or Windows Hello as the platform authenticator automatically.

A common mobile pattern: after the first password login, store a **refresh token in the Keychain** protected by biometrics. On the next app launch, Face ID releases the token and the app silently gets a new session.

### 6.5 Other biometrics

| Type | How it works | Where |
|---|---|---|
| **Fingerprint** | Capacitive, optical or ultrasonic sensor maps ridges | Touch ID, Android phones, laptops |
| **Windows Hello face** | Infrared camera (works in the dark, resists photos); keys in the TPM | Windows laptops |
| **Iris** | Near-infrared image of iris patterns | High-security access, some border control |
| **Voice** | Voiceprint analysis | Phone banking. Increasingly weak against **AI voice cloning**. |
| **Palm / vein** | Infrared image of vein patterns under the skin | Payments (e.g. Amazon One), Japanese ATMs |
| **Behavioural** | *How* you type, move the mouse, hold the phone, walk | Passive fraud detection in banking; **continuous authentication** |

### 6.6 Biometric concepts worth knowing

- **FAR / FRR:** *False Acceptance Rate* (letting the wrong person in) vs *False Rejection Rate* (locking the right person out). Tightening one loosens the other; every system picks a threshold.
- **Liveness detection / presentation attack detection:** telling a real person from a photo, video, mask, or deepfake. It's essential, and harder for systems that use ordinary 2D cameras.
- **Template protection:** store a derived mathematical template, not the image, in secure hardware.
- **Biometrics are identifiers, not secrets.** Your face is on every photo of you, and you leave fingerprints on every glass. That's why they work best as a *local unlock with liveness checks*, not as a password.
- **Legal:** biometric data is "special category" data under **GDPR**, and laws like Illinois' **BIPA** require explicit consent, so collecting it server-side carries serious legal liability. Another reason to let the device handle it.

---

## 7. Certificate and key-based authentication

### 7.1 Mutual TLS (mTLS)

Normal TLS (HTTPS): **the server proves its identity** with a certificate; the client stays anonymous at the TLS level. **Mutual TLS: both sides present certificates.**

```
 Client (has cert + private key)                     Server
   │ ClientHello ─────────────────────────────────▶  │
   │ ◀── ServerHello + server certificate + "CertificateRequest: send me yours"
   │ verify server cert against a trusted CA         │
   │ client certificate + CertificateVerify ───────▶ │ verify client cert against a trusted CA
   │   (signature proving it holds the private key)  │ identity = certificate subject
   │ ◀════════════ encrypted channel, both sides authenticated ════════════▶ │
```

- **Identity** = the certificate's subject, e.g. `CN=payments-service` or a SPIFFE ID like `spiffe://prod/ns/payments/sa/api`.
- **Strengths:** authentication happens **before any HTTP request** is processed; there's no shared secret to leak; it's phishing-resistant.
- **Costs:** you need a **certificate authority (CA)**, and certificates must be issued, rotated and revoked. That's operationally heavy without automation.
- **Where it's used:**
  - **Service-to-service** in microservices. **Service meshes** (Istio, Linkerd) do it automatically with short-lived certificates.
  - **Open banking** and financial APIs.
  - **IoT devices**: each device gets a certificate at manufacture.
  - **Zero-trust networks** (§12.7).

### 7.2 SSH keys

How you log into servers and push to GitHub. It's the same challenge–response idea as passkeys, and it predates them:

1. `ssh-keygen` creates a key pair; the public key goes into the server's `~/.ssh/authorized_keys` (or your GitHub settings).
2. On connect, the server challenges the client to **sign** session data with the private key.
3. The server verifies the signature with the stored public key.

Larger organisations use **SSH certificates**: a CA signs short-lived certificates instead of copying public keys to every server. Hardware security keys can hold SSH keys too (`ed25519-sk` key type).

### 7.3 Smart cards and PIV/CAC

A card with a chip (secure element) holding a private key and certificate, unlocked by a PIN, e.g. US government **PIV/CAC** cards and many national ID cards. It's "something you have" + "something you know", and works for computer login, VPNs, document signing and building access.

### 7.4 Hardware security keys

Devices like YubiKey or Google Titan. They support **FIDO2/WebAuthn** (passkeys, §5), older **FIDO U2F** (second factor only), and often OTP, smart-card (PIV) and OpenPGP modes. They're the **strongest widely available factor** and are recommended for admins, developers with production access, and anyone likely to be targeted.

---

## 8. Single Sign-On (SSO)

### 8.1 What it is

> **SSO:** log in **once** to a central **Identity Provider (IdP)**, then access **many applications** without logging in again.

| Term | Meaning | Examples |
|---|---|---|
| **IdP** (Identity Provider) | The system that authenticates users and vouches for them | Okta, Microsoft Entra ID (formerly Azure AD), Google Workspace, Keycloak, Auth0, Ping, JumpCloud |
| **SP** (Service Provider) / **RP** (Relying Party) | The app that trusts the IdP | Slack, GitHub, Salesforce, your SaaS app |
| **Federation** | A trust relationship between an IdP and SPs, often across organisations | "Our Okta is trusted by Slack" |

### 8.2 Why companies want it

- **One identity per employee:** one password + MFA instead of 50 accounts.
- **Instant offboarding:** disable the user in the IdP and they lose access to *every* app. Without SSO, someone has to remember to remove them from each app, and forgotten accounts are a classic breach path.
- **Central policy:** enforce MFA, passkeys, device checks and session lengths **in one place**.
- **Audit:** one log of who logged into what and when.
- **Fewer passwords** means less phishing surface and fewer help-desk resets.

### 8.3 How it works under the hood

SSO is **not** one shared session across apps. There are **separate sessions**:
- the **IdP's own session** (a cookie on the IdP's domain), and
- **each app's local session** (a cookie on each app's domain).

The "single" sign-on happens because the second app's redirect finds the **IdP session still alive**, so the IdP skips the login prompt.

```
 Browser                App A (notes.com)          IdP (login.company.com)          App B (tasks.com)
   │ GET notes.com ──────────▶ │ no local session                │                               │
   │ ◀── redirect to IdP ───── │                                 │                               │
   │ GET /authorize?app=A ─────────────────────────────────────▶ │ no IdP session → login page   │
   │ POST password + MFA ──────────────────────────────────────▶ │ ✔ create IdP session          │
   │ ◀── Set-Cookie: idp_sid (login.company.com) + redirect to A with code/assertion             │
   │ code/assertion ─────────▶ │ verify → Set-Cookie: a_sid      │                               │
   │                           │                                 │                               │
   │ GET tasks.com ─────────────────────────────────────────────────────────────────────────────▶│ no local session
   │ ◀── redirect to IdP ────────────────────────────────────────────────────────────────────────│
   │ GET /authorize?app=B   Cookie: idp_sid ───────────────────▶ │ IdP session exists → NO PROMPT│
   │ ◀── redirect to B with code/assertion ───────────────────── │                               │
   │ code/assertion ────────────────────────────────────────────────────────────────────────────▶│ verify → Set-Cookie: b_sid
```

The user typed their password once; App B logged them in with only a few redirects.

### 8.4 SSO protocols

| Protocol | Era | Format | Typical use |
|---|---|---|---|
| **SAML 2.0** | 2005 | XML, signed | **Enterprise SSO** into SaaS apps. Still dominant in B2B. |
| **OpenID Connect** | 2014 | JSON / JWT | Modern SSO, social login, mobile apps, SPAs (covered in the first file) |
| **Kerberos** | 1980s | Binary tickets | SSO **inside** a corporate Windows/Active Directory network (§9.1) |
| **CAS** | 2000s | XML / JSON | Universities |
| **WS-Federation** | 2000s | XML | Legacy Microsoft (ADFS) |

### 8.5 SAML 2.0 in depth

You'll meet SAML as soon as your B2B product has an enterprise customer: *"We need SSO with our Okta."*

**SP-initiated flow** (the user starts at your app):

```
 Browser                          Your app (SP)                         Customer's IdP (Okta)
   │ GET app.com/login?org=acme ────▶ │ build <AuthnRequest> (XML)            │
   │ ◀── 302 redirect to IdP SSO URL ─│  ?SAMLRequest=<deflated+base64>&RelayState=/dashboard
   │ GET IdP SSO URL ────────────────────────────────────────────────────────▶ │ authenticate user (or reuse session)
   │ ◀── HTML page with auto-submitting <form method=POST action="app.com/saml/acs"> ── │
   │       containing SAMLResponse = base64(signed XML assertion)            │
   │ POST app.com/saml/acs ──────────▶ │ 1. verify XML signature with IdP's certificate
   │                                   │ 2. check Issuer, Audience, Destination, InResponseTo
   │                                   │ 3. check NotBefore / NotOnOrAfter (allow small clock skew)
   │                                   │ 4. read NameID + attributes → find/create user → session
   │ ◀── Set-Cookie: session + redirect to RelayState (/dashboard)
```

An assertion (abbreviated) looks like this:

```xml
<saml:Assertion ID="_a1b2c3" IssueInstant="2026-10-08T10:00:00Z">
  <saml:Issuer>https://acme.okta.com</saml:Issuer>
  <ds:Signature>…signed with the IdP's private key…</ds:Signature>
  <saml:Subject>
    <saml:NameID>alice@acme.com</saml:NameID>
    <saml:SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
      <saml:SubjectConfirmationData InResponseTo="_req123" Recipient="https://app.com/saml/acs"
                                    NotOnOrAfter="2026-10-08T10:05:00Z"/>
    </saml:SubjectConfirmation>
  </saml:Subject>
  <saml:Conditions NotBefore="2026-10-08T09:59:00Z" NotOnOrAfter="2026-10-08T10:05:00Z">
    <saml:AudienceRestriction><saml:Audience>https://app.com</saml:Audience></saml:AudienceRestriction>
  </saml:Conditions>
  <saml:AuthnStatement AuthnInstant="2026-10-08T10:00:00Z">…</saml:AuthnStatement>
  <saml:AttributeStatement>
    <saml:Attribute Name="groups"><saml:AttributeValue>engineering</saml:AttributeValue></saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

**Key SAML terms:**
- **ACS URL** (Assertion Consumer Service): your endpoint that receives the POSTed assertion.
- **Entity ID**: a unique identifier for the IdP and for your SP.
- **Metadata**: an XML document each side publishes with its entity ID, URLs and **signing certificate**. Exchanging metadata is how a customer "connects" their IdP to your app.
- **Bindings**: *HTTP-Redirect* (request in the URL) and *HTTP-POST* (response in an auto-submitted form).
- **RelayState**: where to send the user after login.
- **IdP-initiated flow**: the user clicks your app's tile in their Okta dashboard and the IdP POSTs an assertion you never asked for. It's convenient, but there's no `InResponseTo` to match, which makes it more open to replay and login-CSRF.

**Security warning:** SAML's XML signatures are notoriously tricky. **XML Signature Wrapping** attacks move the signed element and inject an unsigned one that a naive parser reads instead. **Never write your own SAML validation.** Use a mature library or, better, a provider that handles enterprise SSO for you (WorkOS, Auth0, Okta, Keycloak, BoxyHQ…).

### 8.6 SAML vs OIDC

| | SAML 2.0 | OpenID Connect |
|---|---|---|
| Data format | XML assertions | JSON, JWT ID tokens |
| Signing | XML-DSig (complex) | JWS (simple) |
| Transport | Browser POST of large XML | Redirects + a back-channel token request |
| Mobile apps / SPAs | Awkward | Designed for them |
| API access | Not designed for it | Comes with OAuth access tokens |
| Where you'll see it | Enterprise IdPs, older SaaS | Everything new, social login |
| **When building B2B SaaS** | Support it; enterprises will ask | Support it too. Many IdPs offer both. |

### 8.7 Social login and "Sign in with Apple"

"Sign in with Google / GitHub / Microsoft" is **federated login over OIDC** (the flow in the first file). Technically it's SSO too: one Google session logs you into many unrelated sites.

**Sign in with Apple** is also OIDC-based, with privacy features:
- **Hide My Email:** the user can share a random relay address (`…@privaterelay.appleid.com`) that forwards to their real email. Your app never learns the real address, and the user can switch forwarding off at any time. (To email users through it, your sending domain must be registered with Apple.)
- On Apple devices the login is approved with **Face ID / Touch ID**, using everything from §6.
- Apple sends the user's **name only on the first login**, so store it then or you won't get it again.
- Apple's App Store rules have historically required apps that offer third-party social login to also offer Sign in with Apple, or (after a 2024 update) an equivalent privacy-focused login option.

### 8.8 Single Logout (SLO) is hard

Logging out of *one* app usually **doesn't** log you out of the IdP or the other apps; each has its own session. SAML and OIDC both define logout protocols (front-channel iframes/redirects, back-channel server-to-server calls), but they're fragile in practice. Most systems rely on **short app sessions** plus the IdP session, and on **deprovisioning** to cut access.

### 8.9 Provisioning: SCIM and JIT

SSO answers *"who is logging in"*, but your app also needs user **accounts** to exist, and to **disappear** when someone leaves.

- **JIT (Just-In-Time) provisioning:** create the user account on their first SSO login, from the assertion's attributes. Simple, but nothing happens when someone is *removed*: their account lingers until they try to log in again.
- **SCIM 2.0** (System for Cross-domain Identity Management): a standard REST API (`POST /scim/v2/Users`, `PATCH`, `DELETE`…) that **the IdP calls on your app** to create, update and **deactivate** users and groups automatically. When HR offboards someone in Okta, Okta immediately calls your SCIM endpoint to disable them.
- **Directories** behind the IdP: **LDAP** / **Active Directory** (§9.2), HR systems.

### 8.10 The "SSO tax"

Many SaaS products put SSO only in their expensive enterprise tier, which critics call the "SSO tax". As a product builder: enterprise customers **will** require SAML/OIDC SSO and SCIM before they buy, so plan for it.

---

## 9. Enterprise and network protocols

### 9.1 Kerberos

The SSO system **inside** corporate networks. When you log into a Windows PC on a company domain and then open file shares, intranet sites and printers without typing your password again, that's Kerberos.

Three parties, and **your password never crosses the network**:
- **KDC** (Key Distribution Center), made up of the **AS** (Authentication Server) and the **TGS** (Ticket Granting Server). On Windows this is the **Active Directory domain controller**.
- **You** (the client).
- **The service** (file server, web app…).

```
 Client (alice)                       KDC                                  File server
   │ 1. AS-REQ: "I'm alice" + timestamp encrypted with a key derived from her password
   │ ──────────────────────────────▶ AS                                       │
   │ ◀── 2. TGT (Ticket-Granting Ticket, sealed with the KDC's own key)       │
   │        + session key (sealed with alice's key, so only she can open it)  │
   │                                                                           │
   │ 3. TGS-REQ: TGT + "I want fileserver" ──▶ TGS                            │
   │ ◀── 4. service ticket (sealed with the FILE SERVER's key) + session key  │
   │                                                                           │
   │ 5. AP-REQ: service ticket + authenticator ──────────────────────────────▶ │ opens ticket with its own key
   │                                                                           │ → "this really is alice"
   │ ◀──────────────────────────────────────────── 6. (optional) mutual auth ─ │
```

- The **TGT** is your "logged in for the day" pass (typically ~10 hours). You get **service tickets** for each service without re-entering your password.
- The service never talks to the KDC during login; it trusts the ticket because only the KDC knows the service's key.
- **Clocks must be in sync** (default tolerance about 5 minutes) because timestamps prevent replay.
- On the web: browsers can do Kerberos via `WWW-Authenticate: Negotiate` (SPNEGO), so intranet sites log you in silently ("Integrated Windows Authentication").
- Known attacks if misconfigured: *pass-the-ticket*, *Kerberoasting* (cracking service account passwords offline from tickets), *golden tickets* (forged TGTs after the KDC's key is stolen).

### 9.2 LDAP and Active Directory

- **LDAP** (Lightweight Directory Access Protocol) is a protocol for querying a **directory**: a tree of users, groups, computers and their attributes (`cn=alice,ou=engineering,dc=company,dc=com`).
- **Active Directory** is Microsoft's directory service. It speaks LDAP for lookups and Kerberos for authentication. OpenLDAP and FreeIPA are open-source alternatives.
- **"LDAP authentication"** usually means an **LDAP bind**: the app sends the user's username and password to the directory server, which says yes or no. Many internal tools (Jenkins, Grafana, VPNs) support it.
  - **Downside:** the app **sees the user's password**. Always use LDAPS (TLS), never plain LDAP on port 389 without StartTLS. Modern setups prefer putting an **IdP with SAML/OIDC** in front of the directory so apps never touch passwords.

### 9.3 NTLM

Microsoft's older challenge–response protocol, kept for compatibility where Kerberos can't be used. It's vulnerable to **pass-the-hash** and **relay attacks**, and Microsoft has **deprecated** it. If you see it, plan to move off it.

### 9.4 RADIUS, TACACS+, and 802.1X

Authentication for **network access** rather than apps:
- **802.1X + EAP + RADIUS:** "**WPA2/WPA3-Enterprise**" Wi-Fi, where you log in to the office Wi-Fi with your *company credentials* or a *device certificate* instead of a shared Wi-Fi password. The access point forwards your credentials to a **RADIUS** server, which checks them against the directory.
- **RADIUS** also backs many **VPNs**.
- **TACACS+:** authenticating network engineers logging into routers and switches, with per-command authorization and auditing.

---

## 10. Machine-to-machine beyond API keys

API keys (first file) send the **secret itself** on every request. These methods avoid that or eliminate long-lived secrets entirely.

### 10.1 HMAC request signing

The client **never sends the secret**. It sends a **signature** computed over the request:

```
signature = HMAC-SHA256(secret, method + path + selected headers + hash(body) + timestamp)
```

The server, which also knows the secret, recomputes the signature and compares.

- **Integrity:** if anything in the request is changed in transit, the signature breaks.
- **Replay protection:** the timestamp is signed, and the server rejects requests outside a small window.
- **No secret on the wire:** a logged request doesn't leak the key.

**AWS Signature Version 4** is the best-known example. Every AWS API call (via SDKs or the CLI) is signed like this:

```
Authorization: AWS4-HMAC-SHA256 Credential=AKIA…/20261008/eu-west-1/s3/aws4_request,
               SignedHeaders=host;x-amz-content-sha256;x-amz-date, Signature=fe5f80f7…
```

### 10.2 Webhook signatures

When Stripe or GitHub **calls your server** with a webhook, how do you know it's really them and not someone POSTing fake "payment succeeded" events? They **sign the payload** with a secret shared with you:

- **Stripe:** `Stripe-Signature: t=1728380000,v1=5257a869…` where `v1 = HMAC-SHA256(secret, "t.rawBody")`
- **GitHub:** `X-Hub-Signature-256: sha256=…` where the value is `HMAC-SHA256(secret, rawBody)`

```js
import crypto from "node:crypto";

// Simplified Stripe-style verification. In production, use the provider's SDK helper.
function verifyWebhook(rawBody, header, secret, toleranceSec = 300) {
  const parts = Object.fromEntries(header.split(",").map((kv) => kv.split("=")));
  const timestamp = Number(parts.t);
  if (!timestamp || Math.abs(Date.now() / 1000 - timestamp) > toleranceSec) return false; // replay window

  const expected = crypto.createHmac("sha256", secret).update(`${timestamp}.${rawBody}`).digest("hex");
  const a = Buffer.from(expected);
  const b = Buffer.from(parts.v1 ?? "");
  return a.length === b.length && crypto.timingSafeEqual(a, b);   // constant-time comparison
}
```

Two classic bugs:
1. **Verifying the parsed-and-re-serialised JSON instead of the raw body.** Whitespace and key order change, so the signature never matches. You need the **raw bytes**.
2. **Comparing with `===`.** Use a constant-time comparison (see the timing-attack section of the first file).

### 10.3 Mutual TLS

See §7.1. It's the standard for service-to-service authentication inside a microservice platform, usually automated by a service mesh.

### 10.4 Workload identity: no long-lived secrets at all

The modern trend: **don't give workloads secrets; let the platform vouch for them.**

| Mechanism | How it works |
|---|---|
| **Cloud IAM roles** | An EC2 instance / Lambda / GCP VM gets **short-lived credentials** from the cloud's metadata service, rotated automatically. No access keys in env vars. |
| **Kubernetes service account tokens** | Each pod gets a short-lived, audience-bound JWT identifying its service account |
| **SPIFFE / SPIRE** | Open standard: each workload gets an identity like `spiffe://company.com/payments/api`, proven with short-lived X.509 certificates or JWTs (SVIDs). Platform-agnostic. |
| **OIDC federation for CI/CD** | GitHub Actions (or GitLab CI) issues an **OIDC token** describing the job ("repo acme/api, branch main"). AWS, GCP or Azure **trust GitHub's issuer** and exchange that token for temporary cloud credentials. **No cloud keys stored in GitHub secrets.** |

```
 GitHub Actions job ──▶ GitHub OIDC provider: "give me a token for this job"
        │ ◀── JWT { iss: token.actions.githubusercontent.com, sub: "repo:acme/api:ref:refs/heads/main", aud: "sts.amazonaws.com" }
        │
        └──▶ AWS STS AssumeRoleWithWebIdentity(JWT)
                 AWS verifies the signature with GitHub's public keys, checks the role's trust policy
             ◀── temporary AWS credentials (valid ~1 hour)
```

This is OIDC (from the first file) used for **machines** instead of people.

### 10.5 Sender-constrained tokens (fixing the "bearer" problem)

The first file noted that bearer tokens work for **whoever holds them**. Two standards **bind a token to a key** the client holds, so a stolen token is useless on its own:

- **DPoP** (Demonstrating Proof-of-Possession): the client signs a small proof JWT for every request with its private key; the access token is bound to that key's public key.
- **mTLS-bound tokens:** the access token is bound to the client's TLS certificate.

Both appear in high-security OAuth deployments such as open banking.

---

## 11. HTTP authentication schemes

All of these use the same two headers:
- the server challenges: `WWW-Authenticate: <scheme> …` with status `401`
- the client responds: `Authorization: <scheme> <credentials>`

| Scheme | Credentials | Notes |
|---|---|---|
| `Basic` | `base64(user:password)` | HTTPS only; internal tools (§2.1) |
| `Digest` | Hash of password + nonce | Obsolete (§2.2) |
| `Bearer` | A token (JWT, opaque, OAuth access token) | The default for APIs |
| `Negotiate` | Kerberos/SPNEGO token | Windows intranet SSO (§9.1) |
| `DPoP` | Access token + `DPoP` proof header | Sender-constrained OAuth (§10.5) |
| `AWS4-HMAC-SHA256` | Signed request (vendor-specific) | AWS APIs (§10.1) |
| *(custom headers)* | e.g. `X-API-Key: …` | Common for API keys; not an official scheme |

Cookies (`Cookie: sid=…`) sit outside this mechanism entirely. They're how browser sessions carry authentication.

---

## 12. Patterns layered on top

### 12.1 MFA and assurance levels

MFA combines factors (know / have / are). NIST defines **Authenticator Assurance Levels**:

| Level | Requirement | Example |
|---|---|---|
| **AAL1** | Single factor | Password only |
| **AAL2** | Two factors | Password + TOTP, or a passkey with biometric/PIN |
| **AAL3** | Two factors, **hardware-backed** and **phishing-resistant** | Hardware security key with PIN, smart card |

Ranking of second factors, weakest to strongest: **SMS < email < TOTP app < push with number matching < passkey / security key**.

### 12.2 Step-up authentication

Ask for **stronger or fresher authentication** before **sensitive actions**, even inside an active session:
- GitHub's **"sudo mode"**: re-enter your password or passkey before deleting a repo or changing security settings.
- Banks: an extra confirmation for transfers to new recipients.
- Implementation: record `auth_time` (when the user last authenticated) and the method used (OIDC's `acr`/`amr` claims) in the session, and require `auth_time` within the last N minutes for sensitive endpoints.

### 12.3 Adaptive or risk-based authentication

Change how much friction you add based on **risk signals**:
- New device or browser? Unusual country? **Impossible travel** (London, then Sydney 20 minutes later)?
- IP reputation (known VPN, Tor, datacenter IP), many failed attempts, a password found in a breach list.

Low risk → let them in silently. Medium → ask for MFA. High → block and notify. Most auth providers offer this as a feature.

### 12.4 "Remember me" and device trust

- **Remember this device** (skip MFA for 30 days): set a long-lived, signed, `HttpOnly` cookie identifying the device, and store its hash server-side so it can be revoked.
- **Persistent login:** a long-lived **refresh token** or "remember me" token that can mint new short sessions. Rotate it on each use; if an old one is reused, assume theft and revoke the whole chain (*refresh token rotation with reuse detection*).

### 12.5 Account recovery: the weakest link

Attackers go for **whichever path is weakest**, and that's often "Forgot password?" or a support agent.
- Reset links follow the magic-link rules (§4): random, hashed, single-use, short-lived.
- **After a reset:** invalidate **all existing sessions** and notify the user by email.
- Don't let recovery bypass MFA: someone with only the email account shouldn't be able to disable MFA.
- Offer **recovery codes** and **multiple passkeys / security keys**.
- Train support staff against social engineering ("I lost my phone, please remove 2FA").

### 12.6 Impersonation ("log in as user")

Support and admin tools often need to see the app as a customer does. Do it safely:
- A **separate, audited** flow, requiring an admin role plus step-up auth.
- The session records **both** identities (`actor: admin_7, subject: user_42`), and every action is logged with both.
- Visible banner, time-limited, and ideally restricted from destructive actions.

### 12.7 Zero trust

The old model was a **castle and moat**: everything inside the corporate network (VPN) is trusted. Zero trust (popularised by Google's **BeyondCorp**) says:
- **Network location grants nothing.** Every request is authenticated and authorized, inside or outside.
- Decisions combine **user identity** (SSO + strong MFA), **device identity and health** (managed? patched? disk encrypted?), and **context**.
- Services authenticate each other too (mTLS, workload identity, §10).
- Implemented with identity-aware proxies instead of VPNs, plus short-lived credentials everywhere.

---

## 13. Emerging methods

| Method | Idea | Status |
|---|---|---|
| **Decentralized identifiers (DIDs)** | Identifiers you control yourself, not issued by Google or Facebook, anchored in a ledger or a web domain | W3C standard; limited adoption |
| **Verifiable Credentials (VCs)** | Digitally signed claims ("over 18", "holds a degree from X") issued to your wallet, which you present without the issuer being involved. Supports **selective disclosure**: prove "over 18" without revealing your birthdate. | W3C standard; growing in government and education |
| **Digital ID wallets / mobile driver's licences** | Government ID on your phone (ISO 18013-5 mDL). Apple Wallet and Google Wallet support IDs in some regions; the **EU Digital Identity Wallet** is rolling out under eIDAS 2.0. | Early deployment |
| **Sign-In with Ethereum (SIWE)** | Your crypto wallet signs a standard login message; the server verifies the signature against your address | Used in web3 apps |
| **Behavioural / continuous authentication** | Constantly re-scores how likely it is that the same person is still there (typing rhythm, movement) | Used quietly in banking fraud systems |
| **Post-quantum cryptography** | New signature and key-exchange algorithms (e.g. NIST's ML-KEM and ML-DSA standards) that quantum computers can't break | Being rolled into TLS and, over time, into signing systems |

---

## 14. Comparison of every method

| Method | Factor(s) | Phishing-resistant? | What the server stores | Typical use |
|---|---|---|---|---|
| Password (form login) | Know | ❌ | Salted Argon2id/bcrypt hash | Default consumer login |
| HTTP Basic | Know | ❌ | Password hash | Internal tools over HTTPS |
| SMS / email OTP | Have (phone / inbox) | ❌ | Hashed short-lived code | Low-assurance 2FA, passwordless |
| TOTP app | Have | ❌ (relayable in real time) | **Encrypted** shared secret | 2FA |
| Push approval | Have | ⚠️ (better with number matching) | Device registration | Workforce MFA |
| Magic link | Have (inbox) | ⚠️ | Hashed single-use token | Passwordless consumer apps |
| **Passkey (WebAuthn)** | Have + are/know | ✅ | **Public key** only | Modern consumer & workforce login |
| Hardware security key | Have (+ PIN) | ✅ | Public key | Admins, high-risk users |
| Face ID / Touch ID | Are (unlocks a "have") | n/a (local only) | **Nothing**; the device keeps it | Unlocking passkeys and app keys |
| Smart card / PIV | Have + know | ✅ | Trusted CA / certificate | Government, enterprise |
| mTLS client cert | Have | ✅ | Trusted CA | Service-to-service, IoT, banking APIs |
| SSH keys | Have | ✅ | Public key | Server access, Git |
| Kerberos | Know (then tickets) | ✅ (within the domain) | Keys in the KDC | Corporate Windows networks |
| LDAP bind | Know | ❌ | Directory password hash | Internal tools |
| SAML / OIDC SSO | Whatever the IdP uses | Depends on the IdP's method | IdP certificate / client config | Enterprise SSO, social login |
| API key | Have (secret) | n/a | Hashed key | Developer APIs |
| HMAC signing | Have (secret) | n/a | **Retrievable** secret | AWS APIs, webhooks |
| Workload identity | Platform-attested | n/a | Trust policy | Cloud workloads, CI/CD |

---

## 15. What to use when

| You're building… | Recommended approach |
|---|---|
| **A consumer web app** | **Passkeys** as the primary option + email/password fallback with **TOTP** MFA (or magic links), plus **social login** via OIDC. Sessions in `HttpOnly` cookies. Strongly consider an auth provider. |
| **A consumer mobile app** | OIDC / social login or passkeys; short access token + refresh token stored in the **Keychain/Keystore**, unlocked with **Face ID / fingerprint** for re-entry |
| **A B2B SaaS product** | Everything above **plus SAML and OIDC SSO** for enterprise customers, **SCIM** provisioning, per-organisation MFA policies, audit logs |
| **Internal company tools** | Put them behind the company **IdP via SSO** (OIDC/SAML) or an identity-aware proxy. Don't create separate passwords. |
| **A public developer API** | **API keys** (hashed, scoped, prefixed) for simple access; **OAuth 2.0** when third-party apps act on behalf of your users |
| **Webhooks you send** | **HMAC-sign** every payload with a timestamp; publish a verification guide |
| **Microservices talking to each other** | **mTLS** via a service mesh, or short-lived JWTs from workload identity |
| **CI/CD deploying to the cloud** | **OIDC federation**, not stored cloud keys |
| **A CLI tool** | **OAuth device code flow** (like `gh auth login`) or a browser redirect to `localhost` |
| **IoT devices** | Per-device **X.509 certificates** + mTLS |
| **High-security / admin access** | **Hardware security keys** or device-bound passkeys, step-up auth, short sessions |

---

## 16. Glossary

| Term | Meaning |
|---|---|
| **Authenticator** | Whatever holds or produces the credential: phone, security key, app, Secure Enclave |
| **Platform / roaming authenticator** | Built into the device vs a separate device (YubiKey, phone) |
| **Passkey** | A FIDO2/WebAuthn credential (per-site key pair), usually synced |
| **WebAuthn / FIDO2 / CTAP2** | Browser API / full standard / authenticator protocol |
| **Relying Party (RP)** | The website or app that relies on an authentication (WebAuthn, OIDC) |
| **Phishing-resistant** | Can't be replayed on a fake site, because it's bound to the real origin |
| **HOTP / TOTP** | Counter-based / time-based one-time password (HMAC of a shared secret) |
| **Magic link** | Single-use login link sent by email |
| **Secure Enclave / TPM** | Isolated hardware that stores keys and biometric templates |
| **Liveness detection** | Distinguishing a real person from a photo, mask or deepfake |
| **FAR / FRR** | False acceptance / false rejection rate of a biometric system |
| **mTLS** | Both client and server present certificates during the TLS handshake |
| **CA** | Certificate Authority, which signs certificates that others trust |
| **SSO** | Log in once at an IdP, access many apps |
| **IdP / SP** | Identity Provider (authenticates) / Service Provider (trusts the IdP) |
| **Federation** | A trust relationship between identity systems, often across organisations |
| **SAML assertion** | Signed XML statement from the IdP: "this is alice, these are her attributes" |
| **ACS URL** | The SP endpoint that receives SAML responses |
| **SCIM** | Standard API for the IdP to create, update and deactivate users in your app |
| **JIT provisioning** | Creating the user account on first SSO login |
| **Kerberos / KDC / TGT** | Ticket-based network SSO / the ticket server / your "day pass" ticket |
| **LDAP / Active Directory** | Directory query protocol / Microsoft's directory service |
| **RADIUS / 802.1X** | Network-access authentication (enterprise Wi-Fi, VPN) |
| **HMAC signing** | Proving you hold a secret by signing the request with it, without sending it |
| **Workload identity** | Platform-issued, short-lived credentials for machines (IAM roles, SPIFFE, OIDC federation) |
| **DPoP** | Binding an OAuth token to a client-held key so stolen tokens are useless |
| **Step-up auth** | Re-authenticating before sensitive actions |
| **Adaptive auth** | Adjusting authentication requirements based on risk signals |
| **AAL1/2/3** | NIST authenticator assurance levels |
| **Zero trust** | No implicit trust from network location; verify every request |
| **Verifiable Credential** | A signed, user-held claim that can be presented without contacting the issuer |

---

## Closing thought

Almost every method here is one of three basic ideas:

1. **A shared secret**: passwords, OTP seeds, API keys, HMAC keys. Both sides know it, so it can leak from either side.
2. **A key pair**: passkeys, security keys, SSH, mTLS, smart cards, Face ID-protected keys. Only you hold the private part; the server stores something useless to a thief.
3. **A trusted third party vouching for you**: SSO, SAML, OIDC, Kerberos, workload identity. Someone the server already trusts says "this is alice".

The industry is moving from (1) toward (2) and (3): from secrets that can be phished and leaked to **keys that never leave hardware**, and from **many passwords** to **one strong identity, federated everywhere**. Biometrics like Face ID don't replace keys; they're the convenient way to unlock them.
