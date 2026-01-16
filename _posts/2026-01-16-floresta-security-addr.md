# Security Concerns Regarding Floresta’s Address Relay and AddrMan Implementation

Recently, I reported to @Davidson-Souza some security concerns about Floresta's address relay and addrman (address manager), especially some non-conformities with BIP155. Since Floresta is running only in Signet, he told me to share these thoughts for learning purposes. As soon as Floresta gets ready for "production" use, they will adopt a responsible disclosure policy.

## Violation of the 1,000-Address Limit in addrv2

First of all, I noticed that Floresta, when sending addresses to other peers, was not respecting the rule of sending at most 1'000 addresses per message. This number is specified in BIP 155, where the addrv2 message is defined. The BIP says:

```sh
One message can contain up to 1,000 addresses. Client SHOULD reject messages with more addresses.
```

Violating this limit could cause Floresta nodes to be disconnected and even banned from other full-node implementations such as Bitcoin Core. Also, at the same time, Floresta was not respecting this number when sending addresses, and it was also not verifying it when receiving addresses, resulting in the acceptance of a large number of addresses per message.

## Storing Addresses from Unsupported or Unreachable Networks

Another issue identified in Floresta’s handling of incoming addrv2 messages is the storage and propagation of addresses belonging to networks that Floresta neither supports nor can reach. As a result, Floresta may store and relay addresses that it has no ability to validate.

This behavior directly contradicts the guidance provided in BIP155, which states:

```
Clients SHOULD NOT gossip addresses from unknown networks because they have no means to validate those addresses and so can be tricked to gossip invalid addresses.
```

Relaying such addresses increases the risk of polluting the address gossip network with invalid or malicious entries.

## Acceptance of Private Addresses

Finally, Floresta relies on Rust’s standard library to determine whether an IP address is private. However, the standard library does not classify the 0.255.255.255 address range as private. As a consequence, Floresta currently accepts and processes addresses within this range, despite them being non-routable and invalid for peer-to-peer communication.

-------------------------------------------------------------------------------------

- These concerns allowed me to be able to crash a (limited-resource) florestad by spamming large addrv2 messages. Also, I could fill and bloat the address manager.
- Davidson is promptly addressing these concerns [floresta@781](https://github.com/getfloresta/Floresta/pull/781)
- I also suggested them to adopt a rate-limit for this message, like Bitcoin Core.
