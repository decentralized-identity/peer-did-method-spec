# Peer DID Method Specification

**Specification Status**: v1.0 Draft

**Latest Draft:**

[https://github.com/decentralized-identity/peer-did-method-spec](https://github.com/decentralized-identity/peer-did-method-spec)

**Editors:**

- [Daniel Hardman](https://www.linkedin.com/in/danielhardman/), [Provenant, Inc](https://provenant.net), W3CID: 108790, [daniel.hardman@gmail.com](mailto:daniel.hardman@gmail.com)
- [Stephen Curran](https://github.com/swcurran) - [Cloud Compass Computing Inc.](https://cloudcompass.ca)
- [Sam Curren](https://github.com/TelegramSam) - [Indicio / Decentralized Identity Foundation](https://sovrin.org)

**Authors**

- [Oskar Deventer](https://github.com/Oskar-van-Deventer) - [TNO](https://tno.nl)
- [Christian Lundkvist](https://github.com/christianlundkvist) - [ConsenSys](https://consensys.net)
- [Márton Csernai](https://github.com/csmarc) - [KILT](https://kilt.io)
- [Kyle Den Hartog](https://github.com/kdenhartog) - [Mattr](https://www.mattr.global)
- [Markus Sabadello](https://www.linkedin.com/in/markus-sabadello-353a0821) - [Danube Tech](https://danubetech.com/)
- [Dan Gisolfi](https://www.linkedin.com/in/vinomaster/) - [IBM](https://www.ibm.com)
- [Mike Varley](https://github.com/mavarley) - [SecureKey](https://securekey.com)
- [Sven Hammann](https://github.com/SvenHammann90) - [ETH Zurich](https://infsec.ethz.ch/)
- [John Jordan](https://github.com/jljordan42) - [Province of British Columbia](https://www.gov.bc.ca/)
- [Lovesh Harchandani](https://github.com/SvenHammann90) - [Evernym](https://evernym.com/)
- [Devin Fisher](https://github.com/devin-fisher) - [Evernym](https://evernym.com)
- [Tobias Looker](https://github.com/tplooker) - [Mattr](https://www.mattr.global)
- [Brent Zundel](https://github.com/brentzundel) - [Evernym](https://evernym.com)
- [Stephen Curran](https://github.com/swcurran) - [Cloud Compass Computing Inc.](https://cloudcompass.ca)
- [Denis Rybas](https://github.com/DenisRybas) - [DSR](https://en.dsr-corporation.com/)
- [Alexander Shcherbakov](https://github.com/ashcherbakov) - [DSR](https://en.dsr-corporation.com/)

## Abstract

This document defines a "peer" [DID Method] that conforms to the [DID Spec]. The
method can be used independent of any central source of truth, and is intended
to be cheap, fast, scalable, and secure. It is suitable for most private
relationships between people, organizations, and things. We expect that
peer-to-peer relationships in every blockchain ecosystem can benefit by
offloading pairwise and n-wise relationships to peer DIDs.
        
There are reasonable, tested did:peer libraries for [python] and [java]. It is
convenient to use with [DIDComm v2].

[DID Method]: https://w3c-ccg.github.io/did-spec/#specific-did-method-schemes
[DID Spec]: https://w3c-ccg.github.io/did-spec/
[python]: https://github.com/sicpa-dlab/peer-did-python
[java]: https://github.com/sicpa-dlab/peer-did-jvm
[DIDComm v2]: https://https://identity.foundation/didcomm-messaging/spec/
