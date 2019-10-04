# Cash:web Keyserver - Pay-for-Put Protocol Documentation

## Abstract

We introduce a "Pay-for-Put" protocol, allowing users to pay keyserver operators for key updates.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt). 

## Motivation

Traditional keyservers are subject to certificate spamming attacks.

## Specification

The protocol sits ontop of HTTP and makes use of REST API semantics. It closely follows the [BIP70 protocol](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) and, as such, prior knowledge of it is recommended.

It is to be noted that [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) must be suplemented with [BIP71](https://github.com/bitcoin/bips/blob/master/bip-0071.mediawiki) and [BIP72](https://github.com/bitcoin/bips/blob/master/bip-0072.mediawiki), and that both take on a [slightly altered](https://lists.linuxfoundation.org/pipermail/bitcoin-ml/2017-August/000177.html) form for Bitcoin Cash. We refer to this this collection of BIPs as [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) from hereon after.

The protocol's messages are encoded via [Google's Protocol Buffers](https://developers.google.com/protocol-buffers), authenticated using [X.509 certificates](https://tools.ietf.org/html/rfc5280), and communicated over HTTP/HTTPS.

The protcol consists of three round-trips:
1. An initial PUT with empty body, responded to by a [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) payment request.
2. A POST containing payment, responded to with a [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) payment acknowledgement and a token.
3. A PUT containing signed metadata and the token, responded to by a upload acknowledgement.

### Protocol Buffer Messages

In addition to those provided in [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki), we have the following **version 3** messages.

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

### 1 - Initial PUT

To initiate the protocol the client sends a PUT request to the keyserver on the path `/keys/{address}`. 

* Headers and body MUST NOT be included.
* Addresses MAY be given in P2PKH [cashaddr](https://www.bitcoincash.org/spec/cashaddr.html) or [base58](https://en.bitcoin.it/wiki/Technical_background_of_version_1_Bitcoin_addresses) format.
* Addresses MUST include checksums and prefixes.
* If a keyserver is accepting payments from the main, test or regtest network they MUST only deem the corresponding addresses valid. This is to avoid confusion and ensure that main and testing networks stay segregated.
* The address MUST be a valid Bitcoin Cash address.

Any failure to meet the above criteria SHOULD be responded to with status code `400` and an appropriate error message.

A successful request MUST be responded to with a status code `402` and the `PaymentRequest` message defined in [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki).

```protobuf
message PaymentDetails {
        optional string network = 1 [default = "main"]; // "main" or "test"
        repeated Output outputs = 2;        // Where payment should be sent
        required uint64 time = 3;           // Timestamp; when payment request created
        optional uint64 expires = 4;        // Timestamp; when this request should be considered invalid
        optional string memo = 5;           // Human-readable description of request for the customer
        optional string payment_url = 6;    // URL to send Payment and get PaymentACK
        optional bytes merchant_data = 7;   // Arbitrary data to include in the Payment message
}

message PaymentRequest {
        optional uint32 payment_details_version = 1 [default = 1];
        optional string pki_type = 2 [default = "none"];  // none / x509+sha256 / x509+sha1
        optional bytes pki_data = 3;                      // depends on pki_type
        required bytes serialized_payment_details = 4;    // PaymentDetails
        optional bytes signature = 5;                     // pki-dependent signature
}
```

Details can be found in the "PaymentDetails/PaymentRequest" section of [BIP70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki).

In addition to the [BIP70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) requirements the `merchant_data` field of the `PaymentDetails` message MUST contain the URL? Or can it just contain the address??

**WIP**

The keyserver MUST incude an `OP_RETURN` output in the `outputs` field given by...Rationale for this can be found in later sections.

### 2 - Payment and Token Issuance

The client sends the payment and the keyserver responds with a payment acknowledgement message, both in accordance to [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki). 

Note that the `merchant_data` in the clients `Payment` message SHOULD be equal to the `merchant_data` in the keyservers `PaymentDetails`. This ensures that the payment is tied to the target address.

In addition to the [BIP 70](https://github.com/bitcoin/bips/blob/master/bip-0070.mediawiki) the keyserver generates a token string using the following process:

1. Extract the `merchant_data` from the `Payment` message.
2. Perform HMAC SHA256 with a private secret to yield the raw token.
3. Encode the raw token using URL safe base64.

This response MUST have status code `202` and include both:
* The `LOCATION` header `merchant_data?code={token string}` (?? if merchant data is just address then `/keys/merchant_data...` instead??),
* The `AUTHORIZATION` header `POP {token string}`.

### 3 - Metadata Submission

The client constructs the `Payload` protobuf message. The `Entry` messages SHOULD contain the items to be uploaded. 

* The `format` string SHOULD define the data format of the `body` bytes.
* It is RECOMMENDED that the `header` fields convey information useful for decoding.
* The `timestamp` field is given in UNIX time (seconds) and, if metadata already exists at the address, MUST be strictly greater than the last `timestamp`.
* The `ttl` is given in seconds and counts how long after the `timestamp` the server SHOULD expunge the data. It MUST be strictly greater than 0.

The client constructs the `Metadata` protobuf message according to the following requirements:

* The `pub_key` field MUST be a compressed SECP256k1 key and the address the encoded `HASH160` image of it (with appropriate prefix's and checksums).
* The `scheme` field MAY be either `SCHNORR` or `ECDSA` and defines the signature scheme.
* The `signature` field MUST contain the a compressed signature covering the serialized `Payload`, signed using the private key paired with `pub_key`.

The client sends the `Metadata` in a PUT request to keyserver on the path `/keys/{address}`. The request MUST include either the query string `code={token string}` or an `AUTHORIZATION` header containing `POP {token string}`.

If any of the requirements are not met the keyserver SHOULD respond with status code `400` and an appropriate error message. Otherwise, the keyserver MUST respond with status code `200`, an empty body and empty headers.

## Rationale

### BIP70

### Required OP_RETURN Data

### Statelessness

### Independence from P2P network protocol

## Reference Implementations

### Servers

* Golang - https://github.com/cashweb/keyserver
* Rust - https://github.com/hlb8122/keyserver-rust

### Clients
