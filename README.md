# freenet-trust

Shared infrastructure for **signed claims about things on Freenet**: an issuer
says something about a subject (a piece of content, a key, a contract), and any
Freenet app can choose which issuers it honours.

The first use is helping app developers deal with illegal content, through
**user reporting checked automatically within seconds**. Later uses could
include spam and scam lists, seller ratings and reviews.

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
give them a small kit that meets the requirement: a report button, a checking
service that confirms or rejects a report within seconds, optional scanning of
new uploads, and removal of confirmed material from the network, not just
from the app's screen.

## Scope decisions so far

- **Reports are free; scanning needs a ghost key.** Ghost keys gate services
  that consume resources, and scanning is one. Anyone may report an item,
  with a ghost key or proof-of-work, because reporting illegal material
  should never cost money. Scanning new uploads before anyone reports them
  costs the service a check per upload, so a scan request must be signed by a
  [ghost key](https://freenet.org/ghostkey/), and the service limits scans
  per key.
- **The service acts automatically, and only on known material.** It matches
  files against vetted lists of material already identified by organisations
  such as NCMEC and IWF. No person and no AI model views the material: we
  will not ask anyone to do that. Classifiers that judge material nobody has
  seen before are left out, because their vendors require human review before
  acting on a result (see "Explored and deferred"). New material is handled
  by sending reporters to NCMEC, whose trained analysts review it, and by
  re-checking reported items as the lists grow.
- **Confirmed material is removed at the source.** The service sends a signed
  tombstone to the contract that holds the item, so every peer drops the bytes
  within seconds. Nothing public names where confirmed material is.
- **The checking service has an accountable operator.** Running it carries
  legal duties: in the US, reporting apparent child sexual abuse material to
  NCMEC and preserving it for a year. Freenet Project Inc. would operate the
  service and register with NCMEC's CyberTipline. Both duties are automated.
- **Apps opt in, and nothing here changes freenet-core.** An app opts in by
  having its content contracts accept the service's tombstones. An app that
  ignores or buries reports is handled the way it is today: users report the
  app itself to Atlas and the Official room, and those can delist it.
- **A general system, minimal at first.** The shape of a verdict (an issuer
  signs a claim about a subject) is general, so this is designed as a general
  claims system with moderation as its first application. Trust scoring and
  aggregation are left until a concrete consumer needs them.

## How reporting works (draft)

1. **Any user can report an item.** The report goes to the checking service,
   or to the app's own moderator if the app prefers one, for example a River
   room's owner. For suspected child sexual abuse material, the report UI also
   links to [NCMEC's CyberTipline](https://report.cybertip.org), so material
   that is not yet on any list reaches people trained to review it.
2. **Reports are gated against abuse** with a
   [ghost key](https://freenet.org/ghostkey/) signature, or proof-of-work for
   users without one, since reporting illegal material should never cost
   money. Contracts cannot read the clock, so rate limits are clock-free, for
   example a cap on open reports per key.
3. **A report never removes anything by itself.** The check decides. A flood
   of false reports produces a flood of rejections, not takedowns.
4. **Reports are encrypted to the recipient.** Contract state on Freenet is
   public, so a plain-text report inbox would be a list of reported (possibly
   illegal) items and of who reported them. The reporter signs the ciphertext,
   so the inbox contract can still check the signature without reading the
   report.
5. **A report is a pointer, not a trusted hash.** It carries a locator
   (contract key plus path or item id) and a content hash. The checker fetches
   the bytes, recomputes the hash and judges what is actually there. An app's
   UI computes whatever hash it sends, so this is what stops an app, or anyone
   else, from getting the wrong content flagged by misdirecting reports.
6. **Confirmed material vanishes.** The service sends a tombstone to the
   contract named in the locator, and the item's bytes drop out of that
   contract on every peer that holds it.

## The checking service (draft)

### What it checks and what happens

| Result | Removed | Reported to NCMEC |
|---|---|---|
| Matches a list of known material | At once, by tombstone | Automatically |
| No match | No | No; the reporter is pointed to the CyberTipline |

Known-material lists and matching tools (for example PhotoDNA with NCMEC and
IWF hash lists) are available only to vetted organisations, so Freenet Project
Inc. would apply for access. Perceptual matching (PhotoDNA, PDQ) runs inside
the service and is never published (see "Explored and deferred").

An unmatched report is not forgotten. The service keeps the item's locator and
perceptual hashes, never its bytes, and re-checks them as the lists grow. When
NCMEC's analysts confirm a file someone reported to them directly, it joins the
lists, and the next re-check removes it here. That path takes days rather than
seconds, but it handles new material without anyone at Freenet viewing it.

Perceptual hashes can be deliberately collided, so a harmless image crafted to
match a listed one could be removed and reported under its publisher's key.
To guard against this, a match counts only when two independent algorithms
agree (for example PhotoDNA and PDQ), or when the file matches exactly.

Providers are protected from liability for their reports (18 U.S.C. §2258B)
unless they act with intentional misconduct, actual malice, or reckless
disregard of a risk of physical injury. Whether the checking service counts as
a provider at all is for counsel to settle before launch.

A scanned upload gets the same check and the same outcomes as a reported one.

### Proactive scanning

An app can ask the service to scan new uploads instead of waiting for a
report. The scan request is signed by the submitter's ghost key, and the
service limits how many scans each key may request, weighting the limit by
the key's donation amount and age. The service runs off-network and can read
a clock, so this limit can be an ordinary rate limit, unlike the clock-free
caps contracts need.

The submitter is whoever moderates the space, not the uploader: a bad
uploader would simply skip a scan of their own upload. For River, the room
owner's node submits new media in the room. Scan volume then follows uploads,
and the limit attaches to the moderator's key.

In a private, encrypted space, scanning means sending decrypted bytes to the
service. That is the owner's choice to make, and members should be told it is
on. An owner who does not want that can still use exact-hash checks (see
"Sharing removal decisions across apps"), which reveal nothing to the service.

### Removal

An app opts in by writing its content contracts to accept a tombstone signed
by the checking service. The contract carries the service's root public key,
and each tombstone carries a certificate from that root for the subkey that
signed it, so the service can rotate signing keys without any app changing its
contract. A shared crate will provide the tombstone type, its verification and
its merge rule, so apps do not hand-roll them.

A tombstone keeps the item's id and content hash and drops its bytes. It wins
every merge, so a peer holding older state cannot bring the item back, and it
must never be evicted by a cap on the contract's size. Because it is an
ordinary contract update, every subscribed peer applies it within seconds,
including the checking service's own node, which fetched the item to check it.

Two limits remain. A peer that holds an old copy and never syncs again keeps
the bytes. And between fetching an item and tombstoning it, the service's
node holds the item for a few seconds. Users should be told the item was
removed, not that it was erased from every disk.

The service does not tombstone apps that have not opted in, and publishes
nothing about them, since any public verdict would point at material it cannot
remove.

### Reporting to NCMEC

Reports go through the CyberTipline's API for registered providers, with no
manual step. The law requires reporting an apparent violation; the
identifying details it lists, such as IP addresses, are optional. A report
from this service would carry:

- the file and its hash
- the contract key and item id
- the publisher's public key, if the content is signed
- when the service saw it

It would not carry IP addresses, which the service does not have and must not
collect, or anything about the reporter. The publisher key is a persistent
pseudonym, so it links every report against the same uploader.

Most such reports will not lead to an investigation, because there is no
person to pass them to. We should explain the shape of these reports to NCMEC
when registering.

Reported material is preserved encrypted for the required year and then
deleted, automatically, unless a legal hold applies.

### Speed

Sending a tombstone to subscribers is fast. The slow step is the service
fetching the reported bytes, since a network GET can currently take far longer
than a few seconds. Reporters must not attach the bytes themselves: that would
have users sending the material to a private party, which the law permits
only towards law enforcement.

## Sharing removal decisions across apps (draft, possibly not v1)

When the checking service or a moderator removes something as illegal, the
decision can be published as a claim, so other apps that honour that issuer
can block re-uploads of the same file.

- **Content is identified by a hash of its bytes**, not by where it is stored.
  That works however an app stores media, matches the same file across apps,
  and survives contract re-keys.
- **Claims are only made about bytes the issuer has checked**, as above.
- **The list must never become a directory of illegal material.** Hashing the
  subject is not enough: Freenet content can be crawled, so anyone could hash
  what they find and test it against a public list. Checks therefore need to be
  online and rate-limited, for example with an oblivious PRF (as in Google's
  Password Checkup): the checker learns whether an item it already holds is
  listed, the service learns neither the item nor the result, and nobody can
  enumerate the list.
- **Checks happen where the moderator is, not where every reader is.** For
  River, the room owner's node checks new and recent media in the room. Check
  volume then follows uploads, not views, and the rate limit attaches to the
  moderator's key.
- **Rate limits attach to whoever makes the check, not to the app's creator.**
  An app creator's key cannot be shipped in the app, and a remote service
  cannot verify that a request really came from a given app, so a per-app
  quota could be used up by anyone claiming to be that app.
- **Private spaces are partly covered.** In an encrypted space, such as a
  private River room, the owner's node holds the decrypted bytes and can run
  exact-hash checks without revealing anything to the service.

## Cost

Checking an image is cheap: a hash computation and a lookup. Fetching the
item, especially video, costs more than the check. Report volume is bounded
by the ghost-key or proof-of-work gate. Scan volume follows uploads and is the
larger cost, which is why each scan needs a ghost key and each key has a
limit. Fees for vetted detection services are not yet known. Cross-app checks
are cheap too: an oblivious PRF evaluation is about one elliptic-curve
multiplication, so a million checks costs about a minute of CPU. The main
cost there is the network traffic of carrying checks through contracts, which
is why they are batched and made per upload rather than per view.

## Explored and deferred

- **Classifiers for new material** (Google's Content Safety API, Thorn's
  Safer): the terms we found require the partner to review content before
  acting on a result, or bar sharing results, and we will not ask people to
  review this material. An AI model as the reviewer was considered. A hosted
  model would mean sending the material to a third party, and a self-hosted
  one is in effect another classifier, which is unlikely to satisfy a review
  requirement. Worth asking the vendors before ruling it out.
- **A verdict list read through related contracts**, with content contracts
  dropping items listed in it: freenet-core resolves related contracts only
  when a contract is validated, and never re-validates when the related
  contract changes. A peer that merged an item before the verdict would keep it
  and keep sending it, so peers would never converge. The list would also be a
  public directory of hashes of confirmed material. A tombstone sent to the
  content contract has neither problem.
- **Perceptual hashes as a published list** (PDQ for images, TMK+PDQF or vPDQ
  for video): they cannot use an oblivious exact-match check, they can leak
  image content (PhotoDNA has been shown to be invertible to thumbnails), and
  they can be deliberately collided. So a perceptual list must never be
  published, and matching happens only inside the checking service.

## Open questions

- What do PhotoDNA's terms allow? They bar using the cloud service to provide
  a managed service, which checking for many apps may be. We need the
  vendors' answers in writing.
- Does the checking service count as a provider under the law, and what
  follows if it does not? (A question for counsel.)
- Report-triggered checks cost the same as scans, so proof-of-work reports are
  a free scan and let an offender test altered files until one passes. How
  small should the proof-of-work allowance be, relative to ghost-key reports?
- With nobody reviewing, how does someone appeal a wrongly removed item?
- How do reports work in encrypted spaces, where the service cannot read the
  item?
- How is the service's root key protected against theft or compelled misuse,
  and what transparency log of removals (hashes and reasons only) should it
  publish?
- How many scans should a ghost key get, and how should donation amount and
  key age weight that?
- Should v1 include cross-app sharing of removal decisions, or start with
  per-app reporting only?
- A reporter's ghost-key fingerprint is visible on the report ciphertext and is
  the same across apps, so reporting activity is linkable. Is that acceptable,
  or do we need single-use report tokens that hide who reported?
- What should an app do when the checking service is unavailable?
- How are issuers discovered, and how does an app express "honour these
  issuers" in a way users can inspect?
- Which second consumer (Atlas reviews, seller ratings, spam lists) should the
  general claim format be designed against?

## License

[LGPL-3.0](LICENSE), like the rest of the Freenet ecosystem.
