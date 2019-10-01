# Cash:web Keyserver - Pay-for-Put Protocol Documentation

## Abstract

Traditional keyservers are subject to certificate spamming attacks. To prevent this we introduce a "pay-for-put" protocol, allowing users to pay the keyserver operator for key updates.

## Specification

The protocol sits ontop of HTTP and makes use of REST API semantics. It closely follows the [BIP70 protocol](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) and, as such, prior knowledge of it is recommended.

The protocol messages are encoded via [Google's Protocol Buffers](https://developers.google.com/protocol-buffers), authenticated using [X.509 certificates](https://tools.ietf.org/html/rfc5280), and communicated over HTTP/HTTPS.

The protcol consists of three round-trips:
* An initial PUT with empty body, responded to by a BIP70 payment request,
* a POST containing payment, responded to with a BIP70 payment acknowledgement and a token,
* a PUT containing signed metadata and the token, responded to by a upload acknowledgement

### Protocol Buffer Messages

In addition to those provided in BIP70, we have the following **version 3** messages.

```protobuf
// Basic key/value used to store header data.
message Header {
    string name = 1;
    string value = 2;
}

// MetadataField is an indidual piece of structured data provided by wallet authors.
message Entry {
    // Format is a hint to wallets as to what type of data to deserialize from the metadata field.
    string format = 1;
    // The headers is excess metadata that may be useful to a wallet.
    repeated Header headers = 2;
    // Body of the metadata field.
    bytes body = 3;
}

// Payload is the user-specified data section of a AddressMetadata that is covered by the users signature.
message Payload {
    // Timestamp allows servers to determine which version of the data is the most recent.
    int64 timestamp = 1;
    // TTL tells us how long this entry should exist before being considered invalid.
    int64 ttl = 2;
    // User specified data.  Presumably some conventional data determined by wallet authors.
    repeated Entry entries = 3;
}

// Metadata is the basic unit of the keyserver.  It is used in both PUT and GET requests.
message Metadata  {
    // Serialized version of the XPubKey.  The *hash* of this XPub should correspond to the `key` in the kv store.
    bytes pub_key = 1;
    // Signature is the signature of the metadata by XPubKey.
    bytes signature = 2;
    // Signature scheme provided.  Default is Schnorr, but can be ecdsa.
    enum SignatureScheme {
        SCHNORR = 0;
        ECDSA = 1;
    }
    SignatureScheme scheme = 3;
    // Payload is the metadata set by the user, and covered by the signature.
    Payload payload = 4;
}
```

## Rationale

### Statelessness

### BIP70

## Reference Implementations

* Golang - https://github.com/cashweb/keyserver
* Rust - https://github.com/hlb8122/keyserver-rust
