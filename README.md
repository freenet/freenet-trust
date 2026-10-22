# freenet-trust

Shared infrastructure for **signed claims about things on Freenet**: an issuer
says something about a subject (a piece of content, a key, a contract), and any
Freenet app can choose which issuers it honours.

The first use is helping app developers deal with illegal content, through
**community reporting with ghost keys**. Later uses could include spam and scam
lists, seller ratings and reviews.

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
give them a small kit that meets the requirement: a report button that hides
reported material quickly, a reputation system that makes false reporting
self-defeating, and a way for the app's moderator to remove confirmed
material from the network, mostly without anyone having to look at it.

## Scope decisions so far

- **Reports hide; nothing automatic judges the content.** Enough weighted
  reports hide an item in the app's UI. No service fetches or examines
  reported material. An automatic checker was designed and set aside for now
  (see "Automatic checking (later)").
- **Ghost keys give reports their weight.** A report is signed by a
  [ghost key](https://freenet.org/ghostkey/). Ghost keys cost money that
  better hardware cannot reduce, so an attacker cannot cheaply mint the many
  keys a vote needs. Users without one can make a slower, low-weight report
  with proof-of-work, so that reporting illegal material never costs money.
- **Disputes never unhide anything.** If disputes could bring an item back,
  the people most motivated to dispute illegal material would be the people
  who want it up.
- **Reputation moves only on rulings.** Raw reports and disputes can be bought
  by either side, so neither directly changes anyone's reputation. A ruling by
  a moderator does, including the automatic confirmation of an undisputed
  item.
- **Moderation stays in the app layer.** Each app's moderator restores or
  removes reported items in their own space. Nothing here requires changes to
  freenet-core. An app that ignores or buries reports is handled the way it is
  today: users report the app itself to Atlas and the Official room, and those
  can delist it.
- **A general system, minimal at first.** A ruling (an issuer signs a claim
  about a subject) is the general shape, so this is designed as a general
  claims system with moderation as its first application.

## How it works (draft)

### Reporting

1. **Any user can report an item.** The report names the item and is signed
   by the reporter's ghost key. The report screen also links to
   [NCMEC's CyberTipline](https://report.cybertip.org), so that suspected
   child sexual abuse material reaches people trained to review it and, once
   confirmed, the hash lists other platforms use.
2. **Users without a ghost key can report with proof-of-work.** The puzzle
   takes about a minute on typical hardware. While it runs, the UI shows "Skip
   the wait with a ghost key". The wait also gives a would-be false reporter
   time to reconsider. The puzzle is memory-hard (for example Argon2id), so
   that GPUs and custom hardware gain little, and it is bound to the report, so
   one solution cannot be reused. A proof-of-work report has no lasting
   identity and so no reputation; it counts as a low, fixed weight.
3. **The public part of a report does not say why.** Anyone in the app can see
   that an item has been reported and with what weight, which UIs need in order
   to hide it. The reason (illegal material, spam, harassment) is encrypted to
   the app's moderator. Reports of illegal material then look like any other
   report, so the report list does not work as an index of likely illegal
   material.
4. **Reports live in a report contract for the space** (for example one per
   River room). It checks each report's signature or proof-of-work when the
   report arrives, and caps open reports per key and the number of
   proof-of-work reports per item. Contracts cannot read the clock, so these
   caps are counts, not rates.

### Hiding

An item is hidden in the app's UI once the weighted reports against it pass the
app's threshold. A report's weight is the reporter key's base weight (from its
donation amount and age) multiplied by its reputation. New keys start with low
weight. A key with a long record of reports confirmed by rulings can carry
enough weight to hide an item on its own, so the best reporters become the
fastest.

Hiding is computed in the UI, or by a delegate, from the reports and the
rulings. It is not decided by the contract, so it never affects the contract's
merge. Two users might briefly disagree on whether an item is hidden, which is
harmless for display. A hidden item's bytes stay on the network until the
moderator removes it.

### Disputes

A dispute says a report is false. It is signed by a ghost key and capped per
key. Most disputes come from the uploader of a wrongly reported item. A dispute
never unhides the item. It puts the item in front of the moderator.

### Rulings

Rulings are the only thing that moves reputation, and they move it for both
sides.

| Ruling | Item | Reporters | Disputers |
|---|---|---|---|
| Restore | Unhidden | Lose reputation | Gain reputation |
| Remove | Tombstoned (see "Removal") | Gain reputation | Lose reputation |
| Undisputed for N days | Tombstoned automatically | Gain reputation | (none) |

The last row is what keeps moderators from having to look at reported
material. The moderator's delegate, on a scheduled wake-up, tombstones hidden
items that nobody disputed within the waiting period, and that counts as a
ruling. Delegates can read the clock, so this works without anyone viewing the
item. Only disputed items need a human decision, and since disputes come
mostly from wrongly reported uploaders, what moderators end up looking at is
mostly legitimate content. Someone who uploaded illegal material has little
reason to dispute: it costs a ghost key, and losing burns that key's
reputation.

A moderator who confirms illegal material on a dispute removes it and reports
it to the CyberTipline.

Rulings are signed claims, published by the moderator in a rulings contract.

### Reputation

A key's reputation rises with rulings in its favour and falls with rulings
against it. A key whose reputation reaches zero is ignored.

Reputation carries across apps, so that rulings in one app count in every app.
Without that, reputation would mean little in apps whose moderators rarely
rule. But a malicious moderator could then ruin honest reporters everywhere.
So each app, and each user, chooses whose rulings count toward reputation:
"honour these issuers", the general mechanism this project exists to provide.
Reputation is computed from the rulings of honoured issuers only, by the UI or
a delegate.

### How each attack plays out

- **Mass false reports (griefing).** Hiding one legitimate item costs enough
  keys to pass the threshold. Each restore burns those keys' reputation, so the
  griefer must keep buying fresh keys, which start with low weight, and the
  hide lasts only until a moderator restores it.
- **Disputing illegal material to keep it up.** Disputes do not unhide, so this
  protects nothing. When the moderator removes the item, the disputing keys
  lose reputation.
- **Ruining honest reporters.** This needs rulings to go against them, and only
  moderators make rulings, so cheap keys cannot do it.
- **Disputing just to force a moderator to view illegal material.** Costly,
  capped per key, and it burns the disputing key's reputation when the
  moderator removes the item.
- **Farming reputation to spend later.** Earning it means making reports that
  rulings confirm, which removes real bad content.
- **Apps without an active moderator.** Nothing gets ruled on, so a griefer's
  hide is permanent and nothing gets removed. This is the main weakness; it is
  why apps that show user content need a moderator anyway.

### Removal

The moderator removes an item with a tombstone in the contract that holds it.
A shared crate provides the tombstone type, its verification and its merge
rule, so apps do not hand-roll them.

A tombstone keeps the item's id and content hash and drops its bytes. It wins
every merge, so a peer holding older state cannot bring the item back, and it
must never be evicted by a cap on the contract's size. Because it is an
ordinary contract update, every subscribed peer applies it within seconds.

UIs update in real time. A UI showing an item is subscribed to the contract
that holds it, so the tombstone reaches it as an ordinary update notification.
The shared crate gives UIs a way to recognise a tombstone and read its reason.
On receiving one, the UI removes the item from the page at once, including any
copy already rendered or cached in memory, and can show a placeholder such as
"Removed by the moderator". A UI that only fetched the contract once, without
subscribing, sees the removal on its next read, so UIs that display reportable
content should subscribe.

A peer that holds an old copy and never syncs again keeps the bytes. Users
should be told an item was removed, not that it was erased from every disk.

## App integration (draft)

### What an app gets

- A report button on any item, with the ghost-key path, the proof-of-work
  fallback, a reason only the moderator can read, and the CyberTipline link.
- Hiding within seconds once reports pass the app's threshold.
- A moderator queue of disputed items, with restore and remove.
- Automatic removal of reported items nobody disputes, without anyone looking
  at them.
- Removal from the network by tombstone, with real-time updates in every open
  UI.
- Reporters' reputation from every app, from day one.

### What an app does

1. **Accepts tombstones in its content contract**, using the shared crate,
   from moderation keys that the space's owner has authorised. This is the
   only change to the app's own contract code. It re-keys the contract like
   any contract change, so it is easiest to build in from v1. Identifying items
   by a content hash also allows blocking re-uploads.
2. **Creates a report contract per space**, from a generic contract this
   project provides.
3. **Adds the UI components**: the report flow, hiding, tombstone placeholders
   and the moderator queue. A Rust library comes first (River's UI is Dioxus),
   then a TypeScript package.
4. **Chooses whose rulings count toward reputation**, or accepts the default.

The app writes no delegate.

### Delegates

- **Reporters use the existing ghostkeys delegate.** It already signs for
  third-party apps, shows its own consent prompt, and gives them only a
  restricted scope, so apps build no permission flow. Its key changes with
  every release, so apps find it through its
  [pointer record](https://github.com/freenet/ghostkeys/blob/main/FREENET.md),
  never a hard-coded key. The proof-of-work runs in the browser.
- **Moderators use a shared moderation delegate that this project
  publishes**, a platform delegate like ghostkeys, found the same way through
  a pointer record. It holds the moderation key, which decrypts report reasons
  and signs rulings and tombstones, and it runs automatic confirmation on
  scheduled wake-ups once the moderator grants it Background. That needs
  freenet-stdlib 0.12.1 or later.
- **The moderation key is the delegate's own.** A delegate running in the
  background cannot message another delegate, so it cannot borrow a key held
  by the app's delegate, such as a River room owner's. Instead the owner signs
  a one-time authorisation: "this key may moderate my space".
- **Testing needs a real node.** `freenet local` does not service a delegate's
  contract requests, and wake-ups do not fire there.

### Limits

- Automatic confirmation runs only while the moderator's node is running. If
  it is offline, removal waits.
- Where each user owns the contract that holds their content (a blog per user,
  for example), the moderator cannot tombstone unless those contracts accept
  the moderation key. Hiding is then all the system can do.
- Computing reputation from every honoured issuer's full ruling history gets
  expensive as histories grow. Issuers should probably publish compact,
  signed reputation summaries for UIs to read instead.

## Sharing removal decisions across apps (draft, possibly not v1)

When a moderator removes something as illegal, the decision can be published
as a claim, so other apps that honour that moderator can block re-uploads of
the same file.

- **Content is identified by a hash of its bytes**, not by where it is stored.
  That works however an app stores media, matches the same file across apps,
  and survives contract re-keys.
- **The list must never become a directory of illegal material.** Hashing the
  subject is not enough: Freenet content can be crawled, so anyone could hash
  what they find and test it against a public list. Checks therefore need to be
  online and rate-limited, for example with an oblivious PRF (as in Google's
  Password Checkup): the checker learns whether an item it already holds is
  listed, the service learns neither the item nor the result, and nobody can
  enumerate the list.
- **Checks happen where the moderator is, not where every reader is.** For
  River, the room owner's node checks new and recent media in the room. Check
  volume then follows uploads, not views.
- **Private spaces are partly covered.** In an encrypted space, such as a
  private River room, the owner's node holds the decrypted bytes and can run
  exact-hash checks without revealing anything to the service.

## Automatic checking (later)

An earlier draft had a checking service, operated by Freenet Project Inc.,
that fetched reported items and matched them against NCMEC's lists of known
material (PDQ and MD5 hashes, using Meta's open-source
[Hasher-Matcher-Actioner](https://github.com/facebook/ThreatExchange/tree/main/hasher-matcher-actioner)).
On a match it would tombstone the item within seconds, file a CyberTipline
report automatically, and preserve the file for a year, encrypted to an offline
key split among trustees.

It is set aside because fetching and keeping the material brings US legal
duties and questions that need answers before we build it: whether the
operator counts as a "provider" under 18 U.S.C. §2258A, which would make
fetching, reporting and preservation protected; and what NCMEC's hash-sharing
terms allow. If those are resolved, it can be added as an extra layer that
apps opt into, issuing rulings like any other honoured issuer. Notes from that
design:

- **Remove at the source, never publish a verdict feed.** A public list of
  confirmed items would point at material it cannot remove.
- **Act only on known material.** Classifiers for new material (Google's
  Content Safety API, Thorn's Safer) require human review before acting on a
  result, under the terms we found.
- **A verdict list read through related contracts does not work.**
  freenet-core resolves related contracts only when a contract is validated,
  and never re-validates when the related contract changes, so peers that
  merged an item before the verdict would keep it and never converge.
- **Perceptual hashes must never be published.** They can leak image content
  and can be deliberately collided, so matching happens only inside the
  service.

## Cost

Reporting, hiding and rulings cost almost nothing to run: reports are small
contract updates, and hiding and reputation are computed locally. Cross-app
checks are cheap too: an oblivious PRF evaluation is about one elliptic-curve
multiplication, so a million checks costs about a minute of CPU.

## Open questions

- What should the hiding threshold be, and should it scale with the size of
  the space?
- How long should the waiting period before automatic confirmation be? A
  hidden item stays on the network until it ends, so shorter is safer, but it
  must leave a wrongly reported uploader time to dispute.
- How should a ghost key's base weight depend on its donation amount and age?
  And how much reputation should a ruling add or take away?
- How large should a proof-of-work report's weight be, and how many
  proof-of-work reports should count per item?
- A reporter's ghost key is the same across apps, which is what makes
  reputation carry, but it also makes reporting activity linkable. Is that
  acceptable?
- How are issuers discovered, and how does an app express "honour these
  issuers" in a way users can inspect?
- How do reports work in encrypted spaces, where only members can see the item?
- Should v1 include cross-app sharing of removal decisions?
- Which second consumer (Atlas reviews, seller ratings, spam lists) should the
  general claim format be designed against?

## License

[LGPL-3.0](LICENSE), like the rest of the Freenet ecosystem.
