# Serialization and Deserialization for Backend Engineers

Detailed notes from the video transcript (chapter 7). The topic sounds heavy but has very few moving parts: it's about **how two programs written in different languages, on different machines, agree on what the data they exchange means**.

---

## Table of contents

1. [The problem: different languages, same data](#1-the-problem)
2. [Where this happens: the OSI model (high level only)](#2-the-osi-model)
3. [The solution: a common standard](#3-the-solution-a-common-standard)
4. [Definition](#4-definition)
5. [Why this series focuses on HTTP, Postgres, and JSON](#5-why-http-postgres-and-json)
6. [Types of serialization standards](#6-types-of-serialization-standards)
7. [JSON in detail](#7-json-in-detail)
8. [Demo: JSON in a real request/response](#8-demo-json-in-a-real-requestresponse)
9. [The mental model to keep](#9-the-mental-model-to-keep)
10. [Summary](#10-summary)

---

## 1. The problem

The typical setup:

- **Client** — a browser (e.g. Chrome) running the **frontend**. Usually a JavaScript app: React, Angular, Vue, or any other JS framework/library. Most client-side apps are built with JavaScript because they are highly interactive.
- **Server** — running on `localhost` or remotely in the cloud (AWS, GCP, Azure, anywhere).
- **Network communication** between them — HTTP (traditional REST API endpoints), gRPC, WebSockets, etc. This series focuses on **HTTP / REST**.

To send data, the client builds an HTTP request (covered in earlier chapters):

```
POST /something HTTP/1.1          ← method, URL, version
Host: somedomain.com              ← headers
Content-Type: application/json

{ "name": "some string" }         ← request body
```

The server replies with a response the client must be able to **parse and make sense of**.

### The core question

Say the client is **JavaScript** and the server is **Rust**:

| | JavaScript | Rust |
|---|---|---|
| Typing | Fully dynamic | Very strict |
| Execution | Not compiled (interpreted/JIT) | Compiled |
| Data types | JS objects, numbers, strings… | Structs, `i32`/`u64`, `String`, enums… |

Their data types are completely different. So:

- The client sends a JS object `{ name: "some string" }` in the request body.
- How does the Rust server take that data and understand it **in its own types** (e.g. a struct)?
- How does it run business logic and send a response that the JS client can **again** understand and use to render UI or run its own logic?

How does data that leaves one machine, crosses the internet, and lands on a completely different machine running a completely different language still *mean the same thing*?

---

## 2. The OSI model

Before talking about data transmission over the internet, it helps to have a **high-level** idea of the OSI model. You don't need to master it (IP packets, data frames, etc.) — there are plenty of good resources online for that — just know it exists.

- Data passes through several **layers** on its way across the network.
- The **application layer** is at one end (top); the **physical layer** is at the other (bottom). Whether you call it top or bottom doesn't matter — they're at opposite ends.
- The **same stack exists on the server**: application layer at one end, physical layer at the other, other layers in between.

```
   Client                          Server
┌──────────────┐               ┌──────────────┐
│ Application  │  ← you work   │ Application  │
├──────────────┤     here      ├──────────────┤
│     ...      │               │     ...      │
├──────────────┤               ├──────────────┤
│   Physical   │ ──── bits ──→ │   Physical   │
└──────────────┘               └──────────────┘
```

---

## 3. The solution: a common standard

Imagine you were asked to design this yourself:

> Two machines in different locations, connected over the internet. Come up with a technique (a *protocol*) so one machine can send data to the other, and the receiver can parse it and make sense of it — **in a language-agnostic way**.

The obvious solution: **agree on a common standard** — a fancy way of saying *a set of rules* that both client and server agree to follow for the format of data they send and receive.

The flow then becomes:

1. **Client (JavaScript)** — does whatever it needs during the user's session. When sending data, it **converts** its JS data into the common format → sends it over the network.
2. **Server (Rust)** — receives data in the common format and **converts** it into its own types (e.g. a Rust **struct**). Now it understands the data and runs its logic.
3. **Server responds** — converts the Rust struct **back into** the common format → sends it over the internet.
4. **Client** — reads it and **parses** it into JavaScript data, then does whatever it needs (render UI, etc.).

```
JS object ──serialize──▶ [ common format ] ──deserialize──▶ Rust struct
                                                                 │
                                                          business logic
                                                                 │
JS object ◀─deserialize── [ common format ] ◀──serialize──── Rust struct
```

That common format is a **serialization standard**.

---

## 4. Definition

> **Serialization** = converting data from a language's in-memory representation **into** a common/standard format.
>
> **Deserialization** = converting data **from** that common format back into a language's own representation.

The purpose: during **transmission** (and also **storage** — touched on later in the series), data stays **domain-agnostic / language-agnostic**, so machines, servers, and clients from different languages and environments can make sense of it and respond to it.

---

## 5. Why HTTP, Postgres, and JSON

Backend is a huge domain with hundreds of technologies. Understanding the concepts behind all of them is impossible, so the series picks the **most used** option in each area. Once you have real experience with one, the others are easy to pick up during your career.

| Area | Options in the industry | Chosen for this series | Why |
|---|---|---|---|
| Data transmission | HTTP, WebSockets, gRPC, … | **HTTP** | REST APIs are still the most common client–server communication on the internet |
| Databases | Relational: Postgres, MySQL, SQLite · Non-relational: MongoDB, DynamoDB (AWS) | **PostgreSQL** | Very popular, widely used; often the first choice for startups and enterprise |
| Serialization | JSON, YAML, XML, Protobuf, Avro, … | **JSON** | Used ~80% of the time (rough estimate) for client–server communication |

---

## 6. Types of serialization standards

Serialization standards fall into two broad families:

### Text-based

Human-readable formats.

- **JSON** — the dominant format for HTTP client–server communication.
- **YAML** — used for serialization, though not typically for HTTP communication (common in config files).
- **XML** — same; older, still used in some systems.

### Binary

Data is encoded as compact bytes rather than readable text.

- **Protocol Buffers (Protobuf)** — the most popular binary format (used heavily with gRPC).
- **Avro** and others.

```
Serialization standards
├── Text-based
│   ├── JSON   ← focus of this series
│   ├── YAML
│   └── XML
└── Binary
    ├── Protobuf
    └── Avro, ...
```

For now, the focus is only **JSON**, since it's the most popular choice for HTTP-based communication.

---

## 7. JSON in detail

**JSON = JavaScript Object Notation.** It looks and behaves very much like a JavaScript object — hence the name — but it is **not limited to JavaScript**. It's used everywhere:

- **HTTP / REST API** data transmission between clients and servers (the focus here)
- **Configuration files**
- **Log files** — a very popular choice for logging application/server data at runtime

### Key strength: human-readable

JSON was designed to be readable by humans, and its data types are fundamental ones you'll recognise from any programming language.

### Example

```json
{
  "id": 1,
  "name": "Faheem",
  "isActive": true,
  "skills": ["javascript", "rust"],
  "address": {
    "country": "India",
    "phoneNumber": 3456
  }
}
```

### The rules

1. **Starts with `{` and ends with `}`** (opening and closing curly braces).
2. **Keys must be strings inside double quotes.** `"name"` ✅ — `name` ❌ — `'name'` ❌ — `1` ❌. No other data type can be a key.
3. **Values** can be:
   - **string** — `"India"`
   - **number** — `3456`
   - **boolean** — `true` / `false`
   - **array** — `["a", "b"]`
   - **nested object** — another `{ ... }`
   - *(also `null` — part of the JSON spec, though not mentioned in the video)*
4. **Nested objects follow exactly the same rules** as the outer object: braces, double-quoted string keys, and values of the types above. Nesting can go as deep as needed.

That's essentially all you need to know about JSON's structure.

---

## 8. Demo: JSON in a real request/response

Using the same demo app from the previous video, inspected through a proxy tool's (Burp Suite) HTTP history, this time focusing on the **data format** rather than the HTTP mechanics.

### Request — `POST /api/books`

The body follows the JSON rules: curly braces, double-quoted string keys, one number value and two string values.

```json
{
  "id": 3,
  "title": "Some Book",
  "author": "Some Author"
}
```

This is what gets transmitted over the network.

### Response

The server parses the request JSON, takes the `id`, `title`, `author`, adds the book to its list, and responds with **another JSON**:

```json
{
  "books": [
    { "id": 1, "title": "...", "author": "..." },
    { "id": 2, "title": "...", "author": "..." },
    { "id": 3, "title": "Some Book", "author": "Some Author" }
  ]
}
```

- Opening/closing braces, one key whose value is an **array**.
- Each array element is an **object** — also valid JSON (double-quoted keys; number and string values).

The client receives this, **parses** it, makes sense of it, and **renders** it in its own way. Here the client is JavaScript, but it could just as well be PHP or anything else — that's the point of a language-agnostic format.

---

## 9. The mental model to keep

If you go deep into the OSI model, you'll see that data from the application layer is converted into data frames, IP packets, passes through the transport layer, and finally becomes **bits** (0s and 1s) sent as voltage signals over optical fibre, etc.

**As a backend engineer, none of the intermediate steps are your concern.** The mental model is:

```
Client                                                  Server
  │                                                        │
  │  JSON  ──▶ [ frames → packets → bits 0101... ] ──▶  JSON │
  │                                                        │
  │  JSON  ◀── [ bits 0101... → packets → frames ] ◀──  JSON │
```

- At the **application layer**, the client converts its data into **JSON**. That's where your responsibility ends on the sending side.
- Whatever formats the data passes through afterwards doesn't matter.
- On the receiving side, after all the network work, the data is back to **JSON** before the server sees it. **The server only ever reads JSON.**
- Same thing in reverse for the response.

So: **client and server both adhere to the agreed standard (JSON); the network in between is a black box.**

---

## 10. Summary

- Serialization/deserialization isn't a big topic with lots of concepts — it's a **phenomenon** in the backend world that you need to know exists.
- **Serialization and deserialization are the techniques for converting data into and out of a common, standard format**, so that:
  1. the client sends data to the server in that format,
  2. the server receives it, parses it, makes sense of it, runs business logic,
  3. the server sends the response back in that same format,
  4. the client parses it and does the same.
- The goal: data that is **understandable across domains and languages**.
- Standards are **text-based** (JSON, YAML, XML) or **binary** (Protobuf, Avro). For HTTP client–server communication, **JSON** is the default.
- JSON rules: `{ }` braces, **double-quoted string keys**, values of type string / number / boolean / array / nested object (/ null).

### Quick reference

| Term | Meaning |
|---|---|
| Serialize | Language data → common format (e.g. JS object → JSON text) |
| Deserialize | Common format → language data (e.g. JSON text → Rust struct) |
| Serialization standard | The agreed set of rules for the common format |
| Text-based format | Human-readable: JSON, YAML, XML |
| Binary format | Compact bytes: Protobuf, Avro |
| JSON | JavaScript Object Notation — most common for REST APIs, also configs and logs |
