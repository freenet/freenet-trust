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
new uploads, and a feed the app subscribes to so that confirmed items
disappear from its UI in real time.

## Scope decisions so far

- **Reports are free; scanning needs a ghost key.** Ghost keys gate services
  that consume resources, and scanning is one. Anyone may report an item,
  with a ghost key or proof-of-work, because reporting illegal material
  should never cost money. Scanning new uploads before anyone reports them
  costs the service a check per upload, so a scan request must be signed by a
  [ghost key](https://freenet.org/ghostkey/), and the service limits scans
  per key.
- **Items are checked automatically, and nobody looks at the material.** The checking service matches reported files against vetted
  lists of known material and runs a purpose-built image classifier. No
  person and no general-purpose AI model views the material: we will not ask
  anyone to do that. Because a report carries no identifying information
  about any person (see "Reporting to NCMEC"), a wrong verdict cannot lead to
  anyone being investigated. The worst case is a legitimate item hidden by
  mistake, which can be undone.
- **The checking service has an accountable operator.** Running it carries
  legal duties: in the US, reporting apparent child sexual abuse material to
  NCMEC and preserving it for a year. Freenet Project Inc. would operate the
  service and register with NCMEC's CyberTipline. Both duties are automated.
- **Moderation stays in the app layer.** Nothing here requires changes to
  freenet-core. Each app decides whether to honour the checking service's
  verdicts, its own moderator's decisions, or both. An app that ignores or
  buries reports is handled the way it is today: users report the app itself
  to Atlas and the Official room, and those can delist it.
- **A general system, minimal at first.** The shape of a verdict (an issuer
  signs a claim about a subject) is general, so this is designed as a general
  claims system with moderation as its first application. Trust scoring and
  aggregation are left until a concrete consumer needs them.

## How reporting works (draft)

1. **Any user can report an item.** The report goes to the checking service,
   or to the app's own moderator if the app prefers one, for example a River
   room's owner. For suspected child sexual abuse material, the report UI also
   links to [NCMEC's CyberTipline](https://report.cybertip.org).
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
6. **A confirmed item disappears in real time.** The service publishes its
   verdict to the app's verdict feed, the app's UI hides the item within
   seconds, and the app then removes it with its own mechanism, such as a
   tombstone in the contract.

## The checking service (draft)

### What it checks and what happens

| Result | Hidden | Reported to NCMEC |
|---|---|---|
| Matches a list of known material | At once | Automatically |
| Flagged by the classifier only | At once | Only above a strict threshold or with corroboration (below) |
| Neither | No | No |

Known-material lists and classifiers (for example PhotoDNA with NCMEC and IWF
hash lists, Thorn's Safer, Google's Content Safety API) are available only to
vetted organisations, so Freenet Project Inc. would apply for access.
Perceptual matching (PhotoDNA, PDQ) runs inside the service and is never
published (see "Explored and deferred").

Hiding is cheap to get wrong and easy to undo, so the threshold for hiding
is low. Filing is neither, so the bar for filing is high. Known-material
matches have a near-zero false-positive rate and are filed automatically. A
classifier-only hit is filed only when the score clears a strict threshold or
something corroborates it, such as several independent reports. Filing every
classifier hit would bury NCMEC in reports it cannot use and teach it to
discount ours. Providers that report in good faith are protected from
liability (18 U.S.C. §2258B), but that protection does not cover reckless
disregard, and knowingly filing a high volume of false positives could come
close to it. Whether a classifier score alone counts as the "actual
knowledge" that triggers the duty to report, and where the threshold sits,
are for counsel to settle before launch. Either way, the item is hidden.

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
owner's node (for example a moderation delegate, using scheduled wake-ups)
submits new media in the room. Scan volume then follows uploads, and the
limit attaches to the moderator's key.

In a private, encrypted space, scanning means sending decrypted bytes to the
service. That is the owner's choice to make, and members should be told it is
on. An owner who does not want that can still use exact-hash checks (see
"Sharing removal decisions across apps"), which reveal nothing to the service.

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
person to pass them to. They are still useful: NCMEC adds the file to the hash
lists it shares with other platforms, and its victim-identification work runs
on image content rather than on who uploaded it. We should explain the shape of
these reports to NCMEC when registering.

Reported material is preserved encrypted for the required year and then
deleted, automatically.

### The verdict feed

Each app has a verdict contract, keyed by the service's public key and the
app's id, that only the service can write to. Each entry names the item's
locator and content hash and is signed by the service. The app's UI
subscribes and hides matching items as entries arrive, so the real-time
notification is an ordinary contract update.

The feed tells anyone running the app which items were flagged, and the bytes
remain fetchable until the app removes them. So apps should tombstone promptly,
and entries for items already tombstoned can be pruned.

### Speed

Pushing a verdict to subscribers is fast. The slow step is the service
fetching the reported bytes, since a network GET can currently take far longer
than a few seconds. If that proves too slow, the reporter's client could attach
the bytes, encrypted to the service. That works only if the app identifies
items by a content hash recorded in its contract, so the service can confirm
the bytes are the item at that locator with a small read instead of a large
fetch.

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

Checking an image is cheap: a hash lookup and one classifier run. Report
volume is bounded by the ghost-key or proof-of-work gate. Scan volume follows
uploads and is the larger cost, which is why each scan needs a ghost key and
each key has a limit. Fees for vetted detection services are not yet known. Cross-app checks
are cheap too: an oblivious PRF evaluation is about one elliptic-curve
multiplication, so a million checks costs about a minute of CPU. The main
cost there is the network traffic of carrying checks through contracts, which
is why they are batched and made per upload rather than per view.

## Explored and deferred

- **Perceptual hashes** (PDQ for images, TMK+PDQF or vPDQ for video) catch
  modified re-uploads. They cannot use an oblivious exact-match check, they
  can leak image content (PhotoDNA has been shown to be invertible to
  thumbnails), and they can be deliberately collided. So a perceptual list must
  never be published, and matching happens only inside the checking service.

## Open questions

- With nobody reviewing, how does someone appeal a wrongly hidden item?
- What threshold or corroboration should a classifier-only hit need before it
  is filed with NCMEC? (A question for counsel.)
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
- How should revocation work, and what transparency (counts by claim type, for
  example) should an issuer publish?
- Which second consumer (Atlas reviews, seller ratings, spam lists) should the
  general claim format be designed against?

## License

[LGPL-3.0](LICENSE), like the rest of the Freenet ecosystem.
