## Introduction

### Overview

*This section is non-normative.*

Most documentation about decentralized identifiers (DIDs) describes them as identifiers that are rooted in a
public source of truth like a blockchain, a database, a distributed filesystem, or similar. This publicness lets
arbitrary parties [resolve] the DIDs to an
endpoint and keys. It is an important feature for many use cases.

[resolve]: https://w3c-ccg.github.io/did-resolution/

However, the vast majority of relationships between people, organizations, and
things have simpler requirements. When Alice `(`Corp`|`Device`)` and Bob want to
interact, there are exactly and only 2 parties in the world who should care:
Alice and Bob. Instead of arbitrary parties needing to resolve their DIDs, only
Alice and Bob do. Peer DIDs are perfect in these cases. In many ways, peer DIDs
are to public, blockchain-based DIDs what [Ethereum Plasma] or [state channels]
are to on-chain smart contracts&mdash; or what [Bitcoin's Lightning Network] is
to on-chain cryptopayments. They move the bulk of interactions off-chain, but
offer options to connect back to a chain-based ecosystem as needed. Peer DIDs
create the conditions for people, organizations and things to have full control
of their end of the digital relationships they sustain.

[Ethereum Plasma]: https://education.district0x.io/general-topics/understanding-ethereum/understanding-plasma/
[state channels]: https://education.district0x.io/general-topics/understanding-ethereum/basics-state-channels/
[Bitcoin's Lightning Network]: https://lightning.network/

Some formal terminology may make the application of this insight to DIDs clearer:

[[def: Anywise DID, anywise]]
~ A DID intended for use with an unknowable number of parties (e.g., the global
public or some subset thereof).

[[def: Pairwise DID, pairwise]]
~ A DID intended to be known by its subject and exactly one other party (e.g., one
usable in the Alice and Bob example just above).</dd>

[[def: N-wise DID, n-wise]]
~ A DID intended to be known by exactly *N* enumerated parties including its
subject. A business partnership with 3 members might be a modeled with n-wise
DIDs. Pairwise DIDs are just a special case of an N-wise DID (*N* = 2).

Generally, an [[ref: anywise]] DID needs to be resolvable by strangers (i.e.
publicly anchored DID). These strangers can use the DID to reference its subject
(usually by resolving it on a public ledger) without establishing a
relationship. On the other hand, [[ref: pairwise]] and [[ref: n-wise]] DIDs only
need to be resolvable by the parties in the relationship, and *each* party in
the relationship has to contribute a DID to make the relationship work. Because
of the reciprocal nature of DID usage in enumerated relationships like this, we
call the parties "peers", and it is this dynamic that gives our DID method its
name.

### Benefits

*This section is non-normative.*

Peer DIDs are not suitable for anywise use cases, which are usually public by
intent. However, peer DIDs have certain virtues that make them desirable for
private relationships between a small number of enumerable parties:

- They have no transaction costs, making them essentially free to create, store,
and maintain.
- They scale and perform entirely as a function of participants, not with any
central system's capacity.
- Because they are not persisted in any central system, there is no trove to
protect.
- Because only the parties to a given relationship know them, concerns about
personal data and privacy regulations due to third-party data controllers or
processors are much reduced.
- Because they are not beholden to any particular blockchain, they have minimal
political or technical baggage.
- Because they avoid a dependence on a central source of truth, peer DIDs free
themselves of the often-online requirement that typifies most other DID methods,
and are thus well suited to use cases that need a decentralized peer-oriented
architecture. Peer DIDs can be created and maintained for an entire lifecycle
without any reliance on the internet, with no degradation of trust. They thus
align closely with the ethos and the architectural mindset of the [local-first]
and [offline-first] software movements.

[local-first]: https://www.inkandswitch.com/local-first.html
[offline-first]: http://offlinefirst.org/

For more on when peer DIDs do and do not make sense, see [Comparison to Other
DID Methods](#comparison-to-other-did-methods) in the Appendices.
