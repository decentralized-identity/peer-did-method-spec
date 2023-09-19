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

If `numalgo` == `2`, the generation mode is similar to Method 0 (and therefore also did:key) with the ability to specify additional keys in the generated DID Document. This method is necessary when both an encryption key and a signing key are required.

::: example ABNF for peer DIDs
``` json
peer-did-method-2 = "did:peer:2" 1*element 
element = "." ( purposecode transform encnumbasis / service )
purposecode = "A" / "E" / "V" / "I" / "D" / "S" 
keypurpose = 
transform = "z"
encnumbasis = 46*BASE58BTC
service = 1*B64URL
```
:::

*   Start with the did prefix
    
    `did:peer:2`
    
*   Construct a multibase encoded, multicodec-encoded form of each public key to be included.
*   Prefix each encoded key with a period character (.) and single character from the purpose codes table below.
*   Append the encoded key to the DID.
*   Encode and append a service type to the end of the peer DID if desired as described below.

Service encoding

*   Start with the JSON structure for your service.
*   Replace common strings in key names and type value with abbreviations from the abbreviations table below.
*   Convert to string, and remove unnecessary whitespace, such as spaces and newlines.
*   Base64URL Encode String (Padding MUST be removed as the "=" character is, per the DID Core Specification, not permitted in a DID)
*   Prefix encoded service with a period character (.) and S

Service decoding

*   Remove the period (.) and S prefix
*   Base64URL Decode String
*   Parse as JSON.
*   Replace abbreviations in key names and type value with common names from the abbreviations table below.
*   Add id attribute according to the form
    
    #service
    

Common String Abbreviations

| Common String         | Abbreviation  |
|-----------------------|---------------|
| type                  | t             |
| DIDCommMessaging      | dm            |
| serviceEndpoint       | s             |
| routingKeys           | r             |
| accept                | a             |

Purpose Code List

*   **A** - Assertion
*   **E** - Encryption (Key Agreement)
*   **V** - Verification
*   **I** - Capability Invocation
*   **D** - Capability Delegation
*   **S** - Service

::: example Example Multi-key Peer DID
Encoded Encryption Key: 

``` text
.Ez6MkpTHR8VNsBxYAAWHut2Geadd9jSwuBV8xRoAnwWsdvktH
```

Encoded Signing Key: 

``` text
.VzXwpBnMdCm1cLmKuzgESn29nqnonp1ioqrQMRHNsmjMyppzx8xB2pv7cw8q1PdDacSrdWE3dtB9f7Nxk886mdzNFoPtY
```

Service Block:
``` json
{
    "type": "DIDCommMessaging",
    "serviceEndpoint": "https://example.com/endpoint",
    "routingKeys": ["did:example:somemediator#somekey"],
    "accept": ["didcomm/v2", "didcomm/aip2;env=rfc587"]
}
```

Service Block, after whitespace removal and common word substitution:

``` json
{"t":"dm","s":"https://example.com/endpoint","r":["did:example:somemediator#somekey"],"a":["didcomm/v2","didcomm/aip2;env=rfc587"]}
```

Encoded Service Endpoint:

``` text
.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0
```

Method 2 peer DID:

``` text
did:peer:2.Ez6MkpTHR8VNsBxYAAWHut2Geadd9jSwuBV8xRoAnwWsdvktH.VzXwpBnMdCm1cLmKuzgESn29nqnonp1ioqrQMRHNsmjMyppzx8xB2pv7cw8q1PdDacSrdWE3dtB9f7Nxk886mdzNFoPtY.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0
```
:::

When Resolving the peer DID into a DID Document, the process is reversed.

*   Split the DID string into element.
*   Extract element purpose and decode each key or service.
*   Insert each key or service into the document according to the designated purpose.

::: example Example Multi-key Peer DID

``` json
{
   "@context": "https://w3id.org/did/v1",
   "id": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0",
   "authentication": [
       {
           "id": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0#6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V",
           "type": "Ed25519VerificationKey2020",
           "controller": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0",
           "publicKeyMultibase": "z6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V"
       },
       {
           "id": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0#6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg",
           "type": "Ed25519VerificationKey2020",
           "controller": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0",
           "publicKeyMultibase": "z6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg"
       }
   ],
   "keyAgreement": [
       {
           "id": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0#6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc",
           "type": "X25519KeyAgreementKey2020",
           "controller": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0",
           "publicKeyMultibase": "z6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc"
       }
   ],
   "service": [
       {
           "id": "did:peer:2.Ez6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc.Vz6MkqRYqQiSgvZQdnBytw86Qbs2ZWUkGv22od935YF4s8M7V.Vz6MkgoLTnTypo3tDRwCkZXSccTPHRLhF4ZnjhueYAFpEX6vg.SeyJ0IjoiZG0iLCJzIjoiaHR0cHM6Ly9leGFtcGxlLmNvbS9lbmRwb2ludCIsInIiOlsiZGlkOmV4YW1wbGU6c29tZW1lZGlhdG9yI3NvbWVrZXkiXSwiYSI6WyJkaWRjb21tL3YyIiwiZGlkY29tbS9haXAyO2Vudj1yZmM1ODciXX0#didcommmessaging-0",
           "type": "DIDCommMessaging",
           "serviceEndpoint": "https://example.com/endpoint",
           "routingKeys": [
               "did:example:somemediator#somekey"
           ],
           "accept": [
                "didcomm/v2", "didcomm/aip2;env=rfc587"
           ]
       }
   ]
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
      "serviceEndpoint": "didcomm://queue",
      "accept": [
        "didcomm/v2"
      ],
      "routingKeys": []
    }
  ]
}
```
To encode this value into a `did:peer:4`:

1. Encode the document:
    1. JSON stringify the object without whitespace
    2. Encode the string as utf-8 bytes
    3. Prefix the bytes with the multicodec prefix for json ([varint](https://github.com/multiformats/unsigned-varint) `0x0200`)
    4. Multibase encode the bytes as base58btc (base58 encode the value and prefix with a `z`)
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
did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P:z3U8mHxj7WY8i5EQD5rmkfPaDTws9dw5NRERbsFhsDFh5kvAEAY1afMFTWWbZT2d7TjCYtGBt47e1WpYoXYKSoEjxQX9qhNSw2ujMPiHn5rVyVCwQH1kBLzs95n37MoD6ikVDd29fkJoZaBrGhNaxcFiW9b1N85NWGQ9vs1LEMxsqt5ZHiFXGp1tMU85Da4VETb7veFZjvxRsadgbgeK1tFG83c6SSyDjoVzQbagXJ44kZXaMMmbs4Kk6VJszrCM1CUHPk7kFgHmojWU8kFAmPE88oesxBW5Wdc5cCNB6cB4QzRGuYrkK15hMdoFWq4kVHJn1xbruCzJ7Mb9f2rKUF6KTiUbTYpHuo4Kbnu26tJKQ9D7xHkAM2N3ZPN3eRkXBefKme5mLhGjRgXm6fAZiCHuK4dMyg4Bd9HDXiy8vSdY8cyZnyuJdPsjq5FRvRD92cFNtJZBJJwRQu6WiwKhTL9jELwwGfU2jukeESmARHjpQRTkXhtyG5NHDwj3Yx9CsbyBR5xdGsB3raA8JiMP4nAsbZhfXiBErBUx4MwYRnBDZERZztPjJWJniyKVG6hfoBokzEtkZt6gYMh1tpjsBAcSVw4C9H7o7QrY3mW6DjSufDdHSdWPVJjfHgRzxUM218CSiEwqEctqxJP9fc2FVSDxai7JUnroVzgYzhb62S4ueLGKM83abkd3Fm5NeSuewPRbgwETTLvknz1Wq1G4qygq75Fp3Kr21qknM2tsgrkwyprYR9ZTK5YzY5sHCwNP14VXZeX24QdSfevspNdvFtFtiDq6dUufmy5bKeLdHxx7Mpb7vFToU8bk9zZNUkcgXvX12U6iT1zLEyszTS6B3csHRr1HmvLUgEQKfd2aWjV2ScktEBsjZRHWdWuxQRcs85sF92kW2fVX7k1EGAwYGsnr6Wf9Q7jkM7SgJM5WJ1rHHsxKXwj8j11QmwXRcgREVEJphWdj87tpsC36A4rfEYhyJDw13UB68JgoK544NbsA
```

To construct the short form, simply omit the `:{{encoded document}}` from the end.

Here is an example short form DID for the long form above:

```
did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P
```

##### Resolving a DID

###### Long form

Resolving a long form `did:peer:4` document is done by decoding the document from the DID and "contextualizing" the document with the DID.

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
      "controller": "did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P:z3U8mHxj7WY8i5EQD5rmkfPaDTws9dw5NRERbsFhsDFh5kvAEAY1afMFTWWbZT2d7TjCYtGBt47e1WpYoXYKSoEjxQX9qhNSw2ujMPiHn5rVyVCwQH1kBLzs95n37MoD6ikVDd29fkJoZaBrGhNaxcFiW9b1N85NWGQ9vs1LEMxsqt5ZHiFXGp1tMU85Da4VETb7veFZjvxRsadgbgeK1tFG83c6SSyDjoVzQbagXJ44kZXaMMmbs4Kk6VJszrCM1CUHPk7kFgHmojWU8kFAmPE88oesxBW5Wdc5cCNB6cB4QzRGuYrkK15hMdoFWq4kVHJn1xbruCzJ7Mb9f2rKUF6KTiUbTYpHuo4Kbnu26tJKQ9D7xHkAM2N3ZPN3eRkXBefKme5mLhGjRgXm6fAZiCHuK4dMyg4Bd9HDXiy8vSdY8cyZnyuJdPsjq5FRvRD92cFNtJZBJJwRQu6WiwKhTL9jELwwGfU2jukeESmARHjpQRTkXhtyG5NHDwj3Yx9CsbyBR5xdGsB3raA8JiMP4nAsbZhfXiBErBUx4MwYRnBDZERZztPjJWJniyKVG6hfoBokzEtkZt6gYMh1tpjsBAcSVw4C9H7o7QrY3mW6DjSufDdHSdWPVJjfHgRzxUM218CSiEwqEctqxJP9fc2FVSDxai7JUnroVzgYzhb62S4ueLGKM83abkd3Fm5NeSuewPRbgwETTLvknz1Wq1G4qygq75Fp3Kr21qknM2tsgrkwyprYR9ZTK5YzY5sHCwNP14VXZeX24QdSfevspNdvFtFtiDq6dUufmy5bKeLdHxx7Mpb7vFToU8bk9zZNUkcgXvX12U6iT1zLEyszTS6B3csHRr1HmvLUgEQKfd2aWjV2ScktEBsjZRHWdWuxQRcs85sF92kW2fVX7k1EGAwYGsnr6Wf9Q7jkM7SgJM5WJ1rHHsxKXwj8j11QmwXRcgREVEJphWdj87tpsC36A4rfEYhyJDw13UB68JgoK544NbsA"
    },
    {
      "id": "#6MkrCD1c",
      "type": "Ed25519VerificationKey2020",
      "publicKeyMultibase": "z6MkrCD1csqtgdj8sjrsu8jxcbeyP6m7LiK87NzhfWqio5yr",
      "controller": "did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P:z3U8mHxj7WY8i5EQD5rmkfPaDTws9dw5NRERbsFhsDFh5kvAEAY1afMFTWWbZT2d7TjCYtGBt47e1WpYoXYKSoEjxQX9qhNSw2ujMPiHn5rVyVCwQH1kBLzs95n37MoD6ikVDd29fkJoZaBrGhNaxcFiW9b1N85NWGQ9vs1LEMxsqt5ZHiFXGp1tMU85Da4VETb7veFZjvxRsadgbgeK1tFG83c6SSyDjoVzQbagXJ44kZXaMMmbs4Kk6VJszrCM1CUHPk7kFgHmojWU8kFAmPE88oesxBW5Wdc5cCNB6cB4QzRGuYrkK15hMdoFWq4kVHJn1xbruCzJ7Mb9f2rKUF6KTiUbTYpHuo4Kbnu26tJKQ9D7xHkAM2N3ZPN3eRkXBefKme5mLhGjRgXm6fAZiCHuK4dMyg4Bd9HDXiy8vSdY8cyZnyuJdPsjq5FRvRD92cFNtJZBJJwRQu6WiwKhTL9jELwwGfU2jukeESmARHjpQRTkXhtyG5NHDwj3Yx9CsbyBR5xdGsB3raA8JiMP4nAsbZhfXiBErBUx4MwYRnBDZERZztPjJWJniyKVG6hfoBokzEtkZt6gYMh1tpjsBAcSVw4C9H7o7QrY3mW6DjSufDdHSdWPVJjfHgRzxUM218CSiEwqEctqxJP9fc2FVSDxai7JUnroVzgYzhb62S4ueLGKM83abkd3Fm5NeSuewPRbgwETTLvknz1Wq1G4qygq75Fp3Kr21qknM2tsgrkwyprYR9ZTK5YzY5sHCwNP14VXZeX24QdSfevspNdvFtFtiDq6dUufmy5bKeLdHxx7Mpb7vFToU8bk9zZNUkcgXvX12U6iT1zLEyszTS6B3csHRr1HmvLUgEQKfd2aWjV2ScktEBsjZRHWdWuxQRcs85sF92kW2fVX7k1EGAwYGsnr6Wf9Q7jkM7SgJM5WJ1rHHsxKXwj8j11QmwXRcgREVEJphWdj87tpsC36A4rfEYhyJDw13UB68JgoK544NbsA"
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
      "serviceEndpoint": "didcomm://queue",
      "accept": [
        "didcomm/v2"
      ],
      "routingKeys": []
    }
  ],
  "alsoKnownAs": [
    "did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P"
  ],
  "id": "did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P:z3U8mHxj7WY8i5EQD5rmkfPaDTws9dw5NRERbsFhsDFh5kvAEAY1afMFTWWbZT2d7TjCYtGBt47e1WpYoXYKSoEjxQX9qhNSw2ujMPiHn5rVyVCwQH1kBLzs95n37MoD6ikVDd29fkJoZaBrGhNaxcFiW9b1N85NWGQ9vs1LEMxsqt5ZHiFXGp1tMU85Da4VETb7veFZjvxRsadgbgeK1tFG83c6SSyDjoVzQbagXJ44kZXaMMmbs4Kk6VJszrCM1CUHPk7kFgHmojWU8kFAmPE88oesxBW5Wdc5cCNB6cB4QzRGuYrkK15hMdoFWq4kVHJn1xbruCzJ7Mb9f2rKUF6KTiUbTYpHuo4Kbnu26tJKQ9D7xHkAM2N3ZPN3eRkXBefKme5mLhGjRgXm6fAZiCHuK4dMyg4Bd9HDXiy8vSdY8cyZnyuJdPsjq5FRvRD92cFNtJZBJJwRQu6WiwKhTL9jELwwGfU2jukeESmARHjpQRTkXhtyG5NHDwj3Yx9CsbyBR5xdGsB3raA8JiMP4nAsbZhfXiBErBUx4MwYRnBDZERZztPjJWJniyKVG6hfoBokzEtkZt6gYMh1tpjsBAcSVw4C9H7o7QrY3mW6DjSufDdHSdWPVJjfHgRzxUM218CSiEwqEctqxJP9fc2FVSDxai7JUnroVzgYzhb62S4ueLGKM83abkd3Fm5NeSuewPRbgwETTLvknz1Wq1G4qygq75Fp3Kr21qknM2tsgrkwyprYR9ZTK5YzY5sHCwNP14VXZeX24QdSfevspNdvFtFtiDq6dUufmy5bKeLdHxx7Mpb7vFToU8bk9zZNUkcgXvX12U6iT1zLEyszTS6B3csHRr1HmvLUgEQKfd2aWjV2ScktEBsjZRHWdWuxQRcs85sF92kW2fVX7k1EGAwYGsnr6Wf9Q7jkM7SgJM5WJ1rHHsxKXwj8j11QmwXRcgREVEJphWdj87tpsC36A4rfEYhyJDw13UB68JgoK544NbsA"
}
```

###### Short form
To resolve a short form `did:peer:4` DID, you must know the corresponding long form DID. It is not possible to resolve a short form `did:peer:4` without first seeing and storing it's long form counterpart.

To resolve a short form DID, take the decoded document (which will look exactly like the input doc example above) and follow the same rules described in the [long form](#long-form) section to "contextualize" the document but using the short form DID instead of the long form DID.

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
      "controller": "did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P"
    },
    {
      "id": "#6MkrCD1c",
      "type": "Ed25519VerificationKey2020",
      "publicKeyMultibase": "z6MkrCD1csqtgdj8sjrsu8jxcbeyP6m7LiK87NzhfWqio5yr",
      "controller": "did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P"
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
      "serviceEndpoint": "didcomm://queue",
      "accept": [
        "didcomm/v2"
      ],
      "routingKeys": []
    }
  ],
  "alsoKnownAs": [
    "did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P:z3U8mHxj7WY8i5EQD5rmkfPaDTws9dw5NRERbsFhsDFh5kvAEAY1afMFTWWbZT2d7TjCYtGBt47e1WpYoXYKSoEjxQX9qhNSw2ujMPiHn5rVyVCwQH1kBLzs95n37MoD6ikVDd29fkJoZaBrGhNaxcFiW9b1N85NWGQ9vs1LEMxsqt5ZHiFXGp1tMU85Da4VETb7veFZjvxRsadgbgeK1tFG83c6SSyDjoVzQbagXJ44kZXaMMmbs4Kk6VJszrCM1CUHPk7kFgHmojWU8kFAmPE88oesxBW5Wdc5cCNB6cB4QzRGuYrkK15hMdoFWq4kVHJn1xbruCzJ7Mb9f2rKUF6KTiUbTYpHuo4Kbnu26tJKQ9D7xHkAM2N3ZPN3eRkXBefKme5mLhGjRgXm6fAZiCHuK4dMyg4Bd9HDXiy8vSdY8cyZnyuJdPsjq5FRvRD92cFNtJZBJJwRQu6WiwKhTL9jELwwGfU2jukeESmARHjpQRTkXhtyG5NHDwj3Yx9CsbyBR5xdGsB3raA8JiMP4nAsbZhfXiBErBUx4MwYRnBDZERZztPjJWJniyKVG6hfoBokzEtkZt6gYMh1tpjsBAcSVw4C9H7o7QrY3mW6DjSufDdHSdWPVJjfHgRzxUM218CSiEwqEctqxJP9fc2FVSDxai7JUnroVzgYzhb62S4ueLGKM83abkd3Fm5NeSuewPRbgwETTLvknz1Wq1G4qygq75Fp3Kr21qknM2tsgrkwyprYR9ZTK5YzY5sHCwNP14VXZeX24QdSfevspNdvFtFtiDq6dUufmy5bKeLdHxx7Mpb7vFToU8bk9zZNUkcgXvX12U6iT1zLEyszTS6B3csHRr1HmvLUgEQKfd2aWjV2ScktEBsjZRHWdWuxQRcs85sF92kW2fVX7k1EGAwYGsnr6Wf9Q7jkM7SgJM5WJ1rHHsxKXwj8j11QmwXRcgREVEJphWdj87tpsC36A4rfEYhyJDw13UB68JgoK544NbsA"
  ],
  "id": "did:peer:4zQmNsz8npvrAyj983LTownQhp3PmGVGzMYrhBRGfig6rZ6P"
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
^did:peer:(([01](z)([1-9a-km-zA-HJ-NP-Z]{46,47}))|(2((\.[AEVID](z)([1-9a-km-zA-HJ-NP-Z]{46,47}))+(\.(S)[0-9a-zA-Z=]*)?)))$
```
:::

A match against this regex places `numalgo` (the algorithm for choosing a numeric basis) in capture group 1, `transform` in capture group 2, and `encnumbasis` in capture group 3.

### CRUD Operations

`peer` DIDs implement the following CRUD (Create, Read, Update, and Delete) operations:

*   Create: A `peer` DID is created by `numalgo` as defined in the [Generation Methods](#generation-methods) section of this specification. Once created, the DIDs are shared by the party creating the DID with anyone they want to have and use the DID.
*   Read: Those holding `peer` DIDs store them locally and "resolve" them by looking up their DIDDoc by the `peer` DID identifier string. In [DIDComm](https://identity.foundation/didcomm-messaging/spec/v2.0/), agents may index the DIDs by the keys within the DIDs and look up DIDs via their keys. As noted earlier, `peer` DIDs are not resolvable by anyone other than those that have received the DIDs.
*   Update: `peer` DIDs cannot be updated. Implementations such as [DIDComm](https://identity.foundation/didcomm-messaging/spec/v2.0/) use DID Rotation instead of updating the contents of the DIDDoc when necessasry.
*   Delete: `peer` DIDs are deleted by those holding the DIDs. When a party is no longer interested in participating in the capabilities enabled by the sharing of the DIDs, they can simply delete their local copy, without without notifying other parties using the `peer`.
