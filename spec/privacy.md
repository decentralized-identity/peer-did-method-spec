### Privacy Considerations

*This section is non-normative.*

Peer DIDs should remain pairwise or n-wise, not be reused across relationships. This allows proper isolation to defeat correlation. It also enables granular exercise of sovereignty in a robust, [multidimensional identity](https://medium.com/evernym/three-dimensions-of-identity-bc06ae4aec1c).

![many pairwise DIDs](https://raw.githubusercontent.com/decentralized-identity/peer-did-method-spec/master/pairwise.png)

Alice maintains different pairwise DIDs for each relationship, and discloses different aspects and quantities about herself in each one.

#### Fingerprinting

The names of roles and the arrangement of rules in a peer DID doc could conceivably be used to create a sort of fingerprint of a sovereign domain, in much the same way that browser fingerprinting keys off individual uniqueness in combinations of browser+plugin+hardware configuration. To combat this problem, the following best practices are recommended:

*   Choose rules from a standard inventory.
*   Choose role names from a standard inventory.
*   Only define keys that are relevant to a particular relationship.
*   Never reuse `id`s for keys, rules, or service endpoints across DID docs.