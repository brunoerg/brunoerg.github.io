# The state of Bitcoinfuzz

bitcoinfuzz is a project that does differential fuzzing of Bitcoin protocol implementations and libraries. I created the project as an experimental one, and the first version of the project was really experimental and had a bad design. That said, since the project was getting more attention, one of the things that I worked on was refactoring the whole project to be like cryptofuzz, modularized. It means that you can choose which projects you want to fuzz, and you can build the projects (we call modules) individually.

The first targets that I wrote were descriptor and miniscript parsers, in which we found many bugs.. The first one we reported was the sipa's miniscript implementation incorrectly considering `pk()()` as a valid policy because it identifies `)(` as the name. Currently, we have many other targets like: script evaluation, descriptor parse, miniscript parse, addrv2, psbt, address parse, and others, we intend to work on more ones.

So far, we discovered and reported over 30 bugs in projects such as btcd, rust-bitcoin, rust-miniscript, Embit, Bitcoin Core, Core Lightning, LND, etc. Such as: many implementations using the incorrect type for the key's type value for PSBTs, the CVE-2024-44073 of rust-miniscript, panic on btcd's PSBT parser (https://github.com/btcsuite/btcd/issues/2351), Embit incorrectly pointing a miniscript as valid (https://github.com/bitcoinfuzz/bitcoinfuzz/issues/113), Core Lightning accepting invoices with non-standard witness address fallbacks that other implementations correctly reject (https://github.com/ElementsProject/lightning/pull/8219), and other ones.

## What are the current projects we support?

We currently integrate and support the following projects:

- Bitcoin Core - C++
- rust-bitcoin - Rust
- rust-miniscript - Rust
- embit - Python
- btcd - Golang
- LDK - Rust
- LND - Golang
- Core Lightning - C
- NLightning - C#
- NBitcoin - C#
- Eclair - Scala
- lightning-kmp - Kotlin

There is a PR (under review) that integrates libbitcoin. We welcome any PR that integrates more projects into bitcoinfuzz. However, we intend to review the support of some implementations since an immature implementation can hinder the differential fuzzing with simple issues. We have noticed it for Embit which its miniscript/descriptor implementation misses a lot of things - many fragments are not supported, for example. Also, we ain't to have support for Mako but we removed it since there is no active development around it.

## Lightning Network on Bitcoinfuzz

Differential fuzzing of Lightning implementations would be very valuable. That said, Erick Cestari and Morehouse joined it in order to advance it. Erick is working on it as a fellow from Vinteum and he has been mentored by Bruno and Morehouse. So far, bitcoinfuzz has targets for BOLT11 invoice decoding and BOLT12 offer decoding. There are open PRs adding lightning-kmp module for invoice deserialization, BOLT12 invoice request decoding, adding Eclair module for invoice deserialization and others.


One of the advantages of doing differential fuzzing of Lightning implementations is that the Lightning Network has a well maintained specification. It means that when we find any discrepancy, we have a place to check what should be the correct behavior. It doesn't happen for Bitcoin protocol implementations which we mostly rely on the Bitcoin Core. However, we have seen that differential fuzzing is also a good tool to improve the spec itself. From our experience, the specification is not always clear and might have a window for improvements. As an example, one of the bitcoinfuzz trophies is a fix on the bolt12 specification, where a currency UTF-8 test vector lacked the required 3-byte length. As a result, many implementations were treating it as a case of malformed currency length instead of properly validating the UTF-8 encoding.

Also, bitcoinfuzz found bugs in virtually every implementation we support. E.g.

- Core Lightning was accepting invoices with non-standard witness address fallbacks that other implementations correctly reject;
- When deserializing an invoice with a large expiry value, LND produces a negative expiry value due to overflow;
- rust-lightning was not verifying whether the offer_currency field contains valid UTF-8 for invoices.
- Eclair was accepting bolt11 invoices with empty routing hints in the r field.
- lightning-kmp was failing to verify the signature after public key recovery due to s-value malleability

## Doing differential fuzzing of projects that do not have fuzz testing

Some projects do not have support for fuzzing or do not run their fuzz targets continuously. It means that we could find bugs not because of the
"differential" thing, but simply because the project has not been fuzzed. As an example, we found and reported a panic on btcd when parsing a PSBT due to a bug on the `ReadTaprootBip32Derivation` functions. This kind of bug would be easily caught by a simple fuzz target and a few minutes of running. A similar case happened with rust-miniscript and lightning-kmp.

## Nuances can also get in the way

If we find a discrepancy between two implementations, does that mean there's a bug in one of them? Not always. Small nuances also hinder the process, but these have been curious cases. As an example, Bitcoin Core accepted miniscripts and descriptors with integer values containing trailing zeros or a positive sign (+) - e.g. `older(+1)` or `older(000001)`, we believe that no one uses it in practice, however, since rust-miniscript rejects these cases, it would cause a discrepancy between them. That said, Bitcoin Core addressed it in https://github.com/bitcoin/bitcoin/pull/30577 by using ToIntegral instead of ParseInt64.

## OSS-Fuzz

We recently opened a PR on OSS-Fuzz to integrate bitcoinfuzz into it. OSS-Fuzz is Google's continuous fuzzing infrastructure for open‑source software. We believe that it would benefit our project a lot since we have a current limited fuzzing infrastructure.

## Corpora

We created a repository where we share some corpora for our targets, similar to Bitcoin Core's qa-assets. However, we might keep two folders per target. One folder with all the inputs (maximizing coverage) and another one with all the inputs except those we know could crash - e.g. because we're waiting for one of the implementations to fix a known bug. The goal of the folder without the problematic inputs is to make it possible to run bitcoinfuzz on CI/CD.

## Future work

We have many things in mind to work on. In the short term, we intend to expand bitcoinfuzz to be “agnostic” and work with more fuzzers than libfuzzer;  we intend to add more targets - e.g. BOLT8, BOLT7, BOLT4 and ElligatorSwift encoding/decoding and we have a strong need to improve our build system, especially fixing the link issues.



