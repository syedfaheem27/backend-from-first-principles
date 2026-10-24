# Authentication and Authorization

## Digital Cerificates
- A file or an electronic document that proves the authenticity of a server, user or a device
  using `cryptography` and the `Public Key Infrastructure(PKI)`.
- Another common use case of digital certificates is to confirm the authenticity of a website
  to a web browser, which is also known as `secure sockets layer` or `SSL certificate`.
- A server’s certificate contains its domain name, its public key, validity dates,
  and a digital signature made with the private key of whoever issued it
  (usually an intermediate CA, not a root directly).

**How does a Certificate authority know that the public key presented by the server actually represents the server?**

- Before a Certificate Authority gives out a digitally signed certificate to a server, it needs
  to establish two things:
  - `The domain ownership or control:` done by giving a challenge - telling the server to change
    a DNS TXT record and then verifying it.
  - `Public key ownership:` The CA needs to know whether the public key the server presented has an actual
    corresponding private key with the server.
    This is done via a `Certificate Signing Request (CSR)` which is created by the server.
    `Cerificate Signing Request (CSR):` A request by the server which includes domain identity along
    with the public key and this is digitally signed by the corresponding private key. The CA upon
    recieving the request verifies the signature via the submitted public key.

**How does a web browser trust a digital certificate presented by a server?**
- A browser ships with 100's of root store - a few hundred Certificate Authority(CA) root
  certificates, each containing a public key. The certificate to trust signs a digital signature
  with its own private key and the browser verifies it with the pre-installed public keys
  from the stores own digital certificates.

### Certificate revocation
- The browser needs to check whether the certifcate presented by the server has been revoked or not.
- It checks in the Certificate Revocation List(CRL) but the problem with it is that it may not be
  updated as the browser may have cached it after the last time it was downloaded.
- To go around this, there is `OCSP(Online Certifcate Status protocol)`.
- `OCSP(Online Certificate Status Protocol):` OCSP lets a client ask an OCSP responder about
  a particular certificate's status.
- Downside to it is another network request which can add latency. To cater to this, there is one
  TLS feature called `OCSP Stapling`. When the TLS handshake starts, the server has already asked
  the OCSP responder about the certificate status and the server obtains a signed OCSP response and attaches it to its TLS handshake, allowing the client to validate that response without independently contacting the OCSP responder for that connection. This reduces latency.