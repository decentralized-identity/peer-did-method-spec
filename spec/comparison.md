### Comparison to Other DID Methods

*This section is non-normative.*

#### Ledger-oriented Methods

We could certainly build peer relationships with anywise DIDs based on a public ledger or a similar source of truth. However, in the same way that a company doesn't want the public to resolve private host names inside its corporate intranet, letting others resolve pairwise and n-wise DIDs is unnecessary, and it represents a privacy and security risk as well as a problem of cost, scale, and performance. We strongly recommend that peer DIDs be used for peer relationships.

In a similar vein, peer DIDs could be used, hypothetically, for anywise scenarios. The main disadvantage would be the lack of a formal publication mechanism. Nothing would prevent a user from publishing a peer DID and its associated DID document on a website. However, information published in this way would be hard to discover, maintain as DID docs evolved, and integrate into interoperable applications. DID methods that use a public ledger or a similar source of truth are a better choice here, because they have authoritative answers to the publication problem.

#### `did:key`

The [did:key](https://github.com/digitalbazaar/did-method-key-js) method encodes a public key directly as a DID value, and generates a very simple DID Doc in a deterministic way from that key. The only material that has to be generated or stored when using this method is the public key or DID itself; either can derive the other and the DID Doc. Using a `did:key` is very similar to sharing and then using a public SSH key.

The benefit of the `did:key` method is its simplicity. Like peer DIDs, it has no dependence on an external source of truth, and can be implemented in code with little effort. Like peer DIDs, they are cheap to create and use.

However, `did:key` are not direct equivalent of peer DIDs. Here are some features of peer DIDs that they do not provide:

*   Include multiple keys in the DIDDoc, such as separate verification and key agreement keys.
*   Define servicesin the DIDDoc, such as a DIDComm service endpoint for use in [DID Communication](https://github.com/hyperledger/aries-rfcs/tree/master/concepts/0005-didcomm).
*   Use multiple agents with the DID--each of which has its own keys.

Peer DIDs use a layering approach so the complexity of key rotation and updates need not be supported if only static, ephemeral use cases matter. This is the main complexity difference between peer DIDs and these other two methods, and is optional with peer DIDs. Peer DIDs also use multicodec to encode keys in the same way that `did:key` does. The hope is that these two methods will remain as similar as possible, and that Peer DIDs will be a logical upgrade choice when use cases go beyond those that `did:key` and `did:nacl` are designed to handle.
