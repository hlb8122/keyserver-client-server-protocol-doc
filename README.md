# Cash:web Keyserver - Pay-for-Put Protocol Documentation

## Abstract

Traditional keyservers are subject to certificate spamming attacks. To prevent this we introduce a "pay-for-put" protocol, allowing users to pay the keyserver operator for key updates.

## Specification

The protocol sits ontop of HTTP and makes use of REST API semantics. It follows closely the the [BIP70 protocol](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) and, as such, prior knowledge of it is recommended.

The protocol messages are encoded via [Google's Protocol Buffers](https://developers.google.com/protocol-buffers), authenticated using [X.509 certificates](https://tools.ietf.org/html/rfc5280), and communicated over HTTP/HTTPS.

The protcol consists of three round-trips:
* An initial PUT with empty body, responded to by a BIP70 payment request,
* a POST containing payment, responded to with a BIP70 payment acknowledgement and a token,
* a PUT containing signed metadata and the token, responded to by a upload acknowledgement

PROTOCOL DIAGRAM HERE



## Rationale

### Statelessness

### BIP70

## Reference Implementations

* Golang - https://github.com/cashweb/keyserver
* Rust - https://github.com/hlb8122/keyserver-rust
