---
title: "AusDigtial Transaction Access Point (TAP) 2.0 Specification"
specID: "ausdigital-tap/2"
status: "![raw](http://rfc.unprotocols.org/spec:2/COSS/raw.svg)"
editors: "[Chris Gough](mailto:christopher.d.gough@gmail.com)"
contributors: "[Steven Capell](mailto:steven.capell@gosource.com.au)"
---

## Introduction

This document describes a protocol for exchanging formal documents (such as
invoices) between businesses. TAP is a secure, decentralised, peer to peer
architecture where gateways are optional and minimally trusted.


## Goals

The primary goal of the Transaction Access Point (TAP) 2.0 Specification is
to provide a standard way for business systems to send and receive secure
messages with the following characteristics:

 * Support multiple gateway topologies (traditional EDI "4 corner" model,
   point-to-point and broker-based models, etc)
 * Enforced carrier neutrality; sender encrypts, recipient decrypts,
   intermediaries only have access to payload cyphertext (protect against weak
   permieter-security on complex gateway topologies)
 * Selectively transparent; enable sub-protocols for information sharing with
   appropriate 3rd parties.
 * Sufficient envelope data to allows 3rd parties to verify message cleartext
   and be aware of appropriately linked data.
 * Support evidence of provonance and auditability without compromosing
   security


The Transaction Access Point (TAP) 2.0 Specification defines:

 * API for interacting with business message endpoints
 * Generic envelope structure that supports any type of business message
 * Allowable cryptographic techniques


## Status

This spec is an early draft for consuiltation.

This specification aims to support the Australian
[eInvoicing initiative](http://ausdigital.org), and is under active
development at
[https://github.com/ausdigital/ausdigital-tap](https://github.com/ausdigital/ausdigital-tap).

Comments and feedback are encouraged and welcome. Pull requests with
improvements are welcome too.


## Glossary:

phrase | Definition
------------ | -------------
ausdigital-tap/2 | Version 2 of the [AusDigtial](http://ausdigital.org) [TAP](http://ausdigital.org/specs/ausdigital-tap/2.0/) specification
ausdigital-tapgw/1 | Version 1 of the [AusDigtial](http://ausdigital.org) [TAP-GW](http://ausdigital.org/specs/ausdigital-tap-gw/1.0/) specification
ausdigital-bill/1 | Version 1 of the [AusDigtial](http://ausdigital.org) [BILL](http://ausdigital.org/specs/ausdigital-bill/1.0/) specification
ausdigital-dcl/1 | Version 1 of the [AusDigtial](http://ausdigital.org) [DCL](http://ausdigital.org/specs/ausdigital-dcl/1.0) specification
ausdigital-dcp/1 | Version 1 of the [AusDigtial](http://ausdigital.org) [DCP](http://ausdigital.org/specs/ausdigital-dcp/1.0) specification
ausdigital-nry/1 | Version 1 of the [AusDigtial](http://ausdigital.org) [NRY](http://ausdigital.org/specs/ausdigital-nry/1.0/) specification
ausdigital-idp/1 | Version 1 of the [AusDigtial](http://ausdigital.org) [IDP](http://ausdigital.org/specs/ausdigital-idp/1.0/) specification


## Licence

Copyright (c) 2016, 2017 the Editor and Contributors. All rights reserved.

This Specification is free software; you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the Free
Software Foundation; either version 3 of the License, or (at your option) any
later version.

This Specification is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more
details.

You should have received a copy of the GNU General Public License along with
this program; if not, see
[http://www.gnu.org/licenses](http://www.gnu.org/licenses).


## Change Process

This document is governed by the
[2/COSS](http://rfc.unprotocols.org/spec:2/COSS/) (COSS).


## Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in RFC 2119.


# Introduction

The Transaction Access Point (TAP) is a persistently connected "peer" capable
of sending and receiving business documents, such as invoices. It interacts
with other TAPs following the protocol specified in this document. The TAP is
an autonomous agent in business-to-business document exchange.

A TAP might be provided by a commercial ledger service, or maintained as part
of an independent business system.

The TAP specification has two parts. The main part (ausdigital-tap/2) defines
the protocol all peers must follow (and interfaces they must provide) to send
and receive business documents. The second part is an optional gateway
specification (ausdigital-tapgw/1), which defines a client-server protocol
for trusted business system components (e.g. ledger services) to interact with
independent TAP service providers in a generic way.


## Dependencies

The messages sent between TAPs carry semantic payloads. Currently, these
include ausdigital-bill/1. That specification is maintained independently in
the
[https://github.com/ausdigital/ausdigital-bill](https://github.com/ausdigital/ausdigital-bill)
repository. Future semantic payloads may be supported without change to the
protocol.

All TAPs depend on the following Services:

 * ausdigital-dcl/1
 * ausdigital-dcp/1
 * ausdigital-nry/1

TAPGW providers also depend on the ausdigital-idp/1. TAPs do not need to
authenticate when they interact with each other, due to use of well known
cryprograpic keys and service endpoint addresses.


## TAP Protocol Overview

In this protocol, a Transaction Access Point (TAP) is a business system component that
sends and receives business messages. The TAP Protocol describes how one TAP sends the
message, and how the other TAP responds when a message is received.

The message is sent to the receiving TAP using HTTP POST operation, with a
`Content-Type: multipart/form-data` body. This body contains two parts, `message` and
`signature`.

The `message` part is a mixture of cleartext metadata (used by TAPs) and enciphered
payload (used by trusted business system components). The cleartext metadata does not
contain sensitive business information, whereas access to the business-sensitive
information within the payload is not necessary for participating in the TAP protocol.

The `signature` part is created by a business system component trusted by the sender
(with access to the sender's private key material). The signature can be used as a
unique identifier of the message contents (e.g. transmitted document id).

![Illustration of POST body](./tap_overview_post.png "Business document sent to TAP")

Receiving TAPs may also use the signature as a filter (messages with invalid
signatures MAY be dropped by receiving TAPs, rather than delivered). This allows TAPs
to buffer trusted components from anonymous denial of service attacks.

When a valid message is received, the TAP issues an HTTP 200 status and returns a
response body with `Content-Type: application/json`, containing received message
information and optional HATEOS-style list of callback URLs.

```
{
  "data": {
    "type": "TapMessage",
    "id": "tap-generated UUID of the message",
    "attributes": {
        "documentHash": "proxied field 'hash' from message.json document",
        "deliveryStatus": "processing"
    }
  }
}
```

See the TAP Protocol Details chapter for more information.


# TAP Protocol Details


The TAP Protocol is a very simple REST API. One business sends a message directly
to another business' TAP endpoint (discovered via the `ausdigital-dcp/1` protocol):

 * The sender uses the HTTP POST verb (over HTTPS) to send the signed message to a TAP,
 * The sender MUST use HTTPS,
 * The TAP MUST reply with received message info or error response,
 * The TAP MAY notarize messages that it receives (per `ausdigital-nry/1`).


## Receiving a business message

When a message is sent to a TAP endpoint, it MUST validate the message before
delivering it to the recipient:

 * If the message does not validate against the JSON schema, return HTTP error (400).
 * If the sender participant identifier is not valid, an HTTP 200 may be returned but the message MUST NOT be delivered.
 * If the signing key for the message is not in the sender participant identifier's published list of signing keys (and non-revoked at the time of signing, per `ausdigital/DCP` protocol), an HTTP 200 may be returned but the message MUST NOT be delivered.
 * If the message signature is not made with the signing key of the message, and HTTP 200 may be returned but the message MUST NOT be delivered.


## Sending a business message

Pre-requisites:

 * Sender knows their own business identity, and has access to their own private key.
 * Sender knows the recipient business identity (e.g. ABN).
 * Sender knows recipient Digital Capability Publisher (DCP) address, which can be discovered from business identity using ausdigital-dcl/1.
 * Sender knows appropriate TAP endpoint address for the intended message type (discovered via DCP lookup).
 * Sender knows appropriate recipient public key (discovered via DCP lookup).
 * The sender has a valid cleartext (not encrypted) business document to send (e.g. an invoice), encoding the appropriate business semantics (e.g. UBL2.0) in an appropriate format (e.g. json).

The process is:

 * Create hash of the business document
 * Sign the business document
 * Create cyphertext of the signed business document
 * Create message.json file
 * Generate message.json signature
 * POST message.json and signature to the recipient TAP


### Sign the Business Document

The TAP protocol MAY be used to transport any business message in any format. It MAY
be used to transport messages compliant with Billing Semantics. Messages SHOULD be
formatted as XML or JSON and UTF-8 encoded.

Message signing is interpreted in the sense used by the OpenPGP standard (RFC4880),
however strict compliance would involve 3DES algorithm which is not supported.
Approved signing algorithms are those determined `--safe` by the *modern* (Eliptic
Curve Cryptography compatible) distribution of GnuPG. This is version 2.1.15 at the
time of writing, but any stable release at or above v2.1.15 is appropriate.

The following public key formats are supported:

 * ECC (RFC6637)
 * RSA (RFC4880)

Business documents compression is discouraged, because it is redundant due to
compression at the HTTP layer (per RFC2616 and RFC1952). Business documents MAY be
compressed with ZIP (RFC1951), ZLIB (RFC1950), or BZip2 algorithms.

The business document and signature are combined into a single ASCII-Armoured file
per RFC4880 (i.e. including use of Radix-64 to convert UTF-8 to 7 bit ASCII in the
signed file). For example, assuming the current working directory contains a business
document `doc.json`, a signed document `signed_doc.txt` can be created using the
GnuPG command line program like this:

```bash
gpg2 \
 --output "signed_doc.txt" \
 --clearsign "doc.json"
```

Please see (TODO: examples) for more detailed explanation and real-world examples.


### Create hash of signed business document

The following hash algorithms are approved for use:

 * SHA256 (default)
 * SHA384
 * SHA512
 * BLAKE2b
 * BLAKE2s

It is not necessary to sign the hash, because the entire message is signed.

This hash is of the cleartext "business document" before the signing. When the
recipient decrypts the business document, they are able to verify the hash. If the
recipient-generated hash does not match the hash in the message, the recipient MUST
NOT provide business acceptance of the document.

The hash is calculated before the signing, so, if sender for some reasons calculates
2 different signatures using 2 different keys - the hash is still the same.

Assuming the current working directory contains a document `doc.txt`, a hash of the
signed document `doc.hash` can be created with openssl like this:

```bash
openssl dgst -sha256 -out "doc.hash" "doc.txt"
```

For example, if your doc.txt contains text `{"document_type": "invoice"}` (ending
with newline character) the result is:

```
SHA256(doc.txt)= 14a62c86caa60bb3987248e93e51cacd46392562475738d3213b44f457ee163b
```

where `14a62c86caa60bb3987248e93e51cacd46392562475738d3213b44f457ee163b` is a
document hash in plan SHA256 HEX format.


#### Note about auditability

The TAP interface is used to exchange encrypted messages, but they are wrapped
in an envelope that contains cleartext which the TAP interface has access to.

If a 3rd party is presented with a copy of the message (including the cleartext
envelope), and with a copy of the signed business document, they are able to use
the "hash of the signed business document" to verify that the cleartext document
they have matches the contents of the cyphertext payload.

This is true even though they are unable to decrypt the cyphertext, because they
do not have access to the key material of the recipient.

This means that if a 3rd party has access to the encrypted traffic, while they
are unable to read the message payload, they are able to verify the contents of
a message if someone shares the cleartext with them. It is not possible for a
participant to make a false claim about the contents of messages.

One example of a 3rd party who would benefit from this ability to verify encrypted
payloads is a creditor providing an invoice factoring service. A borrower could
provide their lender with cleartext invoices (and other documents in a BILL-SPEC
conversation), and use the RESTRICT_LIST features of the `ausdigital-nry` to
provide their lender with access to the enctypted messages. The lender could
then confirm that the cleartext provided by the borrower matched the contents
of the encrypted message payloads, even though they do not have the ability to
decrypt them.

Another example of an interested 3rd party would be a value-adding reseller who
want's to prove the provonance of supply. They could do this by providing access
to messages (either with `ausdigital-nry` RESTRICT_LIST, or some other publishing
mechanism) as well as cleartext. Cleartext and messages could be passed down the
supply chain to prove provonance from source, and document value adding steps.


### Create cyphertext of signed business document

The message does not contain plaintext of the business document (signed or otherwise).
It contains the hash of the plaintext (as per above), and cyphertext of the signed
plaintext document.

The cyphertext is created using public key cryptography, using the appropriate public
key for the recipient business endpoint. The public parts of these keypairs are
discoverable using the appropriate DCP for each business identifier URNs, which is
discoverable using the global DCL. Public keys MUST be published in ASCII-Armoured
form in the DCP.

Example:

 * Alice sends document to Bob
 * Both Alice and Bob has public/private keypairs
 * Both Alice and Bob published their public keys to the DCP
 * Alice signs plaintext document by Alice's private key, having signed document as an output
 * Alice encyphers signed document using Bob's public key, having encyphered value as a result
 * Nobody except Bob can decypher the encyphered value (because only Bob owns his private keys)
 ...
 * When message is received, Bob decyphers the encyphered value using his public key
 * After that Bob validates the digital signature, using Alice's public key
 * If everything went fine Bob can be sure that:

   1. Only he decyphered the document
   2. This document has been signed by Alice.

Use of mature, well-known and extensively scrutinised cryptography implementations is
strongly encouraged. The following examples use GnuPG, although any compliant RFC4880
implementation could be used in an equivalent way.

Assuming the recipient's public key is not already in the sender's GnuPG keyring, but
is in the current working directory as `recipient.gpg`, it can be added to the sender's
GnuPG keyring like this:

```bash
gpg2 --import "recipient.gpg"
```

With GnuPG, this is only necessary once. Subsequent messages to the same recipient can
skip the key importing step.


#### Note about published keys

Implementation-specific section.

Once the recipient's key has been added to the sender's keyring, and assuming the
current working directory contains a signed document `signed_doc.txt`, and the user
namespace identifier of the recipient public key is
`91f68ffafa1288ad55cb3e61e937870fb5598cc098e125fe29412ab3047f15e1@dcp.testpoint.io`
then cyphertext of signed business document can be created with:

```
gpg2 --armour \
 --output "cyphertext.gpg"
 --encrypt \
 --recipient 91f68ffafa1288ad55cb3e61e937870fb5598cc098e125fe29412ab3047f15e1@dcp.testpoint.io \
 signed_doc.txt
```

This will create a file `cyphertext.gpg` in the current working directory, which
has ASCII-Armour encoding (suitable for inclusion in a json document).


### Message format

Typical TAP message is:

```
{
  "cyphertext": "string",
  "hash": "string",
  "sender": "string",
  "initiator": "string",
  "responder": "string",
  "reference": "string",
  "callbackURLs": ["string"]
}
```

 * "cyphertext" contains encrypted signed document
 * encrypted by receiver public key
 * signed by sender private key
 * "hash" contains cryptographic hash of the document (before signing and encryption)
 * "sender" contains unique participant identifier of client
 * sender must make his public keys, used for signing operations, available in the DCP
 * together "initiator" and "reference" form a candidate conversation identifier
 * initiator is the sender of the first message in a conversation
 * "responder" is the recipient of the first message in the conversation
 * all but the first message in a conversation must have a "responder"
 * "reference" is an identifier chosen by the initiator (e.g. "invoice-1234567")
 * the initiator must not reuse references, they are locally unique
 * "callbackURLs" are optional and requires if sender opts to receive technical callbacks

While the combination of "initiator" and "refernce" are a minimal candidate identifier
for the conversation, the conversation identifier is all three values together. Thus
each conversation has two valid identifiers (the first message with a null responder,
and all subsequent messages which have a non-null responder).


### Compose Message Example

The `message` part is a mixture of cleartext metadata (used by TAPs) and enciphered
payload (used by trusted business system components). The cleartext metadata does not
contain sensitive business information, whereas access to the business-sensitive
information within the payload is not necessary for participating in the TAP protocol.

The message part does not need to be in any kind of canonical form. It MUST be a
valid json document.

Assuming the current working directory contains:

 * `cyphertext.gpg`, containing encrypted signed business document in ASCII-Armour format
 * `signed_doc.hash`, containing digest of the signed business document
 * `reference.txt`, containing arbitrary string related to the document
 * `sender.txt`, containing the URN of the business identifier of the sender

Then the following python script will create `message.json` file containing the
cyphertext and associated metadata in json format.

```python
import json

def read_cyphertext(filename="cyphertext.gpg"):
    return open(filename, 'r').read().strip()

def read_hash(filename="signed_doc.hash"):
    return open(filename, 'r').read().strip()

def read_reference(filename="reference.txt"):
    return open(filename, 'r').read().strip()

def read_sender(filename="sender.txt"):
    return open(filename, 'r').read().strip()

def compose_message(filename="message.json", indent=4):
    message = {
        'cyphertext': read_cyphertext(),
        'hash': read_hash(),
        'reference': read_reference(),
        'sender': read_sender(),
    }
    result_file = open(filename, 'w')
    result_file.truncate()
    result_file.write(
        json.dumps(message, indent=indent)
    )
    result_file.write('\n')

if __name__ == "__main__":
    compose_message()
```


### Generate signature

The `signature` part is created by a business system component trusted by the sender
(with access to the sender's private key material). The signature can also be used to
uniquely identify the message contents.

Assuming the current working directory contains the message (as `message.json`), the
following command will create a signature `message.sig`:

```bash
gpg2 --output message.sig --sign message.json
```

Implementation note: the easiest way to make sure that specific key is used for
signature is to have only single private key in GPG keyring. GPG for linux support
custom keyring location.


### Posting the message to the recipient TAP

Layered on top of HTTP Protocol:

 * MUST use HTTPS (RFC2818).
 * MUST use only `Content-Type: multipart/form-data` (RFC2388)
 * MAY explicitly declare `Content-Transfer-Encoding: base64`
 * MUST NOT rely on additional TAP-related information in HTTP headers, such as message or conversation identifiers.

The message is sent to the receiving TAP using HTTP POST operation. The posted body
contains two parts, named `message` and `signature`.

Assuming the current working directory contains the message (as message.json) and
signature (as message.sig), the following curl command will post the message to the
recipient TAP (replace `<TAP_URL>` with HTTPS URL discovered from the Service
Metadata Publisher).

```bash
curl -X POST \
 -H "Content-Type: multipart/form-data" \
 -F "message=@message.json" \
 -F "signature=@message.sig" \
 <TAP_URL>
```

## Receipt and Technical Acknowledgement

When a valid message is received, the TAP issues an HTTP 200 status and returns
a response body with `Content-Type: application/json`, containing information
about the message and optional HATEOS-style list of callback URLs.
Check the [API](http://swagger.testpoint.io/?url=http://ausdigital.org/specs/ausdigital-tap/2.0/swagger.yaml){:target="_blank"} for possible
responses, success and errors.

TODO:

 * explain callback URLs
 * explain callback semantic URLs


# Related Material

 * AusDigital Transaction Access Point Implementation Guide (v1.0, available [here](https://github.com/ausdigital/ausdigital-tap/blob/master/docs/1.0/Access_Point_Implementation_Guide_v1.0.pdf)), which provides background to the [AusDigital](http://ausdigital.org) community process.
 * [GitHub issues](https://github.com/ausdigital/ausdigital-tap/issues/) for collaborating on the development of the TAP.
 * A reference [TAP service](http://testpoint.io/tap) (for testing and development purposes).
 * Free, Open-Source Software [TAP implementation](https://github.com/test-point/testpoint-tap).
 * An automated [TAP test suite](https://github.com/test-point/testpoint-tap).
