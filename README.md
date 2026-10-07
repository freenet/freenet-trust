# freenet-trust

Shared infrastructure for **signed claims about things on Freenet**: an issuer
says something about a subject (a piece of content, a key, a contract), and any
Freenet app can choose which issuers it honours.

The first use is helping app developers deal with illegal content. Later uses
could include spam and scam lists, seller ratings and reviews.

**Status: design discussion. There is no code yet.** This repository is a place
to work out the design together before implementation starts. Please use
[Discussions](https://github.com/freenet/freenet-trust/discussions) for ideas,
objections and questions.

## Why

Any system on Freenet that shows users content other people published needs a
way to identify and remove illegal material. Without one, it risks being
delisted from [Atlas](https://github.com/freenet/atlas), Freenet's discovery
index, and having its links banned from the Freenet Official River room. Images
and video are where this matters most.

Most Freenet apps are built by individuals or small teams, often pseudonymous,
who cannot run detection, human review and legal reporting themselves. A shared
system lets an app meet the requirement by honouring a list and adding a report
button, rather than building moderation infrastructure of its own. It also gives
a network effect: material removed in one app can be blocked in all of them.

The same shape (an issuer signs a claim about a subject) is general, so this is
designed as a general system with moderation as its first application, rather
than as a one-off moderation list.

## Design principles (draft)

1. **Claims, issuers, choice.** A claim is a signed statement by an issuer key
   about a subject, with a claim type and a way to revoke it. Apps decide which
   issuers to honour. A policy, such as Atlas's listing rules, may require
   honouring a particular issuer. The system itself stays neutral.
2. **Anyone can be an issuer.** Freenet's own list would be one issuer among
   many, so no single party becomes a mandatory censor.
3. **Clear and hidden subjects.** Some claims want their subjects public
   (a review). Others must not reveal them: a list of illegal content must
   never become a directory of it. Hashing the subject is **not** enough for
   hidden subjects. Freenet content can be crawled, so anyone could hash what
   they find and test it against a public list. Hidden-subject checks therefore
   need to be online and rate-limited, for example with an oblivious PRF
   (as in Google's Password Checkup), so an app can check an item it already
   holds but nobody can enumerate the list.
4. **Identify content by what it is, not where it is.** Content subjects are a
   hash of the media bytes, not a contract key. That works however an app
   stores media, matches the same file across apps, and survives contract
   re-keys.
5. **Start minimal.** v1 covers claims, issuers, revocation and the two subject
   modes. Trust scoring and aggregation, the open research problem in
   decentralized reputation, are deliberately left until a concrete consumer
   needs them, so they cannot delay the urgent part.
6. **Abuse resistance without a server.** Writing claims or making checks can
   be gated with [ghost keys](https://freenet.org/ghostkey/) or proof-of-work,
   rate-limited per key.

## Moderation as the first application (draft)

Moderation is a *service that publishes claims into* freenet-trust, not part of
freenet-trust itself. Its parts:

- **Detect.** Matching against known material first, which is cheap and
  precise. Classifiers for new material later, if at all. Perceptual hashes
  stay private to the detection service.
- **Decide.** Human review before acting on anything a classifier flags.
- **Publish.** Signed hidden-subject claims that apps check.
- **Enforce.** Each app filters what it shows, helped by a small SDK
  (`is_flagged`, blur-until-checked, a report button).

Running detection carries legal obligations for whoever operates it (in the US,
reporting to NCMEC and preserving reported material for a year). Who that
operator is, Freenet or a partner organisation that already does this work, is
an open question, and the design should not assume an answer.

Encrypted content, such as private River rooms, is out of reach by design and
relies on the moderators of those spaces.

## Open questions

- Is an online, rate-limited check acceptable for hidden subjects, and what
  should an app do when the check service is unavailable?
- Who operates moderation detection and review: Freenet, or a partner?
- How are issuers discovered, and how does an app express "honour these
  issuers" in a way users can inspect?
- How should revocation and appeals work, and what transparency (counts by
  claim type, for example) should an issuer publish?
- What is the narrowest useful scope for Freenet's own moderation issuer?
- Which second consumer (Atlas reviews, seller ratings, spam lists) should the
  clear-subject mode be designed against?

## License

[LGPL-3.0](LICENSE), like the rest of the Freenet ecosystem.
