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

## The principle

**Hiding is fast and reversible. Influence is scarce and accountable.**

The two possible mistakes have very different costs. Hiding a legitimate item
by mistake is recoverable: the item comes back when a moderator restores it.
Leaving illegal material visible is not recoverable. So the system hides
quickly on reports, and accepts that a false report will sometimes hide a
legitimate item for a while. It does not try to stop malicious reports from
being made. It makes being wrong cost the reporter something they cannot
cheaply replace.

Four rules carry this:

1. **Weight comes from a scarce identity.** A report counts in proportion to
   the [ghost key](https://freenet.org/ghostkey/) that signed it. Ghost keys
   cost money that better hardware cannot reduce, so the many keys a false
   vote needs are expensive. A slow proof-of-work report is the free
   fallback, at a low fixed weight.
2. **Standing is earned slowly and lost quickly.** A key gains weight only
   when rulings confirm its reports, and it loses weight faster than it gains
   it. A key that took months to build is spent by one false report.
3. **The person a false report harms is the one who challenges it.** The
   uploader of a wrongly reported item has every reason to dispute, at little
   cost. Someone who uploaded illegal material has little reason to, since a
   dispute costs a key and puts the item in front of a moderator. An
   undisputed report is confirmed after a waiting period, and a disputed one
   goes to a person. People therefore look only at disputed items, and those
   are mostly legitimate.
4. **Judgment stays in the space, and trust is chosen by the viewer.** Each
   space's own moderator makes the rulings for that space. Each app and each
   user chooses whose rulings count toward reputation. A malicious moderator
   affects only the people who chose to honour them. Nobody judges centrally.

This gives up some things, and they are stated in "Threat model" below: a
griefer can hide a legitimate item until a moderator restores it, a space with
no moderator can neither restore nor remove anything, no one can overrule a
space's moderator inside that space, and the ghost-key mint is the one central
element.

## Scope decisions so far

- **No central service.** Everything runs in contracts, delegates and app
  UIs. No service operated by Freenet Project Inc. or anyone else fetches,
  examines or judges reported material, and no part of the system depends on
  one being online.
- **Reports hide; nothing automatic judges the content.** Enough weighted
  reports hide an item in the app's UI. Removal follows a moderator's ruling,
  or the absence of a dispute.
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
   child sexual abuse material reaches people trained to review it.
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
4. **Reports live where the app chooses.** Usually that is a generic report
   contract for the space, keyed by the space's stable id (see "App
   integration"), so that it survives re-keys of the app's own contracts. In a
   private, encrypted space, a public report contract would reveal that the
   space exists and link reporters' ghost keys to it, so the reports live
   inside the space's own state instead, readable only by members. Either way
   the shared crate checks each report's signature or proof-of-work when it
   arrives, and caps open reports per key and the number of proof-of-work
   reports per item. Contracts cannot read the clock, so these caps are counts,
   not rates.

### Hiding

An item is hidden in the app's UI once the weighted reports against it pass the
space's threshold. A report's weight is the reporter key's base weight
multiplied by its reputation. Base weight grows with the key's donation amount,
sublinearly and with a cap, and with the time since the key's first report in
the space, also capped. New keys start with low weight. A key with a long
record of reports confirmed by rulings can carry enough weight to hide an item
on its own, so the best reporters become the fastest.

Hiding is computed in the UI, or by a delegate, from the reports and the
rulings. It is not decided by the contract, so it never affects the contract's
merge. Two users might briefly disagree on whether an item is hidden, which is
harmless for display. A hidden item's bytes stay on the network until the
moderator removes it.

### Disputes

A dispute says a report is false. It is signed by a ghost key and capped per
key. Most disputes come from the uploader of a wrongly reported item, and an
uploader can co-sign the dispute with the key they published the item under,
to show the moderator it really comes from them. A dispute never unhides the
item. It puts the item in front of the moderator.

### Rulings

Rulings are the only thing that moves reputation, and they move it for both
sides.

| Ruling | Item | Reporters | Disputers |
|---|---|---|---|
| Restore | Unhidden | Lose reputation | Gain reputation |
| Remove | Removed (see "Removal") | Gain reputation | Lose reputation |
| Undisputed for N days | Removed automatically | Gain reputation | (none) |

Losses are larger than gains, so that standing is slow to build and quick to
lose.

The last row is what keeps moderators from having to look at reported
material. On a scheduled wake-up, the moderator's delegate (or, for an app run
from a server, the same code in the moderator's daemon) rules "remove" on
hidden items that nobody disputed within the waiting period. Delegates can
read the clock, so this works without anyone viewing the item. Each space sets
its own waiting period, and it can be zero: in an app whose items are pointers
to content elsewhere, such as Atlas's listings, an item that is hidden but not
yet removed tells anyone reading where reported material is, so such apps
should remove at once.

Only disputed items need a human decision, and since disputes come mostly from
wrongly reported uploaders, what moderators end up looking at is mostly
legitimate content. A moderator who confirms illegal material on a dispute
removes it and can report it to the CyberTipline.

Rulings are signed claims, published by the moderator in a rulings contract.
Each ruling references the reports it decides, so that reputation can be
updated, and so that anyone can measure how promptly a space's moderator
acts. A ruling stands even if the item has since dropped out of the app's
state, as items in busy chat rooms do. A ruling says what the moderator
decided; what it does to the item is up to the app (see "Removal").

### Reputation

A key's reputation rises with rulings in its favour and falls with rulings
against it. A key whose reputation reaches zero is ignored.

Whose rulings count is chosen, never fixed: "honour these issuers", the
general mechanism this project exists to provide. Reputation is computed from
the rulings of honoured issuers only, by the UI or a delegate.

- **By default a space honours only its own moderators.** A key's standing in
  a space then comes only from rulings made in that space. No issuer is
  honoured by default anywhere, including any run by Freenet Project Inc.
- **Apps may ship suggested lists**, such as the moderators of other spaces
  the app trusts. A suggested list must be visible to the user, show who is on
  it, and be changeable. This lets reputation carry across spaces and apps,
  which matters for apps whose moderators rarely rule. It also gives whoever
  writes the list influence, which is why it must stay visible and optional.
- **Each user can override their app's choice**, adding or removing issuers.

### Threat model

| Attacker | What they try | Mitigation | Trade-off accepted |
|---|---|---|---|
| Griefer | Hide legitimate items with false reports | New keys have low weight, so hiding needs several; each restore burns those keys' reputation, so the griefer keeps buying fresh low-weight keys | Honest first reports from new keys hide slowly. A false hide lasts until a moderator restores it |
| Rich attacker | Buy enough weight to hide at will | Donation counts sublinearly and is capped, so money buys keys, not weight; weight beyond the cap comes only from confirmed reports | Large honest donors carry no more weight than modest ones |
| Stockpiled keys | Mint keys early and let them age, to spend later | Age counts from a key's first report in the space, not from minting, and is capped | Honest reporters new to a space start weak there, however old their key |
| Disputer | Dispute illegal material to keep it up | Disputes never unhide; disputing keys lose reputation when the item is removed | A wrongly hidden item stays hidden until a moderator looks |
| Own-space reputation farm | Create a space, report and rule in your own favour, then spend that reputation elsewhere | By default a space honours only its own moderators, so rulings in the farm count only in the farm. They count elsewhere only where someone chose to honour the farm's moderator | By default reputation does not carry between spaces; carrying it needs someone to curate an honour list |
| Malicious or captured moderator | Remove legitimate items, restore illegal ones, or ruin honest reporters | Rulings are public and reference the reports they decide, so anyone can audit a moderator; reputation is affected only where the moderator is honoured; Atlas and the Official room can delist a space | Within its own space the moderator has the final say. There is no appeal above it |
| Sold or stolen key | Buy a high-reputation key and spend it on false reports | Losses exceed gains, so a few restores empty it; open reports per key are capped | Honest reporters who make a mistake lose more than they would under a symmetric rule |
| Farm-then-spend | Earn reputation with true reports, then spend it on false ones | Earning it means getting real bad content removed; losses exceed gains and open reports are capped, so the reputation can be spent about once | The farmed reputation can still hide some legitimate items once, until restored |
| Linkability | Correlate a reporter's activity across spaces through their ghost key | Report reasons are encrypted to the moderator; users may use a separate ghost key per space or context | A separate key has no reputation from the others. Users choose between privacy and standing |
| Proof-of-work flood | Hide items with many cheap proof-of-work reports | Proof-of-work reports have a low fixed weight and a cap per item, so they cannot hide an item on their own | Users without a ghost key have little influence, and a genuine crowd of them saturates the cap |
| No moderator | Grief a space whose moderator is absent | Atlas can require a working moderator, visible from the space's manifest and its public rulings, before listing a space | In such a space a false hide is permanent and nothing is removed |
| Ghost-key mint | Compromise the mint, or have it issue keys freely, to mint unlimited weight | The mint's blind signatures mean it cannot link keys to people even if compromised; keys from it still start with low weight and earn reputation only through rulings | The mint is the one central element. If it issued keys freely, Sybil resistance would fall back to reputation alone. Honouring more than one mint is a possible later step |

### Removal

A "remove" ruling is a claim. Each app enforces it in whatever way its content
allows, and the shared crate supports each of these:

- **A tombstone in the content contract**, where the app's contract accepts
  the moderator's key. This is the only kind that takes the bytes off the
  network. The tombstone keeps the item's reference and drops its bytes, and
  it wins every merge, so a peer holding older state cannot bring the item
  back.
- **Hiding by the app's UI**, from the published ruling, where the content
  sits in contracts the moderator cannot write to: contracts each author
  owns, or immutable blobs named by their hash. This is the fallback for any
  app, and the UI must say "hidden", not "removed".
- **An app-specific action**, such as banning the author as well, or
  removing everything that depends on the item, like River's edits and reply
  previews. The app decides what a ruling implies.
- **Delisting**, for an index such as Atlas, whose items are pointers to
  content elsewhere.

**Tombstones are self-verifying and anyone can deliver them.** The moderation
delegate signs the tombstone and publishes it with the ruling. Any client of
the app that sees it copies it into the content contract, and the contract
checks the signature against the moderators it has authorised. The generic
delegate therefore never needs to know an app's contract keys or update
format, which change from app to app and on every re-key. Hiding is already
immediate, so the cost of waiting for a client to deliver the tombstone is
small.

**Tombstones are kept as long as the contract could otherwise accept the item
again**, not forever. Apps that drop old content, as River does with old
messages, need a horizon below which nothing is re-admitted anyway, and the
tombstone can go once the item is below it.

UIs update in real time. A UI showing an item is subscribed to the contract
that holds it, so a tombstone reaches it as an ordinary update notification.
On receiving one, or a ruling it enforces by hiding, the UI removes the item
from the page at once, including any copy already rendered or cached in
memory, and can show a placeholder such as "Removed by the moderator". A UI
that only fetched the contract once sees the removal on its next read, so UIs
that display reportable content should subscribe.

A peer that holds an old copy and never syncs again keeps the bytes. Users
should be told an item was removed, not that it was erased from every disk.

## App integration (draft)

Freenet apps store content in very different ways, so the system makes as few
assumptions as it can about how an app is built.

### The pieces an app supplies

- **A space.** Whatever the app moderates as a unit: a River room, an Atlas
  index, a Delta site, a whole marketplace. The app gives it a **stable id**
  that does not change when the app's contracts re-key, such as the owner's
  verifying key plus a name. Report and rulings contracts are keyed by it.
- **Item references.** The app decides how items are identified. The
  reference is opaque to the system; a digest of at least 128 bits is
  recommended, so that nobody can make two items share one. A hash of the
  content is optional: it makes sense for media, where it also blocks
  re-uploads within the space, but not for text or encrypted content.
- **Subjects at more than one level.** A report or ruling can be about an
  item, a whole space (a room, a site), a contract, or a publisher's key.
  Atlas, for example, needs to act on whole apps, not single files.
- **A moderation manifest.** A small record, signed by the space's owner key,
  naming where reports go, where rulings are published, which moderation keys
  are authorised, the space's policy (hiding threshold, waiting period), and
  any suggested honour list. Indexes such as Atlas can then check
  mechanically that an app has a working removal mechanism, and measure from
  the public rulings how promptly it is used.
- **Where moderators are authorised.** In the manifest, or in the app's own
  owner-signed state, as a River room's configuration would hold it. Either
  way the shared crate does the checking.

### What an app gets

- A report button on any item, with the ghost-key path, the proof-of-work
  fallback, a reason only the moderator can read, and the CyberTipline link.
- Hiding within seconds once reports pass the space's threshold.
- A moderator queue of disputed items, with restore and remove.
- Automatic removal of reported items nobody disputes, without anyone looking
  at them.
- Removal from the network, where the app accepts tombstones, and real-time
  updates in every open UI.
- Reporters' reputation from every issuer the app or user honours.

### What an app does

1. **Publishes a moderation manifest** for each space.
2. **Chooses how removals are enforced**: accepting tombstones in its content
   contracts (best, and easiest built in from v1, since changing a contract
   re-keys it), hiding in the UI, or both.
3. **Adds the UI components**: the report flow, hiding, placeholders and the
   moderator queue. A Rust library comes first (River's UI is Dioxus), then a
   TypeScript package.
4. **Chooses whose rulings count toward reputation**: only the space's own
   moderators (the default), or a visible suggested list.

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
  freenet-stdlib 0.12.1 or later. The same logic is also a library, so an app
  moderated from a server, as Atlas is, can run it in its own daemon.
- **The moderation key is the delegate's own.** A delegate running in the
  background cannot message another delegate, so it cannot borrow a key held
  by the app's delegate, such as a River room owner's. The owner authorises it
  instead. This also means a moderation bot never needs the owner's key.
- **Testing needs a real node.** `freenet local` does not service a delegate's
  contract requests, and wake-ups do not fire there.

### How current apps fit

| App | Items | Enforcement | Notes |
|---|---|---|---|
| River | Messages, inline in the room contract | Tombstone, by adding a moderator-removal field to the room state (a routine re-key) | River's existing purge of direct messages is already this pattern. Moderators authorised in the owner-signed room configuration. Private rooms keep reports in room state. Message ids would use a 128-bit digest rather than River's 64-bit id. |
| Atlas | Listings pointing at other apps | Tombstone (Atlas owns its index); delisting for whole apps | Waiting period zero. Consumes manifests and rulings to decide which apps to list. |
| Delta | Pages in a site contract that only the site owner can write | Hiding, plus delisting by indexes, unless sites accept a moderator key | The space is the site; authority above it comes from an index or directory. |
| Harvest | Listings in a seller's store; photos as immutable contracts named by their hash | Hiding by photo hash | A future image contract with a tombstone slot would allow real removal. |
| Raven | Posts with content-hash ids, copied into several contracts | Tombstone in the shared contracts; hiding in authors' own shards | A report must reach every contract holding the post. |
| freenet-git | Repositories, with file contents in immutable pack contracts | Hiding, plus delisting by indexes | Same as Harvest for the packs. |

### Limits

- Automatic confirmation runs only while the moderator's node or daemon is
  running. If it is offline, removal waits.
- Where an app can only hide, the published rulings are a list of places
  where removed material may still be found. Rulings carry no public reason,
  which helps, but such apps should move toward formats that accept
  tombstones, such as blob contracts with a tombstone slot.
- Computing reputation from every honoured issuer's full ruling history gets
  expensive as histories grow. Issuers should publish compact, signed
  reputation summaries for UIs to read instead.

## Sharing removal decisions across apps (deferred)

Blocking a re-upload of removed material in other apps would be useful, but it
is not in v1, because every form we have found needs either an online service
or a public list, and a public list does harm:

- **A public list of content hashes can be tested by anyone.** Freenet content
  can be crawled, so anyone could hash what they find and check it against the
  list. For apps that can only hide, the list would point at material that is
  still retrievable. For uploaders, it would show which files are already
  known, so they could alter those before uploading.
- **Keeping the list private needs a service.** A rate-limited online check
  (for example an oblivious PRF) avoids both problems, but someone has to run
  it, which this design rules out.
- **Exact hashes are easy to evade.** Changing one byte gives a new hash.
  Perceptual hashes resist that but must never be published, since they can
  leak image content and can be deliberately collided.

Within one space, a content-hash item reference already blocks re-uploads of
the same file. Across spaces, a moderator who honours another moderator's
issuer can act on that moderator's rulings when the same item reference
appears. That reuses the existing honour mechanism, and needs no new list.

## A standalone moderation dApp (optional, later)

This is not in v1. In v1, moderation happens inside each app's own UI, through
the shared components, so a moderator works where the content is.

Because the report and rulings contracts are generic, keyed by each space's
stable id and found through each space's manifest, a standalone moderation dApp
is a natural later addition. It would give a moderator one queue across every
space they look after, in any app, with restore and remove signed by the shared
moderation delegate. Tombstones still reach the content, because any client of
the app delivers them.

It has limits. It cannot render every app's items (a River message or a Harvest
photo needs that app's own code), so for a disputed item it hands off to the
app. App-specific actions, such as banning an author, stay in the app. The
trust-settings app suggested under "Reputation" could be the same dApp.

## Considered and rejected

- **A central checking service** that fetches reported items and matches them
  against known-material hash lists. It needs an operator accountable for
  handling the material directly, which conflicts with the goal of a system
  with no central service.
- **A verdict list read through related contracts.** freenet-core resolves
  related contracts only when a contract is validated, and never re-validates
  when the related contract changes, so peers that merged an item before the
  verdict would keep it and never converge.
- **Disputes or counter-reports that unhide, or that lower reporters'
  reputation directly.** Either would let the people who want illegal material
  up keep it up, or burn the reputation of the people reporting it.

## Cost

Reporting, hiding and rulings cost almost nothing to run: reports are small
contract updates, and hiding and reputation are computed locally. Automatic
confirmation is a scheduled wake-up of the moderator's own delegate. Nothing
runs on shared servers.

## Open questions

- What should the hiding threshold be, and should it scale with the size of
  the space?
- How long should the waiting period before automatic confirmation be? A
  hidden item stays on the network until it ends, so shorter is safer, but it
  must leave a wrongly reported uploader time to dispute.
- What exactly should base weight be as a function of donation and activity
  age, and where should the caps sit? How much should a ruling add, and how
  much larger should a loss be than a gain?
- How large should a proof-of-work report's weight be, and how many
  proof-of-work reports should count per item?
- How are issuers discovered, and how does an app present a suggested honour
  list so users can inspect and change it?
- Should reputation decay over time, so a dormant high-reputation key is worth
  less to buy or steal?
- How is a moderation key revoked? A contract cannot read the clock, so if it
  stopped accepting a key's tombstones once the key was revoked, peers that
  saw a tombstone before the revocation would keep it and peers that saw it
  after would not, and they would never agree. One option is that a tombstone
  carries the authorisation it was signed under and stays valid for good,
  while clients refuse to deliver new tombstones from revoked keys.
- How do reports work in encrypted spaces, where only members can see the item?
- Should apps be able to honour more than one ghost-key mint, and weight keys
  by mint?
- Which second consumer (Atlas reviews, seller ratings, spam lists) should the
  general claim format be designed against?

## License

[LGPL-3.0](LICENSE), like the rest of the Freenet ecosystem.
