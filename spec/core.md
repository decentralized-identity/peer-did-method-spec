## Core Characteristics

### Method Name

The method name that identifies this DID method SHALL be: `peer`

A DID that uses this method MUST begin with the following prefix: `did:peer:`. Per the DID specification, this string MUST be in lowercase. The remainder of the DID, after the prefix, is the **method specific identifier (MSI)** described [below](#method-specific-identifier).

### Target System(s)

This DID method applies to any identity management implementation that meets the following requirement:

*   _Generation Algorithm_ — The DIDs are created using the [algorithm described below](#method-specific-identifier), endowing them with properties vital to trust between peers without incurring a dependency on a central source of truth.

### Method Specific Identifier

The peer DID scheme is defined by the following ABNF (see [RFC5234] for syntax):

::: example ABNF for peer DIDs

``` text
peer-did = "did:peer:" numalgo transform encnumbasis
numalgo = "0" / "1" / "2"
transform = "z"
encnumbasis = 46*BASE58BTC
```

:::

Peer DIDs use an underlying number with high entropy, called the numeric basis, as the source of their uniqueness. There are two methods for generating this number, as described in the [next section](#generation-method) of this spec. These methods are represented in the DID value with `numalgo` \== `0` or `numalgo` == `1`.

To build the full MSI, the raw bytes of the _numeric basis_ are prefixed with a [multicodec descriptor](https://github.com/multiformats/multicodec) that makes those bytes self-describing. This descriptor plus the raw bytes are then transformed to text in the manner associated with `transform`. This yields `encnumbasis` in the ABNF.

![composition of a peer DID value](https://raw.githubusercontent.com/decentralized-identity/peer-did-method-spec/master/composition.png)

Composition of a peer DID value

In this version of the spec, the multicodec descriptor for `numalgo` == `0` is dependent on key type and exactly matches the descriptor that would be used for the same value in `did:key`. The multicodec descriptor for `numalgo` == `1` is byte pair `0x12 0x20`, which says that what follows is a SHA2-256 hash, 32 bytes long. (The size is included because, although SHA-256 is known to produce 32-byte digests, other hash algorithms have variable-length output, and we could change the hash algorithm in a future version of the spec.) The `transform` is base58, represented by the [multibase prefix](https://github.com/multiformats/multibase/blob/master/README.md) `z`. `BASE58BTC` in the ABNF means the [Bitcoin base58 character inventory](https://en.bitcoin.it/wiki/Base58Check_encoding#Base58_symbol_chart); for brevity, we omit the expansion--but see the [matching regex](#matching-regex) for details.

The format of the DID value is intended to be future-proof, offering three expandable self-describing choices. Future versions of the spec may describe additional methods of generating the numeric basis; each new method should be associated with the next unused `numalgo` decimal digit when it is defined. Likewise, future versions of the spec may choose a different way to transform raw bytes to text, or a different format for the raw bytes of the numeric basis; these should be reflected in updated values of `transform` or of the multicodec descriptor at the front of the transformed bytes, respectively.

::: note
Base58 is case-sensitive; in this version of the spec, peer DIDs MUST be compared with a case-sensitive algorithm and MUST NOT be normalized in case. The more general rule is that case sensitivity derives from `transform`, so routines that process multiple versions of peer DIDs MUST derive their case sensitivity rules from the corresponding properties of whatever `transform` a particular peer DID uses.
:::

### Generation Method

The unique _numeric basis_ underlying a Peer DID MUST be generated in one of the following ways:

#### Method 0: inception key without doc

If `numalgo` == `0`, a single keypair is chosen, having all possible privileges with respect to the DID. The public key value for this pair is the numeric basis of the DID. The genesis version of the DID doc is considered to be empty, granting all privileges to this key (and algorithmic derivations thereof, to provide key agreement versus signing and so forth). The DID doc offers no endpoint. This makes the DID functionally equivalent to a did:key value, and visually similar, except that a peer DID will have the numeric algorithm as a prefix, before the multibase encoded, multicodec-encoded public key. For example, `did:key:z6MkpTHR8VNsBxYAAWHut2Geadd9jSwuBV8xRoAnwWsdvktH` is equivalent to `did:peer:0z6MkpTHR8VNsBxYAAWHut2Geadd9jSwuBV8xRoAnwWsdvktH`.

#### Method 1: genesis doc

*   Create a genesis version of JSON text of the DID doc for the DID. The genesis version MUST define a single key with the register privilege. This inception key is the key that creates the DID and authenticates when exchanging it with the first peer. It MUST NOT include the DID itself (either the root `id` property or its value). This lets the doc be created without knowing the DID's value in advance. Suppressing the DID value creates a stored variant of peer DID doc data, as opposed to the resolved variant that would have an actual DID value in the root `id` property. (In either the stored or resolved variant of the doc, anywhere else that the DID value would appear, it should appear as a relative reference rather than an absolute value. For example, each `controller` property of a `verificationMethod` that is owned by this DID would say `"controller": "#id"`.)
*   Calculate the SHA256 [RFC4634] hash of the bytes of the stored variant of the genesis version of the DID doc, and make this value the new DID's numeric basis.

By hashing the stored variant, we avoid the circular problem of including the DID in the data that's being hashed. This means that a peer DID doc must be resolved by converting a stored variant of DID doc data into a resolved variant by inserting the value of the DID being resolved.

The root of trust for peer DIDs is the entropy in the inception key. The inception key MUST be new for each DID; it MUST NOT ever be reused--and it must be the case that anyone observing the public half of the inception key cannot somehow derive or steal the private half. This guarantees that nobody other than the holder of the inception key could have created the genesis version of the DID doc. Because only the inception key has the privilege of sharing the DID doc with a peer, no other key holder can establish the relationship contrary to the intent of the inception key holder. This gives peer DIDs a self-certifying property that is vital to cybersecurity of all DIDs. Any DID method that does not guarantee the chain of custody of the DID between when it is created and when it is shared (e.g., written to a ledger or given to a peer) lacks this quality and is susceptible to attacks early in the DID's lifecycle. See [github ](https://github.com/openssi/peer-did-method-spec/issues/112) for more discussion.

#### Method 2: multiple inception key without doc

If `numalgo` == `2`, the generation mode is similar to Method 0 (and therefore also did:key) with the ability to specify additional keys in the generated DID Document. This method is necessary when both an encryption key and a signing key are required. This method also enables including services in the generated DID Document.

::: note
The first iteration of this method left some elements of the encoding and decoding process underspecified. Clarifications have been made to this specification to resolve this. These clarifications will be noted by appending "(clarified)" to the relevant statements.
:::

::: example ABNF for peer DIDs
```
peer-did-method-2 = "did:peer:2" 1*element 
element = "." ( purposecode transform encnumbasis / service )
purposecode = "A" / "E" / "V" / "I" / "D" / "S" 
keypurpose = 
transform = "z"
encnumbasis = 46*BASE58BTC
service = 1*B64URL
```
:::

##### Generating a `did:peer:2`

When generating a `did:peer:2`, take as inputs a set of keys and their purpose. Each key's purpose corresponds to the [Verification Relationship](https://www.w3.org/TR/did-core/#verification-relationships) it will hold in the DID Document generated from the DID.

Abstractly, these inputs may look like the following (in this example, the keys are already multibase, multicodec encoded values):

```json
[
  {
    "purpose": "verification",
    "publicKeyMultibase": "z6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc"
  },
  {
    "purpose": "encryption",
    "publicKeyMultibase": "z6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR"
  }
]
```

To encode the input keys:

* Construct a [multibase](https://github.com/multiformats/multibase), [multicodec](https://github.com/multiformats/multicodec) form of each public key to be included.
* Prefix each encoded key with a period character (.) and single character from the purpose codes table below.
* Concatenate the prefixed encoded keys. The inputs above will result in:
    ```
    .Vz6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc.Ez6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR
    ```

In addition to keys, `did:peer:2` can encode one or more [services](https://www.w3.org/TR/did-core/#services).

The service SHOULD follow the DID Core specification for services.

For use with `did:peer:2`, service `id` attributes MUST be relative. The service MAY omit the `id`; however, this is NOT RECOMMEDED (clarified).

Consider the following service as input:

```json
{
  "type": "DIDCommMessaging",
  "serviceEndpoint": {
    "uri": "http://example.com/didcomm",
    "accept": [
      "didcomm/v2"
    ],
    "routingKeys": [
      "did:example:123456789abcdefghi#key-1"
    ]
  }
}
```

To encode a service:

* Start with the JSON structure for your service, like the example input above.
* Recursively replace common strings in key names and type value with abbreviations from the abbreviations table below (clarified). For the above input, this will result in:
    ```
    {
      "t": "dm",
      "s": {
        "uri": "http://example.com/didcomm",
        "a": [
          "didcomm/v2"
        ],
        "r": [
          "did:example:123456789abcdefghi#key-1"
        ]
      }
    }
    ```

::: warning
Do not use the `dm` common string as the service type for DIDComm v1 `did:peer:2` DIDs.
Instead, use the full DIDComm v1 service type, `did-communication`.
:::

> The `dm` common string is the service type for the DIDComm v2 DID service. It
should **NOT** be used for a DIDComm v1 DID service. If you are defining a
`did:peer:2` DID Doc to be used with [DIDComm v1], make sure you use a [DIDComm
v1 service], and **DO NOT** use the `dm` common string, but rather use the
proper DIDComm v1 service type of `did-communication`. There is no common string
defined for `did-communication` so it **MUST** be spelled out in full.

[DIDComm v1]: https://github.com/hyperledger/aries-rfcs/tree/main/concepts/0005-didcomm
[DIDComm v1 service]: https://github.com/hyperledger/aries-rfcs/blob/main/features/0067-didcomm-diddoc-conventions/README.md#service-conventions

* Convert to string, and remove unnecessary whitespace, such as spaces and newlines. For the above input, this will result in:
    ```
    {"t":"dm","s":{"uri":"http://example.com/didcomm","a":["didcomm/v2"],"r":["did:example:123456789abcdefghi#key-1"]}}
    ```
* Base64URL Encode String (Padding MUST be removed as the "=" character is, per the DID Core Specification, not permitted in a DID). For the above example, this will result in:
    ```
    eyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9kaWRjb21tIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0xIl19fQ
    ```
* Prefix encoded service with a period character (.) and S. For the above input, this will result in:
    ```
    .SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9kaWRjb21tIiwiYWNjZXB0IjpbImRpZGNvbW0vdjIiXSwicm91dGluZ0tleXMiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0xIl19fQ
    ```
* For any additional services, repeat the steps above and concatenate each additional service to the previous (clarified). This will result in a string like the following (newline added between service strings for clarity; in practice, there will be no whitespace between the concatenated services):
    ```
    .SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9kaWRjb21tIiwiYWNjZXB0IjpbImRpZGNvbW0vdjIiXSwicm91dGluZ0tleXMiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0xIl19fQ
    .SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9hbm90aGVyIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0yIl19fQ
    ```
    When encoding multiple services, they MUST be encoded individually and concatenated together. The services MUST NOT be encoded as a JSON list.

Finally, to create the DID, concatenate the following values:

- `did:peer:2`
- Encoded, concatenated, and prefixed keys.
- Encoded, concatenated, and prefixed services.

This concatenation will result in a value like the following:

```
did:peer:2.Vz6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc.Ez6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR.SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9kaWRjb21tIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0xIl19fQ.SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9hbm90aGVyIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0yIl19fQ
```

##### Lookup Tables

###### Common String Abbreviations

| Common String         | Abbreviation  |
|-----------------------|---------------|
| type                  | t             |
| DIDCommMessaging      | dm            |
| serviceEndpoint       | s             |
| routingKeys           | r             |
| accept                | a             |

###### Purpose Codes

| Purpose Code | Verification Relationship     |
|--------------|-------------------------------|
| **A**        | Assertion                     |
| **E**        | Key Agreement (Encryption)    |
| **V**        | Authentication (Verification) |
| **I**        | Capability Invocation         |
| **D**        | Capability Delegation         |
| **S**        | Service                       |


##### Resolving a `did:peer:2`

::: note
Below is the normative approach to resolving did:peer:2 DIDs. However, it is often the case that the resolved value is only used within a particular component. In this case, some of these rules may be relaxed. For instance, the verification material may be transformed into a JWK or other representation if the resolving component is more accustomed to working with those key representations. It is not strictly necessary to first represent the document as described here only to immediately transform it into the more familiar representation. Shortcuts like these are not problematic unless they impact the function of the DID Document.

The `id` value of the Document, verification method, and service objects MUST be set as outlined below. These values are used to reference elements of the document and must be consistent across implementations regardless of shortcuts taken within the resolver.
:::

When Resolving the peer DID into a DID Document, the process is reversed:

* Start with an empty document with the DID Core context and [Multikey context](https://www.w3.org/TR/vc-data-integrity/#multikey) (clarified):
    ```json
    {
        "@context": ["https://www.w3.org/ns/did/v1", "https://w3id.org/security/multikey/v1"]
    }
    ```
* Set the `id` of the document to the DID being resolved.
* Optionally, set the `alsoKnownAs` to the `did:peer:3` value corresponding to the DID being resolved.
* Split the DID string into elements.
* For each element with a purpose corresponding to a key, transform keys into verification methods in the DID Document:
    * Remove the period (.) and the Purpose prefix. Consider the remaining string the "encoded key."
    * Create an empty object. Consider this value the "verification method."
    * Set the `type` of the verification method to [`Multikey`](https://www.w3.org/TR/vc-data-integrity/#multikey) (clarified).
    * Set the `id` of the verification method to `#key-N` where `N` is an incrementing number starting at `1` (clarified). The keys MUST be processed in the order they appear in the DID string. For example, if you have the DID (whitespace added for example only):
        ```
        did:peer:2
            .Vz6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc
            .Ez6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR
            .SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9kaWRjb21tIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0xIl19fQ
            .SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9hbm90aGVyIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0yIl19fQ
        ```
        The key with purpose code `V` and encoded key value `z6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc` will have the `id` `#key-1`.
        The key with purpose code `E` and encoded key value `z6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR` will have the `id` `#key-2`.
    * Set the `controller` of the verification method to the value of the DID being resolved.
    * Set the `publicKeyMultibase` of the verification method to the value of the encoded key.
    * Append this object to `verificationMethod` of the document (clarified).
    * Append a reference to the verification method to the appropriate verification relationship, using the lookup table above. The reference should be the exact value used in the `id` of the verification method (clarified). For example, using the DID above and the key with purpose code `V`:
        ```
        {
          ... // Context and other elements previously added to the document
          "authentication": [..., "#key-1"]
        }
        ```
* For each element with a purpose corresponding to a service, transform the service and add to the DID Document:
    * Remove the period (.) and S prefix.
    * Base64URL Decode String.
    * Parse as JSON.
    * Replace abbreviations in key names and type value with common names from the abbreviations table above.
    * If the `id` is NOT set (clarified):
        * Set `id` to `#service` for the first such service.
        * For all subsequent services WITHOUT an `id`, set `id` to `#service-1`, `#service-2`, etc. incrementing the integer with each service, starting from `1`.

::: note
It is  not possible to express a verification method not controlled by the controller of the DID Document with `did:peer:2`.
:::


::: example Example Peer DID 2

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/multikey/v1",
  ],
  "id": "did:peer:2.Vz6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc.Ez6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR.SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9kaWRjb21tIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0xIl19fQ.SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9hbm90aGVyIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0yIl19fQ",
  "verificationMethod": [
    {
      "id": "#key-1",
      "controller": "did:peer:2.Vz6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc.Ez6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR.SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9kaWRjb21tIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0xIl19fQ.SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9hbm90aGVyIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0yIl19fQ",
      "type": "Multikey",
      "publicKeyMultibase": "z6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc"
    },
    {
      "id": "#key-2",
      "controller": "did:peer:2.Vz6Mkj3PUd1WjvaDhNZhhhXQdz5UnZXmS7ehtx8bsPpD47kKc.Ez6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR.SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9kaWRjb21tIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0xIl19fQ.SeyJ0IjoiZG0iLCJzIjp7InVyaSI6Imh0dHA6Ly9leGFtcGxlLmNvbS9hbm90aGVyIiwiYSI6WyJkaWRjb21tL3YyIl0sInIiOlsiZGlkOmV4YW1wbGU6MTIzNDU2Nzg5YWJjZGVmZ2hpI2tleS0yIl19fQ",
      "type": "Multikey",
      "publicKeyMultibase": "z6LSg8zQom395jKLrGiBNruB9MM6V8PWuf2FpEy4uRFiqQBR"
    }
  ],
  "authentication": [
    "#key-1"
  ],
  "keyAgreement": [
    "#key-2"
  ],
  "service": [
    {
      "type": "DIDCommMessaging",
      "serviceEndpoint": {
        "uri": "http://example.com/didcomm",
        "accept": [
          "didcomm/v2"
        ],
        "routingKeys": [
          "did:example:123456789abcdefghi#key-1"
        ]
      },
      "id": "#service"
    },
    {
      "type": "DIDCommMessaging",
      "serviceEndpoint": {
        "uri": "http://example.com/another",
        "accept": [
          "didcomm/v2"
        ],
        "routingKeys": [
          "did:example:123456789abcdefghi#key-2"
        ]
      },
      "id": "#service-1"
    }
  ],
  "alsoKnownAs": ["did:peer:3zQmd6RdU6e2nDrLn1rjwdA5Buzq7wJwsv3WJ1AgrwKYJoLE"]
}
```

:::

#### Method 3: DID Shortening with SHA-256 Hash

If `numalgo` == `3`, the generation mode is similar to Method 2, but with a shorter DID identifier derived from a SHA-256 hash of the original identifier. The benefit of using Method 3 over Method 2 is the ability to have smaller size didcomm messages as `did:peer:2.` dids tend to be verbose in nature. Method 3 peer dids can only be used after a peer did method 2 has been exchange with the other party and thus can map the shortened did to the longform one. In order to send a message encrypted with method 3 you first MUST send a discover-feature message (using the method 2 as the `to` field) to make sure that the receiving agent is capable of resolving method 3 dids.

::: example ABNF for peer DIDs numalgo 3
```
peer-did-method-3 = "did:peer:3" transform encnumbasis
transform = "z"
encnumbasis = 46*BASE58BTC
```
:::

*   Start with the DID generated using Method 2.
*   Take the SHA-256 hash of the generated DID (excluding the "did:peer:2" prefix).
*   Encode the hash using the base58btc multibase encoding.
*   Construct the final Method 3 DID by concatenating the prefix "did:peer:3" with the encoded hash.

For example, if the Method 2 DID is:

```
did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0
```

First, remove the prefix "did:peer:2":

```
.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0
```

Take the SHA-256 hash of the remaining string and represent as a multi-hash, multi-base encoded(base58btc) string:

```
zQmS19jtYDvGtKVrJhQnRFpBQAx3pJ9omx2HpNrcXFuRCz9
```

Finally, concatenate the prefix "did:peer:3" with the computed and encoded hash:

```
did:peer:3zQmS19jtYDvGtKVrJhQnRFpBQAx3pJ9omx2HpNrcXFuRCz9
```
#### Method 4: Short Form and Long Form

DID Peer Numalgo 4 is a statically resolvable DID Method with a short form and a long form. The short form is the hash over the long form. The combined use of short and long forms allows for fully peer shared DID Documents, with efficient use of the short form after initial exchange.

##### Creating a DID

To create a `did:peer:4` DID, you must start with a document which is very similar in structure to DID Documents. This document is referred to as the "Input Document." This document should look almost exactly like the final resolved DID Document you desire but with a few key differences:

- The document MUST NOT include an `id` at the root. For DID Documents, this is populated with the DID itself. Since we are in the process of generating a DID, we do not yet know the value of the DID. When the DID is resolved later, this value will be correctly filled in.
- All identifiers within this document MUST be relative. For example, the `id` of a `verificationMethod` might be `#key-1` instead of something like `did:example:abc123#key-1`.
- All references pointing to resources within this document MUST be relative. For example, a verification method reference in a verification relationship such as `authentication` might be `#key-1` instead of something like `did:example:abc123#key-1`.
- For verification methods, the `controller` MUST be omitted if the controller is the document owner. If it is controlled by a DID other than the owner of the document, it MUST be included.

For this tutorial, consider an Input Document like the following:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/x25519-2020/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "verificationMethod": [
    {
      "id": "#6LSqPZfn",
      "type": "X25519KeyAgreementKey2020",
      "publicKeyMultibase": "z6LSqPZfn9krvgXma2icTMKf2uVcYhKXsudCmPoUzqGYW24U"
    },
    {
      "id": "#6MkrCD1c",
      "type": "Ed25519VerificationKey2020",
      "publicKeyMultibase": "z6MkrCD1csqtgdj8sjrsu8jxcbeyP6m7LiK87NzhfWqio5yr"
    }
  ],
  "authentication": [
    "#6MkrCD1c"
  ],
  "assertionMethod": [
    "#6MkrCD1c"
  ],
  "keyAgreement": [
    "#6LSqPZfn"
  ],
  "capabilityInvocation": [
    "#6MkrCD1c"
  ],
  "capabilityDelegation": [
    "#6MkrCD1c"
  ],
  "service": [
    {
      "id": "#didcommmessaging-0",
      "type": "DIDCommMessaging",
      "serviceEndpoint": {
        "uri": "didcomm://queue",
        "accept": ["didcomm/v2"],
        "routingKeys": [],
      }
    }
  ]
}
```
To encode this value into a `did:peer:4`:

1. Encode the document:
    1. JSON stringify the object without whitespace
    2. Encode the string as utf-8 bytes
    3. Prefix the bytes with the [multicodec](https://github.com/multiformats/multicodec) prefix for json ([varint](https://github.com/multiformats/unsigned-varint) `0x0200`)
    4. [Multibase](https://github.com/multiformats/multibase) encode the bytes as [base58btc](https://datatracker.ietf.org/doc/html/draft-msporny-base58-03) (base58 encode the value and prefix with a `z`)
    5. Consider this value the `encoded document`
2. Hash the document:
    1. Take SHA2-256 digest of the encoded document (encode the bytes as utf-8)
    2. Prefix these bytes with the [multihash](https://github.com/multiformats/multihash) prefix for SHA2-256 and the hash length (varint `0x12` for prefix, varint `0x20` for 32 bytes in length)
    3. Multibase encode the bytes as base58btc (base58 encode the value and prefix with a `z`)
    4. Consider this value the `hash`
3. Construct the did by concatenating the values as follows:

        did:peer:4{{hash}}:{{encoded document}}

Here is an example long form DID made from the input example above:

```
did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd:z2M1k7h4psgp4CmJcnQn2Ljp7Pz7ktsd7oBhMU3dWY5s4fhFNj17qcRTQ427C7QHNT6cQ7T3XfRh35Q2GhaNFZmWHVFq4vL7F8nm36PA9Y96DvdrUiRUaiCuXnBFrn1o7mxFZAx14JL4t8vUWpuDPwQuddVo1T8myRiVH7wdxuoYbsva5x6idEpCQydJdFjiHGCpNc2UtjzPQ8awSXkctGCnBmgkhrj5gto3D4i3EREXYq4Z8r2cWGBr2UzbSmnxW2BuYddFo9Yfm6mKjtJyLpF74ytqrF5xtf84MnGFg1hMBmh1xVx1JwjZ2BeMJs7mNS8DTZhKC7KH38EgqDtUZzfjhpjmmUfkXg2KFEA3EGbbVm1DPqQXayPYKAsYPS9AyKkcQ3fzWafLPP93UfNhtUPL8JW5pMcSV3P8v6j3vPXqnnGknNyBprD6YGUVtgLiAqDBDUF3LSxFQJCVYYtghMTv8WuSw9h1a1SRFrDQLGHE4UrkgoRvwaGWr64aM87T1eVGkP5Dt4L1AbboeK2ceLArPScrdYGTpi3BpTkLwZCdjdiFSfTy9okL1YNRARqUf2wm8DvkVGUU7u5nQA3ZMaXWJAewk6k1YUxKd7LvofGUK4YEDtoxN5vb6r1Q2godrGqaPkjfL3RoYPpDYymf9XhcgG8Kx3DZaA6cyTs24t45KxYAfeCw4wqUpCH9HbpD78TbEUr9PPAsJgXBvBj2VVsxnr7FKbK4KykGcg1W8M1JPz21Z4Y72LWgGQCmixovrkHktcTX1uNHjAvKBqVD5C7XmVfHgXCHj7djCh3vzLNuVLtEED8J1hhqsB1oCBGiuh3xXr7fZ9wUjJCQ1HYHqxLJKdYKtoCiPmgKM7etVftXkmTFETZmpM19aRyih3bao76LdpQtbw636r7a3qt8v4WfxsXJetSL8c7t24SqQBcAY89FBsbEnFNrQCMK3JEseKHVaU388ctvRD45uQfe5GndFxthj4iSDomk4uRFd1uRbywoP1tRuabHTDX42UxPjz
```

To construct the short form, simply omit the `:{{encoded document}}` from the end.

Here is an example short form DID for the long form above:

```
did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd
```

##### Resolving a DID

###### Long form

Resolving a long form `did:peer:4` document is done by decoding the document from the DID and "contextualizing" the restored Input Document with the DID.

To decode the document:

1. Extract the `encoded document` portion of the DID
2. Verify the hash over the `encoded document` by extracting the `hash` portion of the DID and comparing it against the result of following step 2 ("Hash the document") above to recreate the hash.
3. Perform the inverse of step 1 ("Encode the document") to get the decoded document

To "contextualize" a document:

1. Take the decoded document (which should look identical to the input example above)
2. Add `id` at the root of the document and set it to the DID
3. Add `alsoKnownAs` at the root of the document and set it to a list, if not already present, and append the short form of the DID
4. For each verification method (declared in the `verificationMethod` section or embedded in a verification relationship like `authentication`):
    - If `controller` is not set, set `controller` to the DID

> Note: Implementations may turn relative references in the document into absolute references by prepending the reference with the DID. This is not recommended due to length but this is an implementation detail that should not affect usage of the resolved document. Both relative and absolute references are valid within DID Documents.

Here is an example long form DID Document:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/x25519-2020/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "verificationMethod": [
    {
      "id": "#6LSqPZfn",
      "type": "X25519KeyAgreementKey2020",
      "publicKeyMultibase": "z6LSqPZfn9krvgXma2icTMKf2uVcYhKXsudCmPoUzqGYW24U",
      "controller": "did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd:z2M1k7h4psgp4CmJcnQn2Ljp7Pz7ktsd7oBhMU3dWY5s4fhFNj17qcRTQ427C7QHNT6cQ7T3XfRh35Q2GhaNFZmWHVFq4vL7F8nm36PA9Y96DvdrUiRUaiCuXnBFrn1o7mxFZAx14JL4t8vUWpuDPwQuddVo1T8myRiVH7wdxuoYbsva5x6idEpCQydJdFjiHGCpNc2UtjzPQ8awSXkctGCnBmgkhrj5gto3D4i3EREXYq4Z8r2cWGBr2UzbSmnxW2BuYddFo9Yfm6mKjtJyLpF74ytqrF5xtf84MnGFg1hMBmh1xVx1JwjZ2BeMJs7mNS8DTZhKC7KH38EgqDtUZzfjhpjmmUfkXg2KFEA3EGbbVm1DPqQXayPYKAsYPS9AyKkcQ3fzWafLPP93UfNhtUPL8JW5pMcSV3P8v6j3vPXqnnGknNyBprD6YGUVtgLiAqDBDUF3LSxFQJCVYYtghMTv8WuSw9h1a1SRFrDQLGHE4UrkgoRvwaGWr64aM87T1eVGkP5Dt4L1AbboeK2ceLArPScrdYGTpi3BpTkLwZCdjdiFSfTy9okL1YNRARqUf2wm8DvkVGUU7u5nQA3ZMaXWJAewk6k1YUxKd7LvofGUK4YEDtoxN5vb6r1Q2godrGqaPkjfL3RoYPpDYymf9XhcgG8Kx3DZaA6cyTs24t45KxYAfeCw4wqUpCH9HbpD78TbEUr9PPAsJgXBvBj2VVsxnr7FKbK4KykGcg1W8M1JPz21Z4Y72LWgGQCmixovrkHktcTX1uNHjAvKBqVD5C7XmVfHgXCHj7djCh3vzLNuVLtEED8J1hhqsB1oCBGiuh3xXr7fZ9wUjJCQ1HYHqxLJKdYKtoCiPmgKM7etVftXkmTFETZmpM19aRyih3bao76LdpQtbw636r7a3qt8v4WfxsXJetSL8c7t24SqQBcAY89FBsbEnFNrQCMK3JEseKHVaU388ctvRD45uQfe5GndFxthj4iSDomk4uRFd1uRbywoP1tRuabHTDX42UxPjz"
    },
    {
      "id": "#6MkrCD1c",
      "type": "Ed25519VerificationKey2020",
      "publicKeyMultibase": "z6MkrCD1csqtgdj8sjrsu8jxcbeyP6m7LiK87NzhfWqio5yr",
      "controller": "did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd:z2M1k7h4psgp4CmJcnQn2Ljp7Pz7ktsd7oBhMU3dWY5s4fhFNj17qcRTQ427C7QHNT6cQ7T3XfRh35Q2GhaNFZmWHVFq4vL7F8nm36PA9Y96DvdrUiRUaiCuXnBFrn1o7mxFZAx14JL4t8vUWpuDPwQuddVo1T8myRiVH7wdxuoYbsva5x6idEpCQydJdFjiHGCpNc2UtjzPQ8awSXkctGCnBmgkhrj5gto3D4i3EREXYq4Z8r2cWGBr2UzbSmnxW2BuYddFo9Yfm6mKjtJyLpF74ytqrF5xtf84MnGFg1hMBmh1xVx1JwjZ2BeMJs7mNS8DTZhKC7KH38EgqDtUZzfjhpjmmUfkXg2KFEA3EGbbVm1DPqQXayPYKAsYPS9AyKkcQ3fzWafLPP93UfNhtUPL8JW5pMcSV3P8v6j3vPXqnnGknNyBprD6YGUVtgLiAqDBDUF3LSxFQJCVYYtghMTv8WuSw9h1a1SRFrDQLGHE4UrkgoRvwaGWr64aM87T1eVGkP5Dt4L1AbboeK2ceLArPScrdYGTpi3BpTkLwZCdjdiFSfTy9okL1YNRARqUf2wm8DvkVGUU7u5nQA3ZMaXWJAewk6k1YUxKd7LvofGUK4YEDtoxN5vb6r1Q2godrGqaPkjfL3RoYPpDYymf9XhcgG8Kx3DZaA6cyTs24t45KxYAfeCw4wqUpCH9HbpD78TbEUr9PPAsJgXBvBj2VVsxnr7FKbK4KykGcg1W8M1JPz21Z4Y72LWgGQCmixovrkHktcTX1uNHjAvKBqVD5C7XmVfHgXCHj7djCh3vzLNuVLtEED8J1hhqsB1oCBGiuh3xXr7fZ9wUjJCQ1HYHqxLJKdYKtoCiPmgKM7etVftXkmTFETZmpM19aRyih3bao76LdpQtbw636r7a3qt8v4WfxsXJetSL8c7t24SqQBcAY89FBsbEnFNrQCMK3JEseKHVaU388ctvRD45uQfe5GndFxthj4iSDomk4uRFd1uRbywoP1tRuabHTDX42UxPjz"
    }
  ],
  "service": [
    {
      "id": "#didcommmessaging-0",
      "type": "DIDCommMessaging",
      "serviceEndpoint": {
        "uri": "didcomm:transport/queue",
        "accept": [
          "didcomm/v2"
        ],
        "routingKeys": []
      }
    }
  ],
  "authentication": [
    "#6MkrCD1c"
  ],
  "keyAgreement": [
    "#6LSqPZfn"
  ],
  "assertionMethod": [
    "#6MkrCD1c"
  ],
  "capabilityDelegation": [
    "#6MkrCD1c"
  ],
  "capabilityInvocation": [
    "#6MkrCD1c"
  ],
  "alsoKnownAs": [
    "did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd"
  ],
  "id": "did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd:z2M1k7h4psgp4CmJcnQn2Ljp7Pz7ktsd7oBhMU3dWY5s4fhFNj17qcRTQ427C7QHNT6cQ7T3XfRh35Q2GhaNFZmWHVFq4vL7F8nm36PA9Y96DvdrUiRUaiCuXnBFrn1o7mxFZAx14JL4t8vUWpuDPwQuddVo1T8myRiVH7wdxuoYbsva5x6idEpCQydJdFjiHGCpNc2UtjzPQ8awSXkctGCnBmgkhrj5gto3D4i3EREXYq4Z8r2cWGBr2UzbSmnxW2BuYddFo9Yfm6mKjtJyLpF74ytqrF5xtf84MnGFg1hMBmh1xVx1JwjZ2BeMJs7mNS8DTZhKC7KH38EgqDtUZzfjhpjmmUfkXg2KFEA3EGbbVm1DPqQXayPYKAsYPS9AyKkcQ3fzWafLPP93UfNhtUPL8JW5pMcSV3P8v6j3vPXqnnGknNyBprD6YGUVtgLiAqDBDUF3LSxFQJCVYYtghMTv8WuSw9h1a1SRFrDQLGHE4UrkgoRvwaGWr64aM87T1eVGkP5Dt4L1AbboeK2ceLArPScrdYGTpi3BpTkLwZCdjdiFSfTy9okL1YNRARqUf2wm8DvkVGUU7u5nQA3ZMaXWJAewk6k1YUxKd7LvofGUK4YEDtoxN5vb6r1Q2godrGqaPkjfL3RoYPpDYymf9XhcgG8Kx3DZaA6cyTs24t45KxYAfeCw4wqUpCH9HbpD78TbEUr9PPAsJgXBvBj2VVsxnr7FKbK4KykGcg1W8M1JPz21Z4Y72LWgGQCmixovrkHktcTX1uNHjAvKBqVD5C7XmVfHgXCHj7djCh3vzLNuVLtEED8J1hhqsB1oCBGiuh3xXr7fZ9wUjJCQ1HYHqxLJKdYKtoCiPmgKM7etVftXkmTFETZmpM19aRyih3bao76LdpQtbw636r7a3qt8v4WfxsXJetSL8c7t24SqQBcAY89FBsbEnFNrQCMK3JEseKHVaU388ctvRD45uQfe5GndFxthj4iSDomk4uRFd1uRbywoP1tRuabHTDX42UxPjz"
}
```

###### Short form
To resolve a short form `did:peer:4` DID, you must know the corresponding long form DID. It is not possible to resolve a short form `did:peer:4` without first seeing and storing it's long form counterpart.

To resolve a short form DID, retrieve the saved long form DID or decoded document (like the Input Document such as in the example above) and follow the same rules described in the [long form](#long-form) section to "contextualize" the document but using the short form DID instead of the long form DID.

Here is an example short form DID Document:

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/x25519-2020/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1"
  ],
  "verificationMethod": [
    {
      "id": "#6LSqPZfn",
      "type": "X25519KeyAgreementKey2020",
      "publicKeyMultibase": "z6LSqPZfn9krvgXma2icTMKf2uVcYhKXsudCmPoUzqGYW24U",
      "controller": "did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd"
    },
    {
      "id": "#6MkrCD1c",
      "type": "Ed25519VerificationKey2020",
      "publicKeyMultibase": "z6MkrCD1csqtgdj8sjrsu8jxcbeyP6m7LiK87NzhfWqio5yr",
      "controller": "did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd"
    }
  ],
  "service": [
    {
      "id": "#didcommmessaging-0",
      "type": "DIDCommMessaging",
      "serviceEndpoint": {
        "uri": "didcomm:transport/queue",
        "accept": [
          "didcomm/v2"
        ],
        "routingKeys": []
      }
    }
  ],
  "authentication": [
    "#6MkrCD1c"
  ],
  "keyAgreement": [
    "#6LSqPZfn"
  ],
  "assertionMethod": [
    "#6MkrCD1c"
  ],
  "capabilityDelegation": [
    "#6MkrCD1c"
  ],
  "capabilityInvocation": [
    "#6MkrCD1c"
  ],
  "alsoKnownAs": [
    "did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd:z2M1k7h4psgp4CmJcnQn2Ljp7Pz7ktsd7oBhMU3dWY5s4fhFNj17qcRTQ427C7QHNT6cQ7T3XfRh35Q2GhaNFZmWHVFq4vL7F8nm36PA9Y96DvdrUiRUaiCuXnBFrn1o7mxFZAx14JL4t8vUWpuDPwQuddVo1T8myRiVH7wdxuoYbsva5x6idEpCQydJdFjiHGCpNc2UtjzPQ8awSXkctGCnBmgkhrj5gto3D4i3EREXYq4Z8r2cWGBr2UzbSmnxW2BuYddFo9Yfm6mKjtJyLpF74ytqrF5xtf84MnGFg1hMBmh1xVx1JwjZ2BeMJs7mNS8DTZhKC7KH38EgqDtUZzfjhpjmmUfkXg2KFEA3EGbbVm1DPqQXayPYKAsYPS9AyKkcQ3fzWafLPP93UfNhtUPL8JW5pMcSV3P8v6j3vPXqnnGknNyBprD6YGUVtgLiAqDBDUF3LSxFQJCVYYtghMTv8WuSw9h1a1SRFrDQLGHE4UrkgoRvwaGWr64aM87T1eVGkP5Dt4L1AbboeK2ceLArPScrdYGTpi3BpTkLwZCdjdiFSfTy9okL1YNRARqUf2wm8DvkVGUU7u5nQA3ZMaXWJAewk6k1YUxKd7LvofGUK4YEDtoxN5vb6r1Q2godrGqaPkjfL3RoYPpDYymf9XhcgG8Kx3DZaA6cyTs24t45KxYAfeCw4wqUpCH9HbpD78TbEUr9PPAsJgXBvBj2VVsxnr7FKbK4KykGcg1W8M1JPz21Z4Y72LWgGQCmixovrkHktcTX1uNHjAvKBqVD5C7XmVfHgXCHj7djCh3vzLNuVLtEED8J1hhqsB1oCBGiuh3xXr7fZ9wUjJCQ1HYHqxLJKdYKtoCiPmgKM7etVftXkmTFETZmpM19aRyih3bao76LdpQtbw636r7a3qt8v4WfxsXJetSL8c7t24SqQBcAY89FBsbEnFNrQCMK3JEseKHVaU388ctvRD45uQfe5GndFxthj4iSDomk4uRFd1uRbywoP1tRuabHTDX42UxPjz"
  ],
  "id": "did:peer:4zQmd8CpeFPci817KDsbSAKWcXAE2mjvCQSasRewvbSF54Bd"
}
```
##### Reference implementation

https://github.com/dbluhm/did-peer-4


### Recognizing and handling peer DIDs

*This section is non-normative.*

::: example A typical peer DID
``` text
did:peer:1zQmZMygzYqNwU6Uhmewx5Xepf2VLp5S4HLSwwgf2aiKZuwa
```
:::

Peer DIDs consist entirely of printable ASCII characters and in this version of the spec are exactly 57 or 58 characters long. They have no whitespace, and the only punctuation they use is the `:` character. When rendering in columns with a constrained width, they could be hyphenated one or more times as needed; the hyphens would not be confused with meaningful delimiters, and the final character of the DID would be easy to find. They could also be rendered in a short form with the middle of the long base58 section elided, if it is not necessary to compare them with precision: `did:peer:1z6Tx...WzJJq`. The ellipsis should not obscure the prefix or the initial or final few characters of `encnumbasis`, to preserve enough info for casual distinction. This might be a useful rendering in log files, for example.

A convenient regex to match `peer` DIDs is:

::: example Matching regex
``` text
^did:peer:(([01](z)([1-9a-km-zA-HJ-NP-Z]{46,47}))|(2((\.[AEVID](z)([1-9a-km-zA-HJ-NP-Z]{46,47}))+(\.(S)[0-9a-zA-Z=]*)*)))$
```
:::

A match against this regex places `numalgo` (the algorithm for choosing a numeric basis) in capture group 1, `transform` in capture group 2, and `encnumbasis` in capture group 3.

### CRUD Operations

`peer` DIDs implement the following CRUD (Create, Read, Update, and Delete) operations:

*   Create: A `peer` DID is created by `numalgo` as defined in the [Generation Methods](#generation-methods) section of this specification. Once created, the DIDs are shared by the party creating the DID with anyone they want to have and use the DID.
*   Read: Those holding `peer` DIDs store them locally and "resolve" them by looking up their DIDDoc by the `peer` DID identifier string. In [DIDComm](https://identity.foundation/didcomm-messaging/spec/v2.0/), agents may index the DIDs by the keys within the DIDs and look up DIDs via their keys. As noted earlier, `peer` DIDs are not resolvable by anyone other than those that have received the DIDs.
*   Update: `peer` DIDs cannot be updated. Implementations such as [DIDComm](https://identity.foundation/didcomm-messaging/spec/v2.0/) use DID Rotation instead of updating the contents of the DIDDoc when necessasry.
*   Delete: `peer` DIDs are deleted by those holding the DIDs. When a party is no longer interested in participating in the capabilities enabled by the sharing of the DIDs, they can simply delete their local copy, without without notifying other parties using the `peer`.
