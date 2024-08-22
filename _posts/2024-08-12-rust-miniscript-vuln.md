## `rust-miniscript` vulnerability disclosure (DoS/Stack Overflow) - CVE-2024-44073

On 2nd July, we (Kartik Agarwala ([hax0kartik](https://github.com/hax0kartik)) and I) reported a stack overflow issue on rust-miniscript to Andrew Poelstra. We discovered that rust-miniscript could stack overflow when parsing a miniscript from a string due to a recursion. From our conversations, the reason was that the constant `MAX_RECURSION_DEPTH` was not being applied to prefix combinators like `n:`. It means the parser would take these "large" miniscripts and (iteratively) construct a tree with a very high depth. Then, this will stack overflow when doing any recursive operation. All the miniscript websites that help visualize/parse miniscript using `rust-miniscript` could crash with this input.

Affects: `rust-miniscript` 9, 10, 11 and 12.

---

### How did we find it?

**TL;DR: Fuzzing**

[Bitcoinfuzz](https://github.com/brunoerg/bitcoinfuzz) is a project that applies differential fuzzing in Bitcoin implementations and libraries. One of the targets does differential fuzzing with `rust-miniscript` and `Bitcoin Core` for parsing a miniscript from a string. So far, it found some bugs and has been helpful.

However, `bitcoinfuzz` had only support for `libfuzzer` and, at some point, Kartik suggested using some structures from `Bitcoin Core` to add support for more fuzzers in `bitcoinfuzz`. So, we started using `bitcoinfuzz` with AFL and, even fuzzing for a long time with libfuzzer, this bug was found in a few minutes with AFL. Just luck? We don't know haha.

#### Why the issue was not discovered before?

Both `rust-bitcoin` and `rust-miniscript` support fuzzing. Also, `rust-miniscript` has two targets called `roundtrip_miniscript_str` and `roundtrip_miniscript_script` which should be able to find this. However, the effectiveness of fuzzing depends on large campaigns, that is why it is important that both projects are continually fuzzed.

### Timeline

**07/02/2024** - Reported the issue to Andrew Poelstra via e-mail. \
**07/02/2024** - Andrew confirmed the issue and cc'ed Sanket. \
**07/08/2024** - Sanket reproduced the issue and opened a PR addressing it. \
**07/08/2024** - I tested and confirmed the fix. \
**07/18/2024** - Andrew discovered that the issue could also affect parsing Miniscripts from Script. \
**07/18/2024** - Sanket started working on a fix for it. \
**07/20/2024** - Sanket opened a PR addressing it. \
**08/06/2024** - I verified that all fixes were backported. Andrew and Sanket confirmed this, and we proceeded to have a CVE.

### Responsible Disclosure

We have found **MANY** bugs with bitcoinfuzz, and we have carefully analyzed all of them to know what is just a simple bug and what could be something more critical. Critical bugs/vulnerabilities have been reported to the according project team.

### Acknowledgements

I would like to first thank Kartik Agarwala for working with me on this project. Kartik is a Summer of Bitcoin intern and he has done a brillant work. Also, thanks Andrew and Sanket for the cooperation on it and other cases.
