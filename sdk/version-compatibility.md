# Version compatibility and updates

The written policy behind [#174](https://github.com/Iskandarus/photo-duplicates-manager/issues/174), part of the platform epic [#139](https://github.com/Iskandarus/photo-duplicates-manager/issues/139). Documents only, with one exception. Nothing here was implemented when it was written, and the issues named throughout are where each piece lands. The exception is the **shared frontend package** line added to section 2: that one shipped with #247, minors, negotiation, changelog rule and all, and its rules are enforced by tests in CI rather than waiting for an issue. Read every other row as a plan.

The contract already specifies the *mechanism* of version negotiation - [`core-api.md`](core-api.md) section 13 - and deliberately stops short of the policy and points here. This document is that policy. There were two; the resource manager contract was withdrawn at 1.3 (#278) and [`resource-manager-api.md`](resource-manager-api.md) is the record of it.

Companion artifact:

- [`asp-back/contracts/CHANGELOG.md`](CHANGELOG.md) - the record every rule below refers to. Seeded with the three 1.0 entries and empty of changes on purpose.

Publishing a contract means someone else's code depends on it, on their release schedule. The desktop app updates itself wholesale; a third-party resource manager does not update with it. The compatibility policy has to be chosen before the first external resource manager exists, because afterwards it cannot be chosen freely.

---

## 1. What this decides

Four questions, in the order they will actually be asked:

1. How far back does support reach, and what ends it (sections 3, 4).
2. What a user sees when the two sides disagree (section 5).
3. How a change is announced, and how long before something is removed (section 6).
4. How an update lands ten services without ever leaving a mixed set behind (sections 7, 8).

Everything else here exists to make one of those four answerable by someone who did not read the epic.

---

## 2. Eight version lines, and only three of them are contracts

| Line | Shape | Declared in | Negotiated | Owner |
| --- | --- | --- | --- | --- |
| Resource manager data contract | `MAJOR.MINOR` | `manifest.contractVersion`, plus the `/v1` path | per request, `PDM-Contract-Version` | #142, **withdrawn at 1.3 by #278** |
| Resource manager UI message protocol | `MAJOR.MINOR` | `manifest.ui.protocolVersion` | once, at the handshake | #142 section 11, **retired at 1.0 by #197** |
| Core API | `MAJOR.MINOR` | `PDM-Core-Version` alone since 2.0 - see 3.2 | per request, `PDM-Core-Version` | #166 |
| Shared frontend package | SemVer `X.Y.Z` | `@pdm/frontend-shared`'s `package.json`, and `SHARED_VERSION` in the bundle | once, in the frame session | #247 |
| Manager frontend package | SemVer `X.Y.Z` | `@pdm/manager-frontend`'s `package.json` | no - every consumer is built from this repository | #250 |
| Product version | SemVer `X.Y.Z`, from the git tag | `tauri.conf.json`, `latest.json`, container image tags | no | release process |
| Export document format | `X.Y` | the `version` member of the exported document | no; the importer reads every older form | #143 |
| Database schema | EF migration id | `__EFMigrationsHistory`, per provider | no; forward only | #143 |

The first three were promises to other people's code, and one of them is left: the core API. The two above it were withdrawn without ever having an implementer outside this repository, which is what made the waiver in 3.2 available both times. The last three are ours and appear here only because they are the lines most likely to be confused with the first three.

The **fourth and fifth are neither**, and are the two additions this document has taken since it was written. Both are promises to **our own code**, and they differ in one way that decides how much machinery each needs.

`@pdm/frontend-shared` is shipped separately from at least one of its consumers: the app host and each internal manager frontend bundle their own copy, so two versions of it run in one browser window as a matter of course (#247, part of #217). It therefore follows the rules below in full - minors additive and never expiring, negotiation down to the older side, a major gap refused, and a changelog whose newest entry must agree with the shipped version.

`@pdm/manager-frontend` (#250) has four consumers and all four are built from this repository, in the same command, so **two versions of it never meet**. It carries a version and a changelog anyway, for the half of the rule that still applies: a change that is not written down is invisible to whoever ships the next release, and four bundles land in four services that a deployment may update at different moments. What it does not need is negotiation, because there is nobody to negotiate with.

Both keep their changelog beside the package rather than in [`asp-back/contracts/CHANGELOG.md`](CHANGELOG.md), because that file is read by a third party writing a resource manager and neither package is part of what they implement - an external manager is never embedded and draws its own screens. See `shared-frontend-package.md` section 4 and [`resource-manager-frontend.md`](resource-manager-frontend.md) section 1.2.

**The SDK is not a ninth line** (#192). `pdm-resource-manager-sdk-2.0.0.zip` - the contract and the documents around it, as one download for somebody who does not have this repository - carries a version, and that version is the **core API's own** plus a revision counting re-publications of the same contract. It declares nothing and negotiates nothing: a client built from that bundle negotiates `PDM-Core-Version`, which is row three. A number of its own would be a second number about the same thing, which is what this table exists to prevent, and its changelog is the contract's changelog because there is nothing in the bundle that changes independently of the contract.

**`pdm-harness` carries that same number, and is not a line either** (#194). It plays the caller's half of the same document, ships from the same release, and a fix in it is a re-publication - which is what the revision counts. What would make it a version line is somebody having to negotiate with it, and nobody does: it is a program you run, and the only version it has an opinion about is the core API it serves. See `resource-manager-sdk-distribution.md` and [`manager-harness.md`](manager-harness.md).

### The product version is not a contract version

- Shipping `0.9.0` -> `1.0.0` does not bump any contract. Shipping `0.9.0` -> `0.9.1` may bump one.
- A contract version is never derived from, printed as, or compared against the product version. The moment a third party can infer one from the other, they will pin to the wrong one.
- The one rule that does bind them, in the other direction: **a contract change must ship in a product release, and that release's notes must name it**, because the contract reaches anybody only through a release.

---

## 3. The support window

### 3.1 Within a major, minors do not expire

**PDM supports every minor of a major, for the life of that major.** There is no "three minors back" and no clock.

That is a stronger promise than #174 asked for, and it is cheaper than the weaker one. Minors are additive by rule (section 4), so every field a later minor adds is optional, which means the code path for its absence had to exist on the day the field was added. Supporting minor 1.0 from a 1.7 core is not extra work - it is the work already done, kept. A window measured in minors would be a promise to eventually *delete* fallbacks we are keeping anyway, in exchange for cutting a user off from their photos.

What "supported" means, and what it does not:

| Supported means | Supported does not mean |
| --- | --- |
| The registration is usable and every operation of the negotiated minor works. | It receives features added later. A manager on 1.0 will never use an operation introduced in 1.4. |
| No expiry date, no countdown in the admin panel. | Bug-for-bug behaviour. Fixes land in the current minor and are not backported into a negotiation path. |
| The boundary combinations are exercised in CI - current, the floor, and one below it (section 10). | That every minor is exercised. The number of live combinations grows with every minor and the rehearsal stays at three points, which is a deliberate trade and not an oversight. |
| Nothing is removed from under a caller inside the major. | Performance parity. An old manager may take the slow fallback, and that is visible rather than hidden (section 5.5). |

### 3.2 Majors overlap for twelve months

A major bump is the only routine way anything is removed. The path prefix makes an overlap possible - `/v1` and `/v2` are different routes in the same process - so an overlap is what we use.

**Unless there is nobody to overlap with.** Three versions have used that waiver, for the same reason every time: the document had no implementer outside this repository and could not have had one, because the SDK was not published (#192 was open). The UI message protocol was withdrawn at 1.0 with no window (#197); core API 2.0 was declared without one (#274) and **served without one** (#278), so a 1.x client is refused by name rather than kept working; and the resource manager API was **withdrawn outright at 1.3** (#278) rather than superseded, because the reversal of #254 left nothing for a manager to serve. The rule that survives is the one that matters - **the waiver is written into the changelog entry, not inferred from the absence of complaints** - so that the first version anybody outside can build against inherits the twelve months rather than the precedent.

**The waiver is spent by a publication, and no publication has happened yet.** #192 built the mechanism - the contract and the documents around it pack into a downloadable bundle and a tag publishes it - and **no `sdk-v*` tag has been cut**, so the bundle has never left this private repository. That is checkable rather than remembered: the releases of the public `photo-duplicates-manager-releases` repository carry the desktop's `v*` tags and nothing else.

So a **fourth** version used the waiver: core API 3.0 (#318) removed `POST /v1/events` with the notification channel it relayed into, on exactly the same ground, and its changelog row says so. What does not change is the rule that matters - the waiver is written into the changelog entry, never inferred from the absence of complaints - and what it now attaches to: **the first version anybody outside can build against inherits the twelve months**, and from the day that bundle is published that is whatever version it carries. It is spent by the publication and not by a count of downloads, because an implementer we do not know about is the case this rule exists for. See `resource-manager-sdk-distribution.md`.

A withdrawal is a removal with no successor, and it is held to the same rule: the row says what went, why no window was owed, and what replaces it. A document that simply stopped being mentioned would be indistinguishable from one nobody had updated.

Core API 2.0 **kept its path prefix**, and 3.0 kept it too. That is not a contradiction of the paragraph above, it is the consequence of it: `/v1` and `/v2` exist so that two majors can be served side by side, which is the overlap that is not being run, and moving the prefix with no second major ever served would cost every caller a rewrite of every URL and buy nothing. What refuses a mismatched major is `PDM-Core-Version`, which answers `400 unsupported_core_version` naming both - a diagnosis where a moved prefix would give a bare 404 on a path that used to work. The moment this API has a client not built in this repository, the prefix is the first thing to reconsider. See section 13 of [`core-api.md`](core-api.md).

- When a new major first ships, the previous one keeps being served for **at least twelve months, and at least two product minor releases**, whichever ends later.
- The removal date is fixed on the day the new major ships, published in the changelog, and can be moved later but never earlier.
- From that same day, every registration still on the old major is flagged in the admin panel with the date. This is the one version notice that is unconditional (section 5.4).
- After the overlap the old major stops being served. Registrations on it become unusable and say why; saved results stay readable in the degraded state #172 defines.

A major bump requires re-registration, which #142 section 13 states and this is the reason: a grant is expressed in scopes and capabilities that a major may redefine, so an administrator re-consents rather than having an existing grant silently reinterpreted against a new vocabulary.

### 3.3 The only way a minor stops being supported

A **support floor**: the oldest minor the core will negotiate, normally the first minor of the major. It may be raised only for a defect that cannot be fixed additively - in practice, a security defect where continuing to speak an old minor is itself the vulnerability.

Raising it requires all of:

- an entry in the changelog naming the minor, the defect class and the date;
- a line in the release notes of the release that raises it;
- at least **30 days** between the announcement and the release, unless the defect is being actively exploited;
- the admin panel flagging affected registrations for the whole notice period.

On the desktop the floor arrives with an app update the user did not ask for, so the notice has to survive that: the app shows a one-time, dismissible explanation naming the manager and the remedy. A manager that silently vanishes from the switcher after an update is not an acceptable outcome even for a security fix.

### 3.4 The window, per direction

| Direction | Other side is older | Other side is newer |
| --- | --- | --- |
| PDM calls a resource manager (#142) | Supported for the life of the major. Negotiated down to the manager's minor. | Supported. Negotiated down to the core's minor; the manager's newer fields are ignored. |
| A client calls PDM (#166) | Supported for the life of the major. PDM serves the client's minor. | Supported. PDM serves its own minor and says so in the response header. |

### 3.5 There used to be a third line

The UI message protocol had its own window and its own failure mode here: a data-contract gap costs an operation, while a protocol gap could mean a frame that never finished booting.

It is gone. [#139](https://github.com/Iskandarus/photo-duplicates-manager/issues/139) was revised - a resource manager is a separate application and PDM renders the entire user interface - and `resource-manager-ui-protocol.v1.schema.json` was retired without ever being implemented (#197). `manifest.ui.protocolVersion` is deprecated along with the rest of that block and is negotiated with nothing.

An application of a manager's own that calls PDM is on the **core API** line, row two of the table above, and negotiates per request with `PDM-Core-Version` like any other client.

### 4.1 Allowed in a minor

| Change | Why it is safe |
| --- | --- |
| Add an optional request field | The receiver ignores what it does not know. |
| Add a response field | Same, in the other direction. |
| Add an operation | The negotiated minor gates the call; an older side is never asked. |
| Add an enum member | Only under the rule in 4.3. |
| Add an error code | A reader that does not know it falls back to the HTTP status. |
| Add a capability flag | Absent reads as false, so an old side reads as "cannot", which is true. |
| Relax a constraint - widen a `maxLength`, lower a `minimum` | Every value that was valid stays valid. |
| Add an optional header | Optional in both directions by definition. |

### 4.2 Not allowed in a minor

| Change | What breaks |
| --- | --- |
| Remove or rename anything | The other side is still sending or reading it. |
| Make an optional field required | Old senders omit it and start failing. |
| Tighten a constraint | Values that were legal become 400s with no version signal. |
| Change the status or error code for an existing situation | Error handling is behaviour, not decoration. |
| Change a default | An unchanged caller gets different behaviour with no change on its side. |
| Change what an existing field may contain, where an old reader would mishandle the new values | The field name is unchanged, so nothing warns anyone. |
| Add a required field "with a default" | That is the previous row with better manners. The old sender still omits it. |

### 4.3 Who tolerates unknowns

One rule, both directions:

> The reader ignores what it does not know. The writer never needs the reader to understand a new value for the call to be correct.

The first clause is the easy half, and both contracts already spell out instances of it: an unknown scope in `GET /v1/capabilities` is a capability the client cannot use rather than an error, an unknown stage key renders from its `label`, an unknown error code is treated as its HTTP status, an unknown per-item code in a batch response is an item that failed.

The second clause is the half that gets lost. If a new enum member changes what the *reader must do*, then a reader that ignores it does the wrong thing, and the change was never additive. Adding `partially_moved` to a per-item operation status looks like an addition; an old core reads it as unknown, files it under failure, and tells the user nothing moved while half the selection did. That belongs in a major, or behind a capability flag the old side does not declare.

### 4.4 The exception additive-only cannot express: destructive semantics

**A field whose absence changes what a destructive operation does may never be added in a minor.**

The failure is concrete. Suppose 1.3 adds an optional `dryRun` to `files:delete`. The safe behaviour now depends on the *receiver* understanding a field, while 4.3 entitles the receiver to ignore fields it does not know - so a 1.0 manager that is handed `dryRun: true`, by a core bug, by a lying manifest, or by a proxy that replayed a request, does exactly what it is supposed to do and deletes the photos. The user asked for a preview and got a deletion, no rule in 4.1 was broken, and nothing on the wire says anything went wrong.

So, for anything in either contract whose name contains delete, recycle, move or purge:

- New behaviour for an existing destructive operation goes behind a **capability flag**, never behind a request field alone. The core uses it only when the flag is declared *and* the negotiated minor is at least the one that introduced it - both conditions, because a lying manifest is a case the contracts already anticipate.
- The alternative is a **new operation with a new name**, which an old manager answers with 404 and nobody loses a photo.
- The same reading applies to the core API: PDM may not begin honouring a new field on `POST /v1/files/delete` in a way that a caller which omits the field would find surprising.

`Idempotency-Key` is the model to copy. It changes what a retry does, so it is required from the start rather than added later.

### 4.5 Limits and capabilities are data, not contract

`manifest.limits` may change at any time with no version bump. It describes this deployment of this manager, not the shape of the contract. The core re-reads it with the manifest and honours the new value on the next call; a manager lowering `maxOperationBatchSize` mid-session is legal, and the core must not have cached the old value into a request it is still assembling.

`manifest.capabilities` behaves the same way with one asymmetry that follows from #144 and #145: **losing** a declared capability takes effect immediately, because declared AND granted can only shrink; **gaining** one changes nothing until an administrator grants it.

---

## 5. On a mismatch: degrade, do not refuse

### 5.1 The principle

Refusing turns a user's photos unreachable on an app update they did not ask for. Both contracts already choose degradation; this section is the policy version of that choice, including the short list of cases that are refused anyway.

### 5.2 The situations from the issue, answered

| Situation | What happens | What the user sees |
| --- | --- | --- |
| App updates, manager stays old, contract minor changed | Negotiated down to the manager's minor. Everything it could do before, it still does. | Nothing, unless a capability is actually lost; then one line above the working area. |
| Manager updates, app stays old | Negotiated down to the core's minor. The manager's newer fields are ignored. | Nothing. |
| Manager declares a version we no longer support | Refused. The registration stays and is marked unusable; saved results stay readable (#172). | A message naming the version it speaks, the oldest we serve, and the remedy. |
| Built-in manager, whose version is always ours | The same negotiation code runs (section 10). | Nothing. |
| Web: the operator upgrades the API but runs old manager containers, or the reverse | Supported inside the window, and reported. | Nothing user-facing; the admin panel shows the mixed set (section 8). |

### 5.3 Refusal is reserved for exactly three cases

1. A **major** mismatch, outside the overlap of section 3.2.
2. A minor **below the support floor** of section 3.3.
3. A manifest that **lies** - the manager rejects a `PDM-Contract-Version` the core computed as the minimum of the two, which the contract already answers with 400 `unsupported_contract_version`. Unreachable if both sides are honest, which is why it exists.

Everything else degrades. There is no fourth case, and adding one later is a policy change, not an implementation detail. Note that none of the three refuses the *resource*: the registration is marked unusable while its saved results stay readable.

### 5.4 Notices attach to lost function, not to numbers

- **The admin panel** always shows declared, negotiated, and a chip when a manager is behind. It is the version-reporting surface and it can be as pedantic as it likes (section 9).
- **The working area** gets a notice only when the gap removes something the user could otherwise have had. "This source cannot supply previews, so the app is downloading full images" earns a line. "This source speaks 1.2 and the app speaks 1.4" does not, and a banner that is always on is a banner nobody reads - including on the day it finally matters.
- **One unconditional exception**: once a removal date exists for a major (3.2) or the floor is being raised (3.3), every affected registration carries the notice from the day it is announced, whether or not anything is degraded yet.

### 5.5 Degradation must be observable

Every fallback the core takes because of a version gap is logged once per registration per boot, naming the negotiated minor and the feature lost, and counted for the bug report bundle (#175). Silent degradation is how "the app got slow after the update" becomes unanswerable. The thumbnail fallback in #142 section 8 already carries this requirement for the same reason; this generalises it.

---

## 6. Announcing a change

### 6.1 The changelog is the artifact

[`asp-back/contracts/CHANGELOG.md`](CHANGELOG.md) - one file for all three contract lines, because they change in the same commits and an implementer should read one file.

The rule that keeps it honest: **the `info.version` of each OpenAPI document and the newest entry for that line must agree**, which is a CI check rather than a habit - `ContractHygieneTests` in `PDM.Conformance.Tests`, run per pull request by `.github/workflows/contracts.yml` (#151). A contract change that is not in the changelog is a bug.

The shared frontend package keeps its own, `front-end/packages/shared/CHANGELOG.md`, in the same shape and under the same rule - `test/version.test.ts` reads its `package.json` and its changelog and fails when they disagree, run per pull request by the same workflow. It is a second file rather than a fourth line in the first one because the two have different readers: that file is read by somebody writing a resource manager, and an external manager never bundles this package.

### 6.2 Deprecation and Sunset, where PDM is the server

For the core API, the client may be an MCP configuration on a machine we have never seen, so the announcement has to travel on the wire:

- `Deprecation` and `Sunset` response headers (RFC 9745, RFC 8594) on any operation with a removal date. Both are defined in `components.headers` of [`core-api.v1.yaml`](core-api.v1.yaml) from 1.0, although nothing is deprecated yet, so that a generated client knows to read them and the header cannot be spelled two different ways once there is something to announce;
- the version that removes an operation in `GET /v1/capabilities` -> `deprecations[]`, which the contract already carries for exactly this purpose. `Sunset` gives the date, `removedIn` gives the version, and they are published in the same release note;
- neither is something a client is required to read. Removal still respects section 3.2 regardless of whether anyone noticed.

For the resource manager contract the direction reverses and there is nothing to send: PDM is the client, so the announcement is the changelog, the release notes and the admin panel.

### 6.3 Release notes

Every release whose payload changes a contract version says so, in this shape:

> Resource manager contract 1.2 (was 1.1): adds thumbnail cropping hints. Managers on 1.0 and 1.1 keep working unchanged.

The wording matters more than usual, because it is read by people who did not read the epic and who will otherwise assume a number going up means something they must do.

### 6.4 The timeline for a major

| When | What |
| --- | --- |
| T-0, the release that first serves `/v2` | Changelog entry, release note, removal date fixed. Every `/v1` registration flagged in the admin panel with the date. |
| T-0 onward | Both majors served side by side. `/v1` registrations keep working with no degradation. |
| At least two product minor releases, and at least 12 months | The overlap. Migration guidance ships with the SDK (#151). |
| Removal, in a named release | `/v1` stops being served. Registrations on it become unusable and say why. Saved results stay readable (#172). |

That is the timeline for a major with somebody on the far side of it. Nothing declared so far has had one, so nothing has run this table: the UI protocol was withdrawn at 1.0 (#197), core API 2.0 was declared and then served with the waiver written into its changelog rows (#274 and #278), and the resource manager API was withdrawn at 1.3 with a row of its own (#278). What replaces rows one to three in that case is a single release note saying that the old version stopped being served, and the honesty of the waiver - which is why 3.2 insists it is recorded rather than assumed.

## 7. Desktop: one version, one payload

### 7.1 The rule

Everything the installer ships is one version. The desktop payload is the API, the three cleanup workers, the four built-in resource managers and the MCP server (#168), and they are replaced together or not at all. **There are no per-service updates on the desktop.** (It included a notification service until #318; nothing had subscribed to it since #251.)

Not because per-service updates are impossible, but because the moment any two of the nine can move independently, the set of combinations we can be asked to support goes from one to a number nobody will test, and negotiation becomes load-bearing for a failure mode we invented ourselves.

The consequence is worth stating plainly, because it removes a whole row from #174's list of situations: **"the app updated and a built-in manager stayed old" is not a situation.** The only version skew a desktop install can have is against a **third-party** manager, which is not in the payload and never was - which is precisely the case negotiation exists for.

The files that enforce this are already the ones CLAUDE.md names together: `build-desktop.ps1` `$Services` and `backend.rs` `ServiceSpec` must list the same set, and the bundle is one artifact per platform.

### 7.2 A mixed install must be caught, not tolerated

Negotiation cannot catch an interrupted update, a partially restored backup or an antivirus quarantine, because the built-ins all declare the contract version they were compiled with and the disagreement is in code we assumed was uniform.

- Every spawned service reports a **build stamp** - the product version plus the commit - on the endpoint the facade already polls: `/health/ready` for the API (`HealthEndpoints.cs`), `/worker/status` for the three workers (`WorkerControlEndpoints.cs`), and whichever of the two each of the four managers and MCP ends up using.
- `wait_until_ready` (`backend.rs:320`) compares each stamp against the facade's own. A disagreement is fatal and explicit: the app does not boot into a half-broken state, it names which service came from which build and offers a reinstall.
- The cost is one string comparison per service on a response the facade already fetches. The alternative is an install that fails somewhere in the middle of a scan, hours later, with a message about something unrelated.

### 7.3 The manifest completeness gate

This failure has already happened once: `v0.2.0` and `v0.2.1` shipped without a `latest.json`, and the in-app updater answered "could not fetch a valid release JSON" - the whole story is in `auto-update.md`.

Today `make-latest-json.ps1:138` *warns* when a platform's `.sig` is absent and uploads the incomplete manifest anyway. Mid-race on Codemagic that is correct - the last platform to finish completes the manifest. From a deterministic caller it is not: it means that platform silently stops receiving updates, which from the user's side is indistinguishable from "no update available".

Policy:

- The deterministic callers - the GitHub Actions `release` job (`build-desktop.yml`) and the manual `desktop-latest-json` backstop - **fail** on an incomplete manifest. A `-RequireAllPlatforms` switch on the shared generator, passed by those two callers only, leaves the racing path untouched.
- Publishing the draft is the moment an update ships, and it is a human step, so the release checklist carries one line: `latest.json` lists every platform.

### 7.4 Downgrade is not supported, and must fail loudly

The updater only moves forward, but a user can install an older build over a newer one, and the per-user data folder survives the reinstall along with a database migrated by newer code.

- Migrations stay forward-only. No down migrations, for either provider.
- On startup, **every service that opens the database** compares the applied migration set against the migrations it knows. An applied migration it has never heard of means the data is newer than the code: refuse to start, and say so, rather than run a model against a schema it does not describe.
- "Every service" is the operative word, and it is why this is a shared guard rather than an API concern. In App mode the three cleanup workers share the API's SQLite file (CLAUDE.md), so a guard that only the API runs produces the worse half of the outcome: the API refuses to start as intended, and an older embedding or job cleanup worker starts happily and begins deleting rows in a schema it does not describe. Silent, and it writes. The check belongs next to the connection interceptor every service already shares, so a new service cannot forget it.
- The message names the actual remedy - install the newer version again - and never suggests deleting the database. The user data folder is deliberately outside the install directory (`user-data.md`) precisely so that a reinstall is not a data loss event, and an error message must not undo that.
- The build stamp of 7.2 does not cover this. It compares services against each other, so ten services all rolled back together agree perfectly and say nothing about the database they are about to open.

### 7.5 What the payload does not contain

- **Third-party managers.** Registered by URL, updated by whoever runs them. A desktop update never touches them and must never claim to - the same honesty #171 requires of the data-wipe wording.
- **An externally launched stdio MCP process** (#168). Claude Desktop holds a path and starts the binary itself. It is not ours to update or to kill. Two requirements follow: the stdio entry point keeps a **stable path across updates**, and because it reaches the engine through the core API like any other client, it is covered by core API negotiation rather than by the one-payload rule.

### 7.6 The upgrade test

The acceptance criterion in #174 asks for "an upgrade test from the previous release", and it is the only test that catches this whole section, because none of it is reachable from a clean install.

The shape: CI installs the **previous published release**, runs it once so the data folder exists and the database migrates, applies the new build **over it**, and asserts

1. every service reports the new build stamp,
2. the database migrated with no data loss,
3. settings and the per-user folder survived,
4. one search runs end to end.

The honest limitation: only Windows can run the NSIS installer unattended over an existing install in CI, so Windows is the platform where this is a real upgrade. macOS and Linux get the same assertions from an install-over-install rather than through the updater plugin, which does not exercise signature verification or the payload swap. That gap is the weakest link in this document and should be named in #168 rather than discovered later.

---

## 8. Web: a mixed set is legal, and must be visible

A rolling upgrade produces a mixed set for at least a few minutes, so it has to be supported rather than merely survived.

- **One source for every first-party service.** What holds today is weaker than a version tag and worth stating exactly, because the stronger version is not available: `docker-compose.yml` has no `image:` key for any PDM service - each is `build: {context, dockerfile}`, the only tagged image in the file is `postgres:16-alpine`, and nothing in `build-and-publish.yml` pushes anywhere. There is no registry, so there is nothing to pin. The invariant that does hold is that one checkout builds every first-party service, so a mixed set requires an operator to bring services up from two checkouts - deliberate, if undetectable from the compose file alone.
- **A version tag needs a registry first, and there still is none.** #154 added four more services to the compose file and every one of them is `build: {context, dockerfile}` like the rest, so a deployment still builds from a checkout rather than pulling a tag. Once first-party images are published, every one of them carries the same `${PDM_VERSION}` and the compose file references it once, so a mixed set means editing one line into several. Until then the reporting below is the only thing standing between an operator and a set they did not intend.
- **Upgrade order: the core first.** Inside a major both orders are safe, because additive-only cuts both ways. Core-first is still the rule, for one reason: a new core with an older manager is what every desktop install runs the day after an app update with a third-party manager registered, so it is the combination with by far the most coverage. Old core with a new manager is the one we would be testing least.
- **Across a major, order stops being a preference.** The core is the side that speaks both majors during the overlap, so a manager that has moved to `/v2` is unreachable until the core is upgraded. Core first, always.
- **The set must be readable.** Health aggregation (#175) reports each service's product version and the contract minor in force, and the admin panel shows it (#153). An operator who has been running a mixed set for a month should discover that from a screen, not from a log.
- **Absent is not outdated.** A deployment legitimately runs only the managers it wants: since #154 each of the four is behind its own compose profile and none is up by default. Reporting must distinguish "not deployed" from "deployed and behind", the same distinction `history.md` section 4.6 draws between down and gone.

---

## 9. Version reporting

| Surface | What it shows | Lands in |
| --- | --- | --- |
| Admin panel, per manager | Declared contract minor, negotiated minor, product version when the manager reports one, health, origin, and a status: current / behind / behind and losing something / unsupported, with the reason and any date | #153 |
| Admin panel, the core | Majors served, current minor, oldest supported minor, the floor if it has been raised, and pending removals | #153 |
| `GET /v1/resource-managers` | The **negotiated** `contractVersion` only. A caller has no business seeing what a manager declared, only what is in force | already in the contract |
| `GET /v1/capabilities` | `coreVersion`, `oldestSupportedVersion` and `deprecations[]`, so a client can display the window instead of discovering it as a 400 | #177 |
| Every service | A build stamp on the endpoint the facade already polls (7.2) | #168 |
| Each mounted internal frontend | The `@pdm/frontend-shared` version it bundled, against the host's, and what is negotiated between them | #247 exposes it; #249 decides what the host does with it |
| The bug report bundle | The whole table above, because the first question on any report is which versions were involved | #175, #116 |

`oldestSupportedVersion` is new in this document and has been added to [`core-api.v1.yaml`](core-api.v1.yaml). It is folded into 1.0 rather than introduced as 1.1, because nothing is implemented yet and 1.0 is not published to anyone.

---

## 10. Built-ins are not a special case

Built-in managers ship with the app, so their contract version is always ours and the negotiated minor is always the current one. That makes the entire degraded path dead code in the only configuration we run by default - until a third party finds the bug in it.

Two rules follow, and they are the ones most likely to be quietly skipped:

1. **The same negotiation code runs for built-ins.** No compile-time shortcut, no "internal" flag that skips the manifest read, no assumption anywhere that `contractVersion` equals the core's.
2. **A rehearsal in CI.** The fake resource manager (#173) is scriptable to declare any minor. The suite runs the full flow at the current minor, at the support floor, and one below the floor expecting a clean refusal with the right error and the right message. The conformance suite (#151) runs the four built-ins at the current minor.

Once a second minor exists, one built-in in the desktop upgrade test (7.6) *declares* the previous minor for the duration of the test. It is still the same binary from the same payload - only the declared version is overridden - so 7.1 holds, and it is the cheapest way to keep the degraded path alive in the configuration users actually run.

---

## 11. The neighbouring version lines

Not contracts, but they get asked about in the same breath.

| Line | Policy |
| --- | --- |
| Export document format (#143) | The importer reads every older form, with no expiry. An exported result is a file a user keeps for years, and there is no negotiation to fall back on - so the compatibility burden is entirely on the importer, permanently. |
| Database schema | Forward only, two providers, one migration each (CLAUDE.md). The startup check in 7.4 is what turns a downgrade into a message instead of corruption. |
| MCP protocol | Owned by MCP and negotiated by its own handshake. Not ours, and not to be conflated with the core API version the MCP server also speaks (#140). |
| The Tauri updater manifest | Owned by the plugin. Our only stake is that `make-latest-json.ps1` remains the single place its format lives. |
| The minisign signing key | Not a version, but the same class of irreversible: rotating it strands every installed app. `auto-update.md` already says so; it belongs on this list so nobody rediscovers it. |

---

## 12. Where this document differs from the issue

- #174 asks "how many minors back". The answer here is **all of them**, for the life of the major (3.1). A count would be a promise to delete fallbacks we keep anyway.
- #174's acceptance says a manager one minor behind "keeps working, with a visible notice". It keeps working; the notice is **conditional on something actually being lost** (5.4). An always-on version banner is not read on the day it matters.
- #174 frames the desktop risk as "a partial update leaving a new core with old managers must be impossible". This document makes that structurally impossible for built-ins by refusing per-service updates (7.1), and then spends its effort on the risk that remains - which is not partial updating but a **corrupted install** (7.2). Different failure, different fix.

---

## 13. Open points

- **Twelve months is calibrated against nothing, and has now been waived three times.** There is no third-party manager, so the overlap in 3.2 is a guess and everything declared so far skipped it (#197, #274, #278). It is cheap to lengthen and expensive to shorten, which is the only reason to state it now rather than later. Revisit once one external manager exists - which is also the moment the waiver stops being available.
- **Lazy start interacts with the build stamp.** #168 is considering starting managers on first use. A manager started an hour into a session is checked an hour late, so a mixed install could pass startup and fail later - exactly what 7.2 is trying to prevent. A stamp file per bundle subdirectory, read at boot before anything is spawned, would catch it earlier. Decide with #168.
- **The floor has no example.** 3.3 assumes a security defect that cannot be fixed additively. None exists, so the 30-day notice may be wrong in either direction, and the mechanism is untested by anything but imagination.
- **Two majors per deployment, but per manager?** This document assumed throughout that one deployment could hold a `v2` registration and a `v1` registration at once, which the path prefix and the per-registration base URL should have allowed. Nothing ever tested it, and #254 has since removed the direction it was about: PDM dials no manager, so there is no per-registration base URL and no second document to be on two majors of. What is left of the question is the core API's own prefix (3.2), where the answer is that no deployment has ever served two majors of it either.
- **Product versioning itself is undecided.** The app is `0.1.0` and tags are numeric `X.Y.Z` with no stated meaning. Once contracts are public, someone will ask what a product major means. Out of scope here, and worth its own decision before `1.0`.
