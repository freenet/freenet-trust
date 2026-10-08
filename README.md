# freenet-trust

Shared infrastructure for **signed claims about things on Freenet**: an issuer
says something about a subject (a piece of content, a key, a contract), and any
Freenet app can choose which issuers it honours.

The first use is helping app developers deal with illegal content, through
**user reporting**. Later uses could include spam and scam lists, seller
ratings and reviews.

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
who cannot build moderation infrastructure from scratch. This project aims to
give them a small kit that meets the requirement: a report button, a moderator
queue, and optionally a way to share removal decisions with other apps.

## Scope decisions so far

- **Reporting first, not proactive AI scanning.** Scanning content creates
  legal obligations for whoever runs it (in the US, reporting to NCMEC and
  preserving reported material for a year), needs human review of the worst
  material, and costs money at scale. A reporting system avoids nearly all of
  that and fits how Freenet apps already work. The trade-off is that it is
  reactive: material stays up until someone reports it and a moderator acts.
  Proactive detection may come later as an opt-in service, but it is out of
  scope for v1.
- **Moderation stays in the app layer.** Nothing here requires changes to
  freenet-core. An app that ignores or buries reports is handled the way it is
  today: users report the app itself to Atlas and the Official room, and those
  can delist it.
- **A general system, minimal at first.** The shape of a report outcome (an
  issuer signs a claim about a subject) is general, so this is designed as a
  general claims system with moderation as its first application. Trust
  scoring and aggregation are left until a concrete consumer needs them.

## How reporting works (draft)

1. **Any user can report an item.** The report goes to the moderator of that
   app or space, for example a River room's owner. For suspected child sexual
   abuse material, the report UI also links to
   [NCMEC's CyberTipline](https://report.cybertip.org), so the material never
   has to pass through anyone else.
2. **Reports are gated against abuse** with a
   [ghost key](https://freenet.org/ghostkey/) signature, or proof-of-work for
   users without one, since reporting illegal material should never cost
   money. Contracts cannot read the clock, so rate limits are clock-free, for
   example a cap on open reports per key.
3. **A report never removes anything by itself.** A moderator decides. That
   turns a flood of false reports into a nuisance in the queue rather than a
   way to take content down.
4. **Reports are encrypted to the moderator.** Contract state on Freenet is
   public, so a plain-text report inbox would be a list of reported (possibly
   illegal) items and of who reported them. The reporter signs the ciphertext,
   so the inbox contract can still check the signature without reading the
   report.
5. **A report is a pointer, not a trusted hash.** It carries a locator
   (contract key plus path or item id) and a content hash. The moderator
   fetches the bytes, recomputes the hash and judges what they actually see.
   An app's UI computes whatever hash it sends, so this is what stops an app,
   or anyone else, from getting the wrong content flagged by misdirecting
   reports.
6. **The moderator removes the item** using the app's own mechanism, such as a
   tombstone in the contract or a denylist the UI filters against.

## Sharing removal decisions across apps (draft, possibly not v1)

When a moderator removes something as illegal, they can publish that decision
as a claim, so other apps that honour that moderator (or a curated issuer) can
block re-uploads of the same file. This spreads human decisions, not automated
ones.

- **Content is identified by a hash of its bytes**, not by where it is stored.
  That works however an app stores media, matches the same file across apps,
  and survives contract re-keys.
- **Claims are only made about bytes a moderator has checked**, as above.
- **The list must never become a directory of illegal material.** Hashing the
  subject is not enough: Freenet content can be crawled, so anyone could hash
  what they find and test it against a public list. Checks therefore need to be
  online and rate-limited, for example with an oblivious PRF (as in Google's
  Password Checkup): the checker learns whether an item it already holds is
  listed, the service learns neither the item nor the result, and nobody can
  enumerate the list.
- **Checks happen where the moderator is, not where every reader is.** For
  River, the room owner's node (for example a moderation delegate, using
  scheduled wake-ups) checks new and recent media in the room. Check volume
  then follows uploads, not views, and the rate limit attaches to the
  moderator's key.
- **Rate limits attach to whoever makes the check, not to the app's creator.**
  An app creator's key cannot be shipped in the app, and a remote service
  cannot verify that a request really came from a given app, so a per-app
  quota could be used up by anyone claiming to be that app.
- **Private spaces are partly covered.** In an encrypted space, such as a
  private River room, the owner's node holds the decrypted bytes and can run
  exact-hash checks without revealing anything to the service.

## Cost

Checking is cheap: an oblivious PRF evaluation is about one elliptic-curve
multiplication, so a million checks costs about a minute of CPU. The main
running cost is the network traffic of carrying checks through contracts,
which is why checks are batched and made per upload rather than per view.
Reporting itself costs almost nothing.

## Explored and deferred

- **Proactive AI detection** (hash matching against vetted lists of known
  material, open-weight classifiers): technically feasible and cheap in
  compute for images, but it brings the legal and human-review burden described
  above. Revisit as an opt-in service with an accountable operator.
- **Perceptual hashes** (PDQ for images, TMK+PDQF or vPDQ for video) would
  catch modified re-uploads. They cannot use an oblivious exact-match check,
  they can leak image content (PhotoDNA has been shown to be invertible to
  thumbnails), and they can be deliberately collided. So a perceptual list must
  never be published, matching would need a trusted service, and a match could
  only ever trigger human review.

## Open questions

- Should v1 include cross-app sharing of removal decisions, or start with
  per-app reporting only?
- A reporter's ghost-key fingerprint is visible on the report ciphertext and is
  the same across apps, so reporting activity is linkable. Is that acceptable,
  or do we need single-use report tokens that hide who reported?
- What should an app do when the check service is unavailable?
- How are issuers discovered, and how does an app express "honour these
  issuers" in a way users can inspect?
- How should revocation and appeals work, and what transparency (counts by
  claim type, for example) should an issuer publish?
- Which second consumer (Atlas reviews, seller ratings, spam lists) should the
  general claim format be designed against?

## License

[LGPL-3.0](LICENSE), like the rest of the Freenet ecosystem.
