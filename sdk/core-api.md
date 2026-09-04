# Core API

The written contract behind [#166](https://github.com/Iskandarus/photo-duplicates-manager/issues/166), part of the platform epic [#139](https://github.com/Iskandarus/photo-duplicates-manager/issues/139), and reversed in direction by [#254](https://github.com/Iskandarus/photo-duplicates-manager/issues/254).

**Served by `PDM.API` under `/core/v1` since [#177](https://github.com/Iskandarus/photo-duplicates-manager/issues/177).** The code lives in `asp-back/PDM.API/CoreApi/`.

**The document is 2.0 and this build serves 2.0.** [#274](https://github.com/Iskandarus/photo-duplicates-manager/issues/274) wrote the major on paper first, so that the four pieces which implement it - #275 the ingest path, #276 the managers pushing, #277 the reads moving to the manager, #278 the retirement of the pull side - implemented something that was decided rather than deciding as they went. That gap is closed: #278 removed the gateway, so no deployment can dial a resource manager, and a build that could would have been lying about the half a client branches on. **Section 18 is still the one place that says what a caller actually talks to**, and `CoreApiContractDriftTests` now pins the agreement rather than the pair.

Machine-readable companion:

- [`core-api.v1.yaml`](core-api.v1.yaml) - OpenAPI 3.1, what PDM serves.

Checkable without any PDM code:

```bash
npx @redocly/cli lint --config asp-back/contracts/redocly.yaml asp-back/contracts/core-api.v1.yaml
```

The document validates clean against 3.1 apart from a missing `info.license`, which stays open until the repository has one to point at - the same single warning the resource manager contract carries, and the one rule `redocly.yaml` turns off.

**The lint runs in CI** (#151, `.github/workflows/contracts.yml`), together with a check that this document's `info.version` and the entries for the Core API line in [`contracts/CHANGELOG.md`](CHANGELOG.md) agree. There is no conformance suite against this API in the sense there is one against a resource manager, and there should not be: this is the side PDM serves, so it is covered by `PDM.API.Tests`. Whether a suite that plays PDM and checks a *caller's* use of this API is worth building is open, and belongs with the retirement of the one that checks the other direction (#278).

---

## 1. What this is

`/tasks/*`, `/files/*`, `/export/*` and `/import/*` were the first-party frontend's private API: undocumented, unversioned, and authorised with the user's own JWT, which was fine while the only caller was code we shipped in the same bundle.

Once a resource manager owns the whole experience for its resource, it has to drive the engine, and the caller stops being ours. This document is what those routes became: a scoped public API that untrusted UI can be allowed to call.

**They are gone (#251), and this is the only surface now.** Each internal manager draws its own screens (#248, #250) and its own backend calls these operations; the private routes lost their last caller with the app host's working area, and what is left under `/tasks` is admin diagnostics. See `history.md`.

The rule underneath used to be symmetrical. A resource manager owns photos, their metadata, and the experience of working with them; what PDM computed about those photos is PDM's. Every operation here read or changed PDM's own data, and the ones that touched a file did it by asking the resource manager.

**#254 removed the second half of that sentence, and 2.0 is where it leaves the document.** The core only answers: a request arrives, an answer goes back, and PDM never initiates and never reaches outward. So there is no longer anything PDM can ask a manager, which changes this API in two directions at once. It **gains** the operations a manager needs in order to push - declare what its walk found, send the bytes PDM asks for, say the walk is over, report that a reference is gone - and it **loses** every operation that only made sense while PDM could ask: image bytes and previews, recycle, delete and move.

What is left is exactly the part nobody else can do. PDM identifies an image, embeds it, groups it with others, and hands the grouping back in the manager's own references. Every readable thing about a photograph - its name, where it sits, what it looks like, whether it still exists - belongs to the manager and is answered by the manager, to a frontend that is also the manager's (#217, #250).

---

## 2. One contract, one direction

There were two, and the second is withdrawn (#278).

| What a manager's backend does | Where |
| --- | --- |
| asks: start / cancel / resume a search, follow progress, page results, hide and unhide a group, delete a result set, export and import | **this document, #166** |
| feeds: declare enumerated items, upload the bytes PDM asks for, complete the walk, report a reference gone | **this document, sections 9 and 10**, #274 |
| says: four narrow events, relayed to the user's notification channel | **this document, section 19**, #161 |
| is told: progress, endings, and how many images PDM will take next | **this document, section 20**, #246 and #274 |

Four verbs, one document, one direction of dialling. An implementer opens one address and needs no address of its own - no hostname, no certificate, no hole in a firewall - which was the single largest deployment requirement #139 imposed and the one #254 exists to remove.

The clearest example of what stayed on PDM's side is hiding a group. It flips `IsHidden` on `DuplicateGroupEntity` and recomputes the aggregates (HideGroupCommandHandler.cs:61). It never touches a file, and a resource manager has nowhere to keep that flag. So it is an operation in this document, called by their button. The clearest example of what left is deleting a photograph: the manager owns the file, deletes it under its own authority, and tells PDM afterwards - PDM could neither perform that deletion nor verify one, and asking it to relay the request was a ritual rather than a control.

---

## 3. Two consumers

1. **A resource manager's backend.** Four are built in this repository and none of them is special: each holds the client credentials its registration was issued (#145), exchanges them for an app token, and is the one correspondent its own frontend has (#217, #250). A third party writes exactly this and nothing else - there is no interface left for it to implement.
2. **The MCP server** (#140), a standalone project that reaches the engine only through this API - no project reference on `PDM.API`, no in-process path.

Two independent consumers is still the reason to treat this as a product rather than a rename of the internal routes. If an operation is awkward for either of them it is wrong for both. `GET /v1/resource-managers` exists because of that rule: a manager's own backend already knows which registration it is, and the MCP server has no other way to offer "pick a source".

**A third consumer was designed for and never arrived**, and this is where that is written down rather than left as a puzzle in the history. It was a workspace running cross-origin in a frame, with its calls brokered by a first-party shell over a postMessage protocol. The protocol was retired at 1.0 without ever being implemented (#197), the shell was never built (#152, closed), the app host's working area went with #251, and 2.0 removes the last four operations that existed only for it: two token kinds and two image byte routes (#253). Sections 9, 10 and 18 say what each one was for, because "why is this here" is a question somebody will otherwise answer by guessing.

---

## 4. Scoping - the hard part

The caller is a program somebody else wrote, holding credentials an administrator issued. It must never be able to reach beyond its own resource.

### The enforcement chain

Every call passes the same four checks, in this order, and each one has its own error code so a client can tell them apart:

1. **Authentication.** A usable token. Otherwise `unauthenticated`.
2. **Scope.** The registration was granted this operation. Otherwise `insufficient_scope`.
3. **Permission.** The user behind the call may perform it, evaluated against the existing `PermissionMatrix` (Permission.cs). Otherwise `insufficient_permission`.
4. **Ownership.** The object belongs to this (user, registration). Otherwise `forbidden`.

Ownership is a predicate in the query, not a filter applied afterwards. A result set is fetched `WHERE userId = @user AND resourceManagerId = @registration`, so there is no code path that loads somebody else's row and then decides not to return it. That distinction matters because the failure mode of the second shape is silent: a new query written next year simply forgets the filter, and nothing fails until someone notices their photos in another tab.

This is the single enforcement point for the grant model in #145. Authority for a call is `granted scopes AND the user's permissions`. The manifest is a claim, never an authorisation. Evaluated here and nowhere else, so there is one place to review and one place to get wrong.

### One credential, and it is never the user's

**No ambient authority.** No operation in this document accepts the user's JWT or refresh cookie, and 2.0 leaves exactly one token class.

| Token | Minted by | Held by | Scoped to |
| --- | --- | --- | --- |
| app | `POST /v1/tokens/app`, with the app id and secret from the registration | a resource manager's backend, the MCP server | (user, registration) |

1.x had three. A **capability** token was minted with the user's own session and attached by a first-party shell to calls it brokered for a frame; a **media** token was the narrow, read-only thing that frame itself held so it could put real URLs in `img src`. Both were sound, and neither ever had a caller: there is no shell, and a manager's frontend has one correspondent, its own backend (#217). #253 kept them with a stated trigger - "re-read when #160 ships or at the next major of this API, whichever comes first" - and 2.0 is that trigger firing. What replaced them is not a smaller credential but a different topology: the browser talks to the manager, the manager talks to PDM, and nothing in the browser ever holds anything of PDM's.

Minting cannot be chained. An app token cannot mint another token of any kind - a token that mints tokens is privilege escalation with extra steps - so authority enters this system exactly once, at an exchange that requires a secret no browser can hold.

Revocation is immediate, not eventual (#145). Disabling a registration or changing a grant moves the grant stamp, which voids every token in flight rather than leaving them to expire; in-flight calls carrying one are rejected with `unauthenticated`, and the caller's next exchange either succeeds under the new grant or answers `forbidden`. Both are true statements a backend can act on, which is why 2.0 does not need the `capability_expired` / `capability_revoked` split: that distinction mattered to a shell holding a user's session open, and there is no such holder.

### Ceiling, not floor

A grant can only narrow. A resource manager granted delete still cannot delete for a read-only user.

| Scope | Operations | Narrowed by |
| --- | --- | --- |
| `searches:read` | list and read searches | `SessionStatusRead` |
| `searches:write` | start, cancel, resume | `SearchStart`, `SearchCancel` |
| `searches:ingest` | declare items, upload content, complete a walk | `SearchStart` |
| `results:read` | list and read results and their groups | `SearchResultRead` |
| `results:delete` | delete a stored result | `SearchResultDelete` |
| `groups:write` | hide and unhide | `GroupHide`, `GroupUnhide` |
| `images:forget` | report that a reference is gone | none |
| `export:read` | export a result | `ExportRead` |
| `import:write` | validate and import | `ImportValidate`, `ImportCreate` |
| `stream:subscribe` | hold the answer channel open | `SessionStatusRead` |

Two are new at 2.0 and four went with the operations they authorised (`images:read`, `files:recycle`, `files:delete`, `files:move`). A fifth, `events:publish`, went at 3.0 with `POST /v1/events` (#318) - see section 19.

**`searches:ingest` is deliberately not part of `searches:write`.** A caller that may start a search is not automatically one that may feed it: starting is a request for work, feeding is a stream of somebody's photographs into this deployment's memory, and an administrator should be able to grant one without the other. They share a *permission*, `SearchStart`, because as far as the user is concerned they are halves of one act - somebody who may not start a scan has nothing to feed - and because a revocation that stops one mid-scan ought to stop the other.

**`images:forget` is narrowed by no permission, and that is the interesting one.** It reports a deletion that has already happened, on the manager's own resource, under the manager's own authority. Narrowing it by `FileDelete` would mean a read-only user's stored result goes on naming files that no longer exist, permanently, with every later page showing tiles that cannot open. That protects nothing and produces a wrong answer, which is the test a permission has to pass. The grant is still a ceiling: a registration nobody granted it to cannot report anything at all.

`GET /v1/capabilities` returns the intersection already computed, so a client that hides everything absent from that list never presents an action that 403s. That is not a convenience: a UI which shows a delete button to a read-only user has already failed, whatever the API does afterwards.

### `SearchStartLocal` needed a new anchor, and then needed nothing

One permission in that matrix did not survive the move unchanged, and it is worth keeping the history because the shape it moved through is the shape a future one will.

`Permission.SearchStartLocal` was admin-only and was checked by comparing the request's resource manager against the local one. In Web mode that is what stopped an ordinary user from scanning the server's own filesystem. #143 made the comparison a string one, and after #149 `local` is just another registration - at which point comparing against the string `local` would be trivially defeated by registering a second manager that also read the host disk. So the gate moved onto the registration: an administrator marked a registration as reading the host's own filesystem, and `POST /v1/searches` against such a registration additionally required the permission.

**#254 removed all of it, because it removed the disk.** Nothing in this process opens a photograph on any path, and the only program that reads a local one is `PDM.ResourceManager.Local` - which is ours, reads its own machine rather than PDM's, and is not built for a server at all. There is no server scan for the permission to gate, so the permission is gone, and so are `CloudLocalBrowse` and `CloudLocalDrives` beside it.

One consequence is worth stating plainly rather than discovering: **no core API scope is now narrowed by a permission an ordinary user lacks.** That was the only one, so the floor the enforcement chain still checks separates an authenticated caller from an anonymous one and nothing finer. The mechanism is unchanged - `PermissionsFor` and the matrix are still consulted per scope - so a future permission can separate the roles again the day there is something to separate them over.

### Cross-registration reads answer 403

A resource manager asking for another manager's result gets `forbidden`, and that is covered by a test. It is a deliberate trade: 403 discloses that an id exists, where 404 would not.

Two things make it the right one here. Ids are unguessable by construction (see section 5), so what leaks is bounded to "this id you already somehow have belongs to someone else". And a distinguishable answer is what makes the isolation testable: a 404 hides a scoping bug behind the same response as a typo, and the acceptance criterion for this issue is precisely that the isolation can be asserted.

### Rate limiting

Per registration and per user, both. Per registration alone lets one user exhaust a manager's whole allowance for everybody; per user alone lets a chatty caller starve the rest of the app. `GET /v1/capabilities` reports the limit so a client can pace itself instead of discovering it as 429s.

Ingest has a second, separate mechanism on top, and the two are not the same thing. A rate limit is about how often a caller may call; a **credit** is about how much of somebody's photograph library PDM is willing to be holding at once (section 9). A caller can be well within its rate limit and still have no credit, and the remedies differ - wait `Retry-After` seconds, or wait to be told there is room - which is why they answer with different codes.

---

## 5. Identifiers

### PDM's ids are opaque and unguessable

`searchId`, `resultId`, `groupId`, `imageId` are minted by PDM, stable, and safe in a path segment.

Unguessable is a requirement, not an observation. `DuplicateSearchResultEntity.Id` and `JobRecordEntity.Id` are sequential integer primary keys, and a sequential public id turns the 403-versus-404 decision above into a way to enumerate other people's work by counting. The public id is therefore its own column - random, indexed, unique - and not the primary key. That is two migrations, PostgreSQL and SQLite, and it belongs with the registry work rather than being discovered during #152.

`groupId` and `imageId` already exist in the right shape: `DuplicateGroupEntity.GroupId` and `DuplicateGroupImageEntity.ImageId` are GUID strings.

### Images are read by `imageId`, and a reference is input in exactly two places

The 1.0 rule was absolute: `imageRef` was output only. The reason it was absolute is worth keeping, because 2.0 relaxes it in a way that does not touch that reason at all.

The rule existed because **an `imageRef` addresses the whole resource, not the result**. A reference is minted by the manager for any item it can see, so accepting one on a *read* would have turned PDM into a proxy for that manager's entire library, with PDM's credentials and PDM's caching in front of it. An `imageId` exists only inside a result this caller owns, so it cannot address anything else and scoping needs no extra check. A second reason went with it: a move that changed a reference did not invalidate a caller's list of selections, because the `imageId` was unchanged.

2.0 takes an `imageRef` as input on two operations, and neither of them is a read of anything.

| Operation | Why a reference is the right key |
| --- | --- |
| `PUT /v1/searches/{searchId}/items/{imageRef}/content` | the caller is *supplying* the image, and PDM has no id for it yet - the id is minted from what arrives |
| `POST /v1/images/gone` | the caller is the manager whose reference it is, reporting on its own resource, and it holds no `imageId` for a file it has just deleted |

In both, the reference names something the caller already owns and is already authorised for, and PDM is not being asked to *fetch* anything. The proxy risk the original rule guarded against needs a read to exist, and there is no longer any read of image bytes in this document at all (section 10).

`imageRef` is still on every `ResultImage`, and at 2.0 it is **required** rather than optional: with `name`, `displayPath`, `available` and `trashed` gone, it is the only field a caller can join this row to its own item with.

---

## 6. Paths and verbs

Paths are `/v1/...` under the deployment's core API root. The major version is in the path; the minor is negotiated per request.

**No `:` and no `.` in any path.** It was the brokered channel's grammar: a guest-supplied path was constrained to `^(/[A-Za-z0-9_~-]+)+$`, with dots excluded from the class rather than filtered afterwards, so a dot segment could not be expressed at all. The broker is gone and the rule stays - every path here was written to it, a segment reads better than a verb suffix in a router and in a log, and there is no benefit to having two spellings. So `/v1/images/gone`, not `/v1/images:gone`, and the `resource:verb` style the resource manager contract used is still not used here.

One reference does appear in a path segment, on the upload. Two things follow and both are in the document rather than left to habit: the caller percent-encodes it, and `.` and `..` stopped being valid references at 2.0. That is a tightening, which only a major may do, and it is one rule in one place instead of a normalisation step every implementation has to remember.

Everything else follows from being callable by code we did not write:

- `PUT` for the image bytes. The target is named by the URL and a second send of the same image is the same state, so a retry needs no idempotency key and no negotiation: the reference is the key, and it is one the caller already holds.
- `POST` where something is reported rather than replaced. `POST /v1/images/gone` carries a list in a body, and a body on `DELETE` is dropped or rejected by enough proxies and clients that it is not something to hand a third party.
- `PATCH /v1/groups/{groupId}` with `{"hidden": true|false}` instead of `hide` and `unhide`. The state is set, not toggled, so a retried call is a no-op; and it is one scope rather than two that nobody would grant separately.
- Groups are addressed flat, listed nested. `GET /v1/results/{resultId}/groups` for the collection, `PATCH /v1/groups/{groupId}` for the item. A nested item path would carry a second key that can disagree with the first, and the ownership check would then have to pick one to believe.

---

## 7. Searches

### A search has an identity

`POST /tasks/search/duplicates/cancel` cancels "the user's running task". `POST .../resume` resumes "the user's most recent resumable job". Neither takes an identifier, because the current model has one active search per user, keyed by user id in `TaskCoordinator`.

That model cannot survive this epic. With several resource managers, a per-user singleton is a channel between them: manager A cancels manager B's search, and B watches progress it did not start. So searches get ids, `GET /v1/searches` is scoped to the caller's own registration, and `search_already_running` is reported per (user, resource manager) rather than per user.

The engine now agrees. Since #163 a running search is tracked under a `SearchScope` - the user and the resource manager it scans - rather than under the user alone, so "already running" is answered about the search that would actually collide, cancelling one search leaves another manager's alone, and withdrawing a grant stops that manager's work and nothing else. The private `/tasks/*` routes named no search and meant "whatever this user is running", which is a set rather than a single thing; they acted on all of it, and they are gone (#251).

Whether the engine *should* run two searches at once - model leases, memory, disk - is a separate question and belongs to #168. What changed is that the model no longer forbids it.

`POST /tasks/search/session/start` has no successor. It creates a `DuplicateSession` through the domain service and has no caller: nothing in the frontend requests it. Sessions are core bookkeeping that the cleanup workers key off, and starting a search creates one; a public operation that mints one out of band would be an orphan in the API too.

### Engine tuning is not a search parameter

`DuplicateSearchRequest` carries `NTrees` and `K`, each an index-build parameter, which is to say a denial-of-service knob in an untrusted caller's hands. They stay out of the API.

It carried `CacheConcurrency` beside them until #326, and that one is worth a sentence because the reasoning around it was half wrong. Keeping it out of the API was right; the claim that it came "from a server-config snapshot taken per job" was not, because no configuration key carried it and the snapshot never recorded it. Nothing set it, so what every deployment ran was the `const int` of 4 it fell back to. The field is gone, and how many images PDM embeds at once is derived from the machine with `BatchProcessing:EmbeddingConcurrency` able to lower it - see embedding-concurrency.md.

`threshold` is the one search parameter a user actually chooses, so it is the one the API accepts.

**It is a cosine similarity, not a distance** (#298). The engine calls two images a pair when `similarity >= threshold`, and it always has; the document described the same field as "distance below which two images are considered duplicates" from #166 until 2.0, with a default of `0.03`. Nothing implemented that description, so it cost nothing for as long as the only caller was the app host's own start page, which sent `0.97` of its own. When the app host stopped starting scans (#251) and each manager's frontend started (#248, #250), that frontend implemented the field as written and sent one to twenty percent - and every scan came back as one group holding every photograph, because `0.03` is below the similarity of any two photographs whatsoever. The lesson is narrower than "read the document": a range check is not a unit check, and `0 <= x <= 1` was the only thing four separate gates asked. What catches it now is a grouping test with two vectors of a known similarity, which fails if the comparison and the default ever mean opposite things again.

Two parameters left at 2.0 for a different reason: they were instructions to a walk PDM no longer performs. `recursive` asked how deep to go, which is now the caller's own decision and needs nobody's permission. `cacheFolderRef` asked PDM to compute embeddings over a wider folder than the one being searched, so that a later search inside it would be nearly free - a good feature, expressed in units PDM does not have. **PDM has no concept of a directory and never had** (#163): handed a stream of items, it could not tell which were inside the narrower folder. The caller can, so the same feature is `cacheOnly` on a declared item - embed it and keep the embedding, leave it out of this result.

### The account a scan runs against (1.4)

Resource manager contract 1.2 let one `userRef` hold several accounts at one manager, and #229 built the whole boundary for it: the `accountRef` is the manager's own opaque value, minted there, never an account id at the upstream service. What was missing on this side is the half a person actually sees. A scan started against "the default account" is right until somebody links a second one, and a saved result that cannot say which account it belongs to leaves `connectionState: differentAccount` as a fact with nothing behind it.

So `accountRef` is optional on `StartSearchRequest` and reported on `Search` and on `Result`. Three things about it are worth stating plainly, because each of them is a decision:

- **PDM does not know what an account is, and this does not change that.** It never mints one, never parses one and never compares one to anything except itself. The value is kept on the job, copied onto the result, and handed straight back to the manager on every later call about those images. That is the whole of its life here.
- **Absent means the default account**, which is the meaning every call had before there was a field - the account the user connected first, the one a 1.1 call already acted on. That is what makes this a minor rather than a behaviour change.
- **A ref belonging to nobody is refused, not replaced.** The manager is the only side that can tell, and it answers `forbidden` rather than falling back to the default, because a wrong ref that quietly became the default is a scan of the wrong drive with nothing in any log saying so.

A frontend that offers the choice is the manager's own (#250): the account list, the connect flow and the disconnect never touch this API at all, and the only thing that crosses is the ref.

### Progress with a denominator, and where it comes from

A denominator comes from one of two places and never from a guess. **The caller counts its own resource and says so** with `expectedTotal` on a declaration (2.3, #227), or **the walk ends** and what arrived is the total. Where there is neither - a caller that sends no count, which is every 2.2 client - `totalItems` is null, `percent` is absent, and a client shows "found N so far, processed M" exactly as before.

**PDM cannot count and this did not change that.** It receives one batch at a time and never materialises a folder (#163), so counting here would be the caller's walk performed a second time, at PDM's expense, over the wire. The number is the caller's because the caller is the only party that can produce it.

Three rules govern it, and all three exist because a count is a claim rather than a fact:

- **A converging total is a total.** `totalItems` is present from the moment a count arrives and `percent` with it, while `isTotalKnown` keeps the narrower meaning it always had - the walk is over, this will not move again. A client that needs to say whether a number is settled reads the second, not the presence of the first.
- **Being wrong is not an error.** Nothing loops until the count is reached and no search completes because of it; only `complete` ends a walk. Send a corrected number whenever you have one - it may move in either direction.
- **It is never reported below what has arrived.** A count that turns out low is raised to what has actually been declared, because a bar that reaches 100 percent and keeps going is worse than one that arrives late.

Count **only what you would declare**. An item your walk will filter out - a format `GET /v1/capabilities` does not list, a file that is not a photograph - is one that will not be in the result, so counting it promises a person photographs that never arrive. The number at the end of the walk has to be the number actually processed.

Beside it, `itemsPerSecond`: how many images a second are being processed right now. It is derived from PDM's own readings and smoothed over a short window, present-and-null where there is not enough history to divide by, and **zero where nothing is moving** - which is a real answer, not a missing one, and is what lets a client say "not moving yet" instead of showing a number that has stopped.

`percent` is **the whole search and not the running stage**. That is a correction rather than a choice: it used to be `processedItems` over `totalItems`, and those are whichever stage's own counters, so the bar filled to the end during embedding and started again from nothing when comparing began.

What ends the walk moved with the direction and nothing else about this changed. In 1.x PDM pulled batches and read `isLast` off one of them; at 2.0 the caller calls `POST /v1/searches/{searchId}/complete`. Both are an explicit end of stream and neither is "the items stopped arriving", for the same reason: PDM cannot tell a finished walk from a slow one, and guessing either way is wrong. A caller that dies mid-walk leaves the search to be found `interrupted` by the cleanup that already exists, which is the honest outcome; a caller that knows it cannot finish says so with `outcome: "failed"` and the search fails with its detail.

`stage` carries a stable machine `key` plus an English `label`, and so does the live progress the stream pushes - both come from `StageNames`, so a caller following a search over the channel and one polling this API localise it the same way. The label is display text PDM never translates; the key is what a client branches on. Minors may add stage keys, so a client renders an unknown key from `label` rather than failing. Any new stage still has to be added to `PDM.Application/Dictionaries/StageNames.cs` and shown as its own stage on the frontend.

`enumerating` arrived with the pull-based walk. It is short-lived by design: enumeration overlaps embedding, so the stage moves on as soon as the first batch lands and the walk carries on underneath - visible as a `discoveredItems` that keeps climbing rather than as a stage that stays put. Two keys stopped being reported by a search at the same time: `fingerprinting` and `computing_hashes` now run once per batch, interleaved with embedding, so they are steps inside `extracting_embeddings` rather than phases of their own. Both keys remain part of the vocabulary - they were never removed, and a client that has strings for them keeps them.

---

## 8. Results and groups

### One collection, one filter

`GET /tasks/search/result` and `GET /tasks/search/hidden` are the same query with `IsHidden` flipped, shipped as two routes with two shapes. They become `GET /v1/results/{resultId}/groups?visibility=visible|hidden|all`. Two routes over one predicate is how the two drift, and it is the same failure that put the resource manager list in both `ProvidersEndpoints` and `FeaturesEndpoints`.

Reading hidden groups needs `results:read` and not a scope of its own, for the same reason: they are the same rows, and a separate scope would imply a separation the data does not have.

### A group can be split across pages

Paging counts images, not groups, so the tail of a large group continues on the next page. Such a chunk carries `isPartial: true`, and `totalSize` holds the size of the whole group while `size` is what is on this page.

This is existing behaviour (`DuplicateGroupDto.IsPartial`, `TotalSize`) that the private API never had to explain, because the only client was written alongside it. A third party rendering `images.length` as the group size gets it wrong on exactly the groups that matter most, so it is stated in the contract rather than left to be discovered.

### Duplicates while the scan is still running (2.1)

`GET /v1/searches/{searchId}/groups` answers with the duplicate groups a **running** search has formed so far, in the same paged shape a saved result's page has. It is [#295](https://github.com/Iskandarus/photo-duplicates-manager/issues/295), and it restores something rather than adding it: duplicates used to appear on the results page while the scan was still running, and that stopped working when the screen that renders them moved to the resource manager's own origin - PDM kept computing a snapshot roughly once a second and kept broadcasting it on its own notification hub, which that screen cannot subscribe to. (That hub is gone at 3.0, and for the same reason one issue larger: #318.)

Four things a caller has to know before it draws one.

**It is a snapshot, not an accumulation.** The grouping stage recomputes the connected components of every similar pair it has found, so each answer replaces its predecessor entirely. Re-reading is always safe and can never double-count, which is the same property that makes a progress reading safe to re-apply (#228) and a credit balance safe to lose (section 9).

**Its `groupId` is not a group's identifier.** A group gets one when the transaction that saves the result writes it, and no operation in this document accepts the value this answers with. What it is for is keying a rendering: it is stable across the snapshots of one search, so a cluster that persists keeps its identity and a page redraws instead of remounting every tile once a second.

**An empty page is an ordinary answer, and so is a late one.** A scan that has not reached its grouping stage and one that has passed it both have no live groups at all, and treating that as "this scan found nothing" is the one wrong reading. A search that has **ended** is a third case and does not behave like either: its last snapshot stays readable here for `SearchPreviewSnapshots.Retention` afterwards, which is deliberate and is the same choice `SearchProgressSnapshots.FinishedRetention` makes two paragraphs of section 7 away - a client that reads a second after a scan ends should see what was found rather than nothing, because "finished" and "nothing was running" look identical to a page with no answer and only one of them is true. It is not the authority: once `GET /v1/searches/{searchId}` reports a terminal status, read `GET /v1/results/{resultId}/groups`, which is persisted, pageable tomorrow, and the only one of the two that carries hiding.

**The push carries the counts and not the groups.** `search_groups` on `GET /v1/stream` (section 20) says how many groups and how many images there now are. A snapshot of a large library is thousands of references, that connection drops for a subscriber that has fallen behind rather than waiting for it, and the screen renders one page - so the message is the signal and this operation is the read, which is the same division `search_ended` and `GET /v1/searches/{searchId}` already have.

What is on a live group is a reference per image and nothing else: no `imageId`, because PDM has not minted one until the result is written, and nothing readable, because it keeps none of that for any image at all (see below, and section 10). The set a caller watches appear is the set the saved result contains plus whatever the last batch added - both are built from the same pairs, and the last snapshot is taken before the result is persisted rather than after.

### Nothing is re-checked on read, because there is nobody to ask

In 1.x, serving a page of groups asked the resource manager for batch metadata (`images:batchMetadata`) so that a photo deleted outside PDM since the scan came back `available: false` rather than as a broken tile - one batched call, not one per image, and `availabilityChecked: false` with every `available` null when the manager could not be reached.

PDM makes no such call at 2.0. So `available` and `trashed` are gone from `ResultImage`, `availabilityChecked` is gone from the page, and `name` and `displayPath` went with them for the same reason: **every one of the four was the manager's own fact, and PDM was fetching it on the caller's behalf from the caller's own manager.** The frontend that renders the tile belongs to that manager. It has all four locally, and it was already making the call that carries them.

Two things are worth stating so they are not rediscovered as regressions. A stored copy would have been worse than an absent one - a name goes stale the moment somebody renames the file, which is why #229 removed the stored name in the first place. And **a result whose images have all been deleted outside PDM no longer notices on its own**: it finds out when the manager reports the references gone (section 10), which is a push instead of a poll and is the only mechanism there is.

### The files the scan could not read (2.4)

A result says how many of them there were, and its first page of groups says which.

Two fields were declared for this from 1.0 and neither was ever filled: `totalBrokenImages` on a result was always `0`, and `brokenImages` on a page of groups was always absent. Underneath, the count existed and was thrown away - the pipeline has always caught an undecodable file, counted it and carried on, and the counter died with the run that made it. So a library holding files this deployment cannot decode scanned quickly and produced a result with **nothing visibly missing from it**, which is worse than an error: there is nothing for anybody to notice.

**Two kinds, because a person acts on them differently.** `errorCode` on an entry says which:

| Code | What it means | What somebody can do |
| --- | --- | --- |
| `unsupported_format` | No decoder in this deployment carries the file's format. | The photograph is whole wherever it lives. Convert it, or scan it from something that reads it. `GET /v1/capabilities` lists what is carried, as `formats`. |
| `unreadable_bytes` | The file announced a format that *is* decoded here, and no decoder would open it. | Nothing. The file is damaged, or it is not the image it says it is. |

The first is **one fact noticed at either of two moments**, and deliberately one code. A declared `mimeType` this deployment cannot decode is refused before any bytes move (section 9), and a file that declared nothing - or `application/octet-stream`, which is what a filesystem hands its manager for anything it does not recognise - is fetched, read and found to name no container PDM carries. It would be a poor API that called one thing two names depending on when it was noticed.

**A transfer that never arrived is neither.** That is a transport problem, the file at the source is probably fine, and it stays where it was: in `notices` on the search, as `content_never_arrived`, `upload_abandoned` or `content_mismatch`. Reporting it as a damaged photograph would send somebody looking for a file that is not broken.

**The counts are the whole scan and the list is bounded.** `totalBrokenImages` counts every file the scan could not read; `brokenImages` carries at most a couple of hundred of them. Fewer entries than the count is the bound and not a contradiction. The list exists so somebody can act on the files, and past a few hundred nobody acts file by file - they act on the pattern, which the two counts carry in full.

An entry is a `ResultImage` with almost nothing in it, which is honest rather than lazy: **nothing about the file was measured, because it did not open**. What is there is the manager's own `imageRef`
- so the frontend drawing the list can name the file itself, which is where every name has come from since 2.0 - and the verdict, as a code to branch on and an English sentence to fall back to. The `imageId` beside it is stable across reads and is accepted by no operation: there is nothing to hide, delete or open.

The list is on the **first page only**, because these belong to the result rather than to any group. A result of a thousand groups is many pages, and repeating the list on each of them would be the whole list answered again per screenful.

An **imported result reports zero**, having scanned nothing.

### Deleting a result is not deleting photos

`DELETE /v1/results/{resultId}` removes what PDM computed. Nothing reaches the resource manager, and at 2.0 nothing could: no operation in this document deletes a photograph. It keeps its own scope (`results:delete`) so that it stays separate from `images:forget` - one throws away a grouping, the other records that somebody else threw away a file, and a caller may hold either without the other.

---

## 9. Ingest: the walk is the caller's

Three operations, and they are the whole of what #254 reversed.

| Operation | What it is |
| --- | --- |
| `POST /v1/searches/{searchId}/items` | a batch of what the walk found; answered per item with `cached`, `needsContent` or `rejected` |
| `PUT /v1/searches/{searchId}/items/{imageRef}/content` | the bytes of one image |
| `POST /v1/searches/{searchId}/complete` | the walk is over, or could not finish |

`POST /v1/searches` still opens the search - it mints the `searchId` the three above name - and it no longer starts anything. In 1.x it answered 202 and PDM went and enumerated the folder itself.

The first operation also carries `expectedTotal` (2.3, #227): how many images the whole walk will find, as the caller counts them. It is the denominator of a progress dialog and PDM has no way of producing one - see *Progress with a denominator* in section 8. Send it as soon as you know it and on every batch afterwards; omitting it is what every 2.2 client does and costs nothing but the percentage.

### The answer to a declaration *is* the cache

The most important property of the first operation is that it is a question with a useful answer. A manager declares an image with a `contentHash` or an `etag`; PDM already keeps an embedding per content version (#230), so it can say `cached` and the item is finished - no credit, no upload, no further call. **A library nothing has changed in declares its contents and uploads nothing at all.**

That is not a new mechanism, it is the old one asked from the other side. The pull-based walk made exactly the same decision after it had received the metadata; the difference is who pays for the transfer when the answer is "I already have it", and in a push model the caller has to be told before it reads a file.

Two fields carry the version rather than one, and it is deliberate. A `contentHash` is a statement about content and survives a move, a copy and a re-upload; an `etag` is a version token that only means anything at one source. PDM records which kind of evidence it holds and matches like with like, never parses either, and never recomputes a hash. A manager that has neither is still correct: its images are asked for, hashed as they arrive, and the hash written back - so a second scan re-fetches but does not re-embed.

The issue that specified this sketched one `contentVersion` field. It became two on the way in, because the namespacing that keeps a hash from being read as an entity tag already exists inside PDM (#230) and collapsing the two on the wire would have moved that decision onto the caller, which knows less about it, and stranded every fingerprint row written before the reversal.

### One list of formats, published (2.2)

`GET /v1/capabilities` answers `formats`, with `mediaTypes` and `extensions`. It is what this deployment can turn into an embedding, and it exists because before #207 there was no such list - there were four, one per resource manager, each written by whoever built that manager.

| Manager | What it enumerated |
| --- | --- |
| local disk | jpg, jpeg, png, bmp, gif, webp |
| Dropbox, OneDrive | the same, plus heic, heif, tiff, tif |
| Google Drive | jpeg, png, gif, bmp, webp, heic - and no tiff |

So the same photograph was visible through one manager and invisible through another, and a user had no way to tell which they were getting. The split is not the interesting part; the question under it is. **A manager legitimately knows what its own service can serve** - Google Drive filters server-side by media type because that is the only filter its API offers - **but what PDM can decode is a fact about PDM**, and there is one pipeline doing the decoding. So the core publishes it and a manager filters against it.

Read it once per scan and filter your walk, in whichever spelling suits your source: `extensions` where you are walking names, `mediaTypes` where your service tells you a type. Match case-insensitively and **do not pin a copy** - the list grows in minors, and a manager holding 2.2's copy is one whose users cannot scan whatever a later minor adds.

There is an enforcement behind the advice, and it is deliberately narrow. A declared item whose `mimeType` names an image format absent from `mediaTypes` is `rejected` with `unsupported_format`, before any bytes are asked for. An item that declares **no** `mimeType`, or `application/octet-stream`, is not refused: silence is not a claim about the format, a filesystem tells its manager nothing about a file it does not recognise, and losing a photograph over that would be the worse error. Such an item is asked for and identified from its own leading bytes, which is what PDM does with every image anyway - the reference it holds is opaque and carries no extension for three of the four managers.

What the list actually gained at 2.2 is the point of the issue: **HEIC and HEIF**, which is what an iPhone has written by default since iOS 11; **TIFF**, which ImageSharp always decoded and two managers never offered; and ten **RAW** families - DNG, CR2, CR3, NEF, ARW, RAF, ORF, RW2, PEF, SRW. RAW is read at the full-size preview every camera embeds for its own screen, so a RAW file costs less to scan than the JPEG beside it rather than more, and a photographer's catalogue is a library PDM can be pointed at.

### Nothing is written to disk, which is what makes the limits real

The bytes go into a pooled buffer that the content hash and the decoder both read (#260). There is no staging file, so `limits.maxImageBytes` is a hard ceiling rather than a policy: one enormous image is refused and the process is not.

The hash is taken as the bytes pass and closed only when the declared length has arrived, so **a transfer that stops halfway fails that image** and the hash of half a photograph can never become a cache key. Where the declaration named a `sizeBytes` or a `contentHash`, both are checked on arrival; a disagreement is `content_mismatch`, the image fails, and the caller may re-read and send it again.

### Credits, and why the unit is an image

Push removed a property that pull had for free. #163 chose pull deliberately - "the core chooses `maxItems` ... that is the backpressure" - and a caller that sends without being asked has to be told what will be accepted. So the limit comes back explicitly, as **credits**: PDM says how many images it will take the bytes of now, and a caller that honours the number never has an upload refused.

**A credit is one image.** Not a rate, and not a byte count. What is scarce is the memory an image in flight occupies and the depth of the embedding queue, both of which are counted in images; `maxIngestCredits * maxImageBytes` is exactly the worst case PDM has to hold for one search, and that product is what a deployment should be tuned on rather than either number alone. A byte-denominated credit would also need the caller to declare a total it may not know before it reads, and a rate would bound the wrong thing - a caller sending one 4 MB photograph a second is not the problem, a caller with two hundred in flight is.

A credit is spent by an upload and returned when PDM is done with those bytes - embedded, cached or failed. Declaring costs none, and an item PDM does not want costs nothing at all.

**Credits ride the stream, and the bytes do not.** The number arrives as `ingest_credits` on `GET /v1/stream` (section 20) and is also on `GET /v1/searches/{searchId}` and on the answer to every batch, because a caller must be able to ingest without holding that connection - the same relationship a progress reading has, where the read is authoritative and the push is the optimisation (#228). It is **absolute, never a delta**: that channel drops messages for a subscriber that falls behind, and a lost delta is a balance that stays silently wrong for the rest of a scan.

The bytes are an ordinary `PUT` over HTTP and were never going to be anything else. The stream is a message bus, not a bulk transport: there is no resume in it, one connection is one queue, a long-lived socket carrying gigabytes through a corporate proxy gives back the deployment win this whole reversal buys, and it would weld a cheap channel held open for hours to a heavy one that churns. An upload over HTTP falls into the enforcement chain, the rate limiter and the problem vocabulary that already exist.

A 429 stays underneath as the floor. `ingest_credit_exhausted` is what a caller that ignores its credits gets, and it is reactive by nature - on a 20 MB photograph it finds out at byte 19,000,000, which is the whole reason credits exist and are not simply a 429 with better manners.

### No idempotency key, and why the reference is a better one

Nothing in ingest carries `Idempotency-Key`. It was a genuine question - the rest of this API speaks that vocabulary, and `POST /v1/imports` still requires one - and the reference wins on three counts.

A key exists to make a `POST` that *mints* something safe to retry. An upload mints nothing: the batch already named the item, the URL already names it, and a second send of the same image is the same state. A separate key would be a second identity for a thing that has one, with a store and an expiry of its own to garbage-collect. And it would not answer the question that actually matters - whether these bytes are the bytes that were declared - which `sizeBytes` and `contentHash` do answer, and which no key could.

The same reasoning covers the other two. `complete` is idempotent on the `searchId`; a second call with the same outcome is a no-op, and with a different one it is `conflict`, because a walk that completed and later claims to have failed is two statements about one fact. Declaring the same reference twice replaces the earlier declaration and recomputes the decision, which is what a manager whose walk restarts needs.

### Resuming, and resuming a walk

An upload is resumable and resume is the exception rather than the shape: an ordinary send carries no `Content-Range` and is one request, and a caller that would rather not think about it sends whole images and retries them whole. A caller that lost a connection mid-image may send `Content-Range: bytes <first>-<last>/<total>`, where `<first>` must be exactly the length of the prefix PDM holds; anything else is `content_mismatch` with a `Range` header saying what it has. A prefix is held in memory against the credit it spent and expires, which fails that image and returns the credit - a partial upload is a small state machine with a garbage collector, and pretending otherwise is how one leaks.

Resuming a *walk* is a different thing and is cheaper than it looks. `POST /v1/searches/{searchId}/resume` reopens an interrupted search, and PDM keeps no cursor for the caller: in 1.x it held one because it did the walking. The caller may simply walk its source again, and every item it re-declares comes back `cached`. The repeat-scan cache is what makes resume cheap, which is the same sentence as the first subsection above.

---

## 10. Images and file operations: what left, and what replaced it

Five operations went at 2.0, and #277 is where the code followed. Each is here with what answers instead, because "why is this gone" is a question somebody will otherwise answer by guessing.

| Removed | Replaced by |
| --- | --- |
| `GET /v1/images/{imageId}/thumbnail` | the manager's frontend asks its own backend, by the `imageRef` on every `ResultImage` |
| `GET /v1/images/{imageId}/content` | the same |
| `POST /v1/files/recycle` | the manager recycles its own file, then `POST /v1/images/gone` |
| `POST /v1/files/delete` | the manager deletes its own file, then `POST /v1/images/gone` |
| `POST /v1/files/move` | the manager moves its own file. **Nothing replaces the `newImageRef` half of it** - see below |

What a manager's own frontend calls instead is `/workspace`, its backend's own surface: two routes for the bytes, one `POST /workspace/images/describe` per page for the captions, and three for the file operations. The shapes are in [resource-manager-frontend.md](resource-manager-frontend.md).

### The bytes were always a round trip through the owner

The image routes were built for a frame that talked to PDM directly, and the loop a manager's frontend drew until #277 makes the point on its own: `/workspace/images/{imageId}/thumbnail` to its own backend, which called `/core/v1/images/{imageId}/thumbnail`, which called the manager's own `/v1/images/{imageRef}/thumbnail`. Three hops to fetch a file from the program that started the request. Removing them shortened it to one.

What was lost with them is real and small: PDM's preview cache, keyed on the manager's `ETag` (#147), and the downscaling fallback for a manager that renders no thumbnail of its own. Both belong to the manager now, which is where the `ETag` came from anyway. The cache is the browser's, revalidating on the same tag, so a second visit to a results page is still a page of 304s. The fallback is thinner: the SDK's `/workspace` layer has nothing that decodes an image, so a manager whose `ThumbnailAsync` answers null serves the original instead of a downscale of it - worse than PDM's answer and better than an empty tile, and it costs nothing today because all four built-ins have served their own previews since #233.

### Deleting somebody's photograph was a ritual, not a control

The three file operations asked PDM to ask the resource manager to act. It is worth being precise about what PDM contributed: it checked a scope, a permission and a declared capability, then relayed a request, then relayed the outcome. It could not perform the deletion. It could not verify one. And since #254 it cannot even ask.

The three-way check went with them, and that is the point rather than a loss:

| 1.x check | Failure | Where the question lives at 2.0 |
| --- | --- | --- |
| scope granted to the registration | `insufficient_scope` | the manager's own authorisation of its own frontend |
| permission of the user behind the call | `insufficient_permission` | the same |
| capability of the resource manager | `resource_manager_capability_missing` | a manager knows what it can do without declaring it to anybody |

Whether this user may delete this file is a question with exactly one correct answerer, and PDM was never it. What PDM keeps is the part nobody else can do.

`dryRun` went with them too. It existed because an agent with a delete tool and a confused model is a photo library shredder (#158), and that concern did not evaporate - it moved to where the delete happens. The MCP server's own opt-in is unaffected, because it was always about MCP's tool surface rather than about this API.

### `POST /v1/images/gone` is the part PDM owns

One operation, and it is the manager reporting rather than requesting: these references no longer exist, so stop naming them.

- **References, not `imageId`s.** The caller has just deleted a file; it holds a reference and no id. Every call is already scoped to (user, registration), so a reference can only reach rows this caller produced.
- **A reference PDM has never seen is not an error.** Managers delete files that were never in a duplicate group, and reporting them is right rather than wasteful. They come back as `unknownRefs` and the call is a 200, and so is a second report of the same reference.
- **A group left with fewer than two images stops being a duplicate group** and goes with them, and the result's aggregates are recomputed. Both are counted in the answer, so a client updates its page without refetching it.
- **It is not `DELETE /v1/results/{resultId}`.** Deleting what PDM computed is a different act with a different scope, precisely so the two can be granted apart.

A collection operation on one result was the obvious alternative and is wrong. A deleted file is gone from *every* result that names it, and a manager cannot know which results those are without asking - so a per-result shape would have made the caller do a query in order to report a fact.

### The one capability 2.0 loses outright

A reference that **changed** is not a reference that is **gone**, and there is no operation for the first. 1.x had one: `POST /v1/files/move` answered with a `newImageRef` and PDM rewrote what it stored, which is why section 5 could claim that a client's list of selected images survives a move that renamed everything underneath it.

That matters for real managers rather than hypothetically. Drive addresses a file by an id that survives a move; `local` and Dropbox address one by its path, so a user dragging a folder changes every reference in it. Such a manager now has two honest options and both lose something:

| Option | What it costs |
| --- | --- |
| report the old references gone | those photographs drop out of their duplicate groups until a rescan |
| say nothing | stored references resolve to nothing, and the tiles stop opening with nothing in any log |

**The first is the recommended one**, because a result that is smaller than it should be is better than one that is wrong, and a rescan restores it.

**#277 took it, and narrowed it by one condition.** A manager reports the old reference gone only when the reference actually changed - that is, when its own `newImageRef` for the moved file differs from the reference it was given. Drive's file ids survive a move, so a Drive move reports nothing and loses nothing; `local`'s and Dropbox's references are paths, so a move there reports every one of them and those photographs leave their duplicate groups until a rescan. Reporting unconditionally would cost Drive its groups for no reason, and reporting nothing would leave the other two with stored references that resolve to nothing and tiles that stop opening with nothing in any log. The manager's own answer is the only thing that can tell the two cases apart.

An operation that renames a reference is still the proper fix, and it is still open. It is **additive** - a minor, not a major - and nothing outside this repository is waiting on it, so it can be taken at any point. Recorded in section 16.

---

## 11. Export and import

`GET /v1/results/{resultId}/export` answers with the export document as JSON, not a file attachment. A real download originates wherever the user is actually looking, which is never here: a manager's frontend builds a `Blob` from what its backend relays, and the MCP server wants the object anyway.

**`POST /import/file` has no successor.** It was `multipart/form-data` for a browser's file input, and a caller here already has the bytes - it reads the file and posts the document.

**`GET /import/result/exists` is folded into validation.** Validating a document already has to resolve which result it would land on, so `conflictsWithResultId` costs nothing and removes a route whose only job was to re-ask a question the previous call had answered - along with the window between the two calls in which the answer could change.

An import may not land under a different resource manager than the one it was exported from. The refs inside were minted by that manager and mean nothing to another, so the result would render as a page of broken tiles.

That check needs the document to name its resource manager. Format 1.0 did not: it carried `dataSource`, the name of the closed enum. #143 bumped the format to 2.0 for exactly this reason - the document now carries `resourceManagerId` (DuplicateSearchResultExport.cs) and the importer still reads the old spelling, mapping the enum member onto the built-in id it became. The `ExportDocument` schema in the OpenAPI document describes that 2.0 shape.

---

## 12. Error model

`application/problem+json` (RFC 9457) with a stable machine `code`. A client branches and localises on `code` and never on `title` or `detail`. Full table in the OpenAPI document.

Pairs are kept apart where each leads a client somewhere different:

| Pair | Difference |
| --- | --- |
| `insufficient_scope` / `insufficient_permission` | an administrator has to act, versus the user may not |
| `rate_limited` / `ingest_credit_exhausted` | wait `Retry-After` seconds, versus wait to be told there is room |
| `conflict` / `search_not_ingesting` | this state prevents it, versus stop walking, nobody is waiting |

`ingest_credit_exhausted` deliberately carries no honest `Retry-After`, which is why it is not `rate_limited` with a number: what has to happen before the caller may send again is an image finishing being embedded, and PDM cannot promise when. The answer is to wait for `ingest_credits` on the stream, or to read the balance on the search.

`content_mismatch` carries two statuses and one meaning: the bytes and the declaration disagree. 400 when the body is wrong - a length or a hash that does not match what the batch said. 409 when a resumed range does not continue the prefix PDM holds, which is a disagreement about state, and the `Range` header then says what it has.

Five codes went at 2.0 and none of them can be produced by anything left in this document. `capability_expired` and `capability_revoked` belonged to a credential class that no longer exists (section 4). `resource_manager_capability_missing`, `resource_manager_auth_required` and `resource_manager_unavailable` - along with the `requiresReauth` and `resourceManagerCode` members of a problem - all reported on a call PDM made to a resource manager, and PDM makes none. A code PDM can never emit is a branch every client carries for nothing.

`detail` never contains a token, a credential, an internal user id, or a path from a resource the caller may not read.

---

## 13. Versioning

`MINOR` negotiated per request through `PDM-Core-Version`. Within a major, a minor may add optional fields, add operations, add enum members and relax constraints; it may not remove or rename anything, make an optional field required, tighten a constraint, or change a status or error code for an existing situation. 2.0 does five of those six, which is what makes it a major.

Negotiation runs the opposite way to the resource manager contract, because here PDM is the server:

- A client sends the version it was built against. PDM behaves as `min(its own minor, that value)` and states what it served in the response header.
- A client asking for a version PDM cannot serve - a different major, or a minor below the floor - gets 400 `unsupported_core_version` naming both.
- No header means the client accepts whatever the deployment serves. That is safe **within** a major, because minors are additive. Across one it is not, and since the path prefix no longer distinguishes majors, the header is the only thing that does. Send it.

### The `/v1` prefix did not move at 2.0

The major used to be in the path, and at 2.0 it stopped being. That is a deliberate decision and not an oversight, so here is the whole of it.

**A path prefix exists so that two majors can be served side by side.** [`version-compatibility.md`](version-compatibility.md) section 3.2 says exactly that - "the path prefix makes an overlap possible - `/v1` and `/v2` are different routes in the same process - so an overlap is what we use" - and then asks for twelve months of it.

**That overlap is not being run here, and is not owed.** Nothing outside this repository implements this API and nothing could have: the SDK is not published (#192 is open), so the only clients of `/core/v1` are the four resource managers built alongside it, updated in the same commit and shipped in the same payload. #197 waived the same window on the same grounds when it withdrew the UI protocol at 1.0. With no second major ever served, a prefix that distinguishes them buys nothing and costs every caller a rewrite of every URL.

**What refuses a mismatched major is the header, and it does so better than a 404 would.** A 1.x client that sends `PDM-Core-Version: 1.4` to a 2.0 deployment gets `400 unsupported_core_version` naming both versions - a diagnosis, where a moved prefix would have given it a bare 404 on a path that used to work. The mechanism was always there; 2.0 is the first time it is the only one.

The cost is written down rather than discovered: a caller that sends no header is served whatever the deployment speaks, and across a major that is a different API. Inside this repository every caller sends the header. Outside it there is nobody, which is the same fact this whole subsection rests on, and if that ever stops being true the prefix is the thing to reconsider first.

Clients must tolerate additions: an unknown scope in `GET /v1/capabilities` is something they cannot use, not an error; an unknown stage key renders from `label`; an unknown error code is treated as its HTTP status.

The support window - how far back support reaches, for how long, how it is communicated - is policy and lives in [`docs/version-compatibility.md`](version-compatibility.md) (#174). This document specifies the mechanism that policy is built on. Two things make the policy sharper for this direction than for the other one: the MCP server is configured by users who will not upgrade in step with a deployment, and on the desktop build the updater replaces the core and the resource managers together, so a partial update that leaves a new core with old managers must be impossible.

Three of the policy's answers matter to a client author here. Minors of a major never expire, so a client built against 1.0 keeps working for the life of `/v1`. A deployment states its window in `GET /v1/capabilities` as `oldestSupportedVersion`, alongside `deprecations[]`, so it can be displayed rather than discovered as a 400. And an operation with a removal date carries `Deprecation` and `Sunset` response headers, with at least twelve months between the announcement and the removal.

Every change to this contract is recorded in [`asp-back/contracts/CHANGELOG.md`](CHANGELOG.md).

---

## 14. The complete flow

A resource manager's backend runs the whole cycle. Every row is a call it makes; nothing in the sequence needs the user's JWT, and nothing in it can name an object belonging to another resource manager.

| Step | Call | Notes |
| --- | --- | --- |
| the backend starts up | `POST /v1/tokens/app` | client credentials from the registration, for one `userRef` |
| it boots a page's session | `GET /v1/capabilities` | the frontend hides every action absent from the effective scopes |
| open the answer channel | `GET /v1/stream` | held while there is work; `hello` says whether to reconcile |
| pick a folder | none | the manager's own picker, on its own origin, talking to this same backend (#248, #250) |
| start | `POST /v1/searches` | 202 with a `searchId` and an opening credit balance |
| walk, and declare | `POST /v1/searches/{searchId}/items` | per item: `cached`, `needsContent` or `rejected` |
| send what is wanted | `PUT /v1/searches/{searchId}/items/{imageRef}/content` | one credit each; `ingest_credits` says how many are free |
| the walk ends | `POST /v1/searches/{searchId}/complete` | the total becomes known and a percentage starts meaning something |
| watch progress | `search_progress` on the stream; `GET /v1/searches/{searchId}` on start-up, after a reconnect, or when the channel is quiet | the read is authoritative, the stream is the optimisation |
| open the result | `GET /v1/results/{resultId}` | counts and origin |
| page the groups | `GET /v1/results/{resultId}/groups?page=1` | identifiers, references and what the scan measured |
| show tiles | none of PDM's | the frontend asks this backend by `imageRef`, and the backend serves its own previews |
| hide a group | `PATCH /v1/groups/{groupId}` `{"hidden": true}` | answers the recomputed aggregates |
| delete a selection | none of PDM's, then `POST /v1/images/gone` | the manager deletes its own files and reports the references |
| export | `GET /v1/results/{resultId}/export` | JSON document; the frontend offers the download |

Two rows are worth reading twice. **"Send what is wanted" is often empty**: a rescan of an unchanged library declares everything and uploads nothing, because every decision is `cached`. And **"show tiles" and "delete a selection" involve PDM only afterwards, or not at all** - which is section 10, and the largest single change 2.0 makes to what a day of using this product costs.

---

## 15. Mapping from today's routes

The acceptance criterion for the surface: every route the private API served in these four groups is accounted for, including the ones that did not survive. **The left column is history since #251** - those routes are removed, and the mapping is kept because it is the record of where each one went.

### `/tasks/*`

| Today | Core API | Note |
| --- | --- | --- |
| `POST /tasks/search/duplicates` | `POST /v1/searches` | `folderRef` instead of a path; engine tuning dropped |
| `POST /tasks/search/duplicates/cancel` | `POST /v1/searches/{searchId}/cancel` | the search is named |
| `POST /tasks/search/duplicates/resume` | `POST /v1/searches/{searchId}/resume` | no more "most recent resumable job" |
| `POST /tasks/search/session/start` | **removed** | no caller; a search creates its own session |
| `GET /tasks/status/active` | `GET /v1/searches?status=active` | scoped to the caller's registration |
| `GET /tasks/status/result` | `GET /v1/results` | opaque `resultId` instead of a composite key |
| `GET /tasks/search/result` | `GET /v1/results/{resultId}/groups` | availability re-check unchanged |
| `GET /tasks/search/hidden` | `GET /v1/results/{resultId}/groups?visibility=hidden` | same query, one route |
| `DELETE /tasks/search/result` | `DELETE /v1/results/{resultId}` | |
| `POST /tasks/search/group/{id}/hide` | `PATCH /v1/groups/{id}` `{"hidden": true}` | idempotent |
| `POST /tasks/search/group/{id}/unhide` | `PATCH /v1/groups/{id}` `{"hidden": false}` | |
| `GET /tasks/image/thumbnail` | `GET /v1/images/{imageId}/thumbnail` | binary, media token, not brokered; itself removed at 2.0 |
| `GET /tasks/jobs` | `GET /v1/searches` | the job record for a search is the search |
| `GET /tasks/status/running`, `/statistics`, `/cleanup`, `/hierarchy/*`, `/force-gc`, `/pool/statistics`, `/jobs/history` | **not here** | admin diagnostics, first-party (#167) |

### `/files/*`

| Today | Core API |
| --- | --- |
| `POST /files/recycle-bin` | `POST /v1/files/recycle` |
| `DELETE /files/permanent` | `POST /v1/files/delete` |
| `POST /files/move` | `POST /v1/files/move` |
| `POST /files/batch` | **removed**; it is the other three switched on a body field |

### `/export/*` and `/import/*`

| Today | Core API |
| --- | --- |
| `GET /export/result` | `GET /v1/results/{resultId}/export` |
| `POST /import/validate` | `POST /v1/imports/validate` |
| `POST /import/json` | `POST /v1/imports` |
| `POST /import/file` | **removed**; a caller already has the bytes |
| `GET /import/result/exists` | folded into `POST /v1/imports/validate` as `conflictsWithResultId` |

### Added with nothing behind them at 1.0

| Core API | Why | At 2.0 |
| --- | --- | --- |
| `GET /v1/capabilities` | a client cannot hide what it cannot see | still here, minus the resource-manager capabilities |
| `GET /v1/resource-managers` | the MCP consumer has no other way to offer "pick a source" | still here, as a catalogue |
| `POST /v1/tokens/app` | #145's credentials, reused by #159 rather than a second credential type | still here, and now the only credential |
| `POST /v1/tokens/capability` | the seam that keeps the JWT out of the frame | **removed**; there was never a frame or a shell |
| `POST /v1/tokens/media` | the seam that keeps image bytes out of the broker | **removed**; there are no core image URLs |

### What 2.0 removed, and what answers instead

The same accounting one major later, so that every retired operation has one place saying where it went.

| 1.x | 2.0 |
| --- | --- |
| `GET /v1/images/{imageId}/thumbnail` | the manager's own backend, by `imageRef` (section 10) |
| `GET /v1/images/{imageId}/content` | the same |
| `POST /v1/files/recycle` | the manager recycles, then `POST /v1/images/gone` |
| `POST /v1/files/delete` | the manager deletes, then `POST /v1/images/gone` |
| `POST /v1/files/move` | the manager moves; a reference that changed cannot be reported, which is the one capability 2.0 loses outright (section 10) |
| `POST /v1/tokens/capability` | nothing. #253's stated trigger, fired |
| `POST /v1/tokens/media` | nothing. The same |
| `ResultImage.name`, `.displayPath`, `.available`, `.trashed` | the manager's frontend has all four locally (section 8) |
| `GroupPageResponse.availabilityChecked` | goes with them |
| `Result.connectionState` | the manager compares the result's `accountRef` with its own session (#250) |
| `ResourceManagerDescriptor.capabilities`, `.health`, `.connected`, `.accountLabel`, `.contractVersion`, `.supportsPicker` | facts about a manager, answered where they are true |
| `StartSearchRequest.recursive` | how deep to walk is the caller's decision |
| `StartSearchRequest.cacheFolderRef` | `cacheOnly` on a declared item (section 7) |
| scopes `images:read`, `files:recycle`, `files:delete`, `files:move` | gone with their operations |
| codes `capability_expired`, `capability_revoked`, `resource_manager_capability_missing`, `resource_manager_auth_required`, `resource_manager_unavailable` | nothing can produce them (section 12) |

### What stays first-party and is deliberately absent

Settings and the cleanup workers, models, identity and roles, health and diagnostics, the admin surfaces for registrations and grants, and `/providers` in its settings-facing form. The SignalR internal broadcast was on this list and is not on any list now - #318 removed the service. The allocation that drew that line file by file is void - see `history.md` section 5.

Browsing is absent for a different reason: it is not ours at all. `/cloud/{provider}/browse`, `/cloud/{provider}/folder-name` and `/cloud/local/drives` become private to the resource manager and leave `PDM.API` with `ICloudBrowser` (#165).

---

## 16. Open points

Decided here rather than deferred, flagged so they can be reopened cheaply:

- **403 rather than 404 for a cross-registration read** (section 4).
- **Images read by `imageId`, and a reference accepted as input on exactly two operations that read nothing** (section 5).
- **`PATCH` on a group rather than `hide` and `unhide`** (section 6).
- **`GET /v1/capabilities` returns the intersection**, rather than granted and permitted separately. A client that wants to explain *why* an action is missing cannot, from this response alone; it learns the reason only by attempting the call. Reporting both sets would let a frontend say "your administrator has not granted this", at the cost of telling a third party about grants it does not have.
- **The `/v1` prefix stays while the major moves** (section 13). The one to reconsider first if this API ever acquires a client that is not built in this repository.

Answered by #274, and listed because each was a real choice with a real alternative:

- **An upload is keyed by `imageRef` and carries no `Idempotency-Key`** (section 9). The alternative was the vocabulary the rest of this API speaks, and it would have added a second identity, a store and an expiry for a `PUT` that already names its target.
- **A credit is one image**, rather than a byte count or a rate (section 9). The alternative that nearly won was bytes, which is the honest unit for a library of RAW files and needs a caller to declare a total before it reads; `maxImageBytes` bounds the worst case instead.
- **"This reference is gone" is a top-level operation on references**, not a collection operation on one result (section 10). A deleted file is gone from every result that names it, and a caller cannot know which those are without asking.
- **The credit balance is absolute and readable, not a delta pushed on a channel that drops messages** (sections 9 and 20).

Genuinely open, and raised by this piece:

1. **Nothing can report that a reference changed rather than vanished** (section 10). 1.x rewrote a stored reference from the `newImageRef` a move answered with; 2.0 has no equivalent, so a manager whose references are paths loses a duplicate group every time a user reorganises a folder. It is an additive fix - one operation, a minor, no second major - and it is deliberately not taken on #274's own authority, because #274's job was to decide what 2.0 is and its Added table is that decision. Decide it before #277 moves file operations to the manager, since that is the piece whose users hit it.

Genuinely open, and belonging to another issue:

1. **Whether two searches can run concurrently.** The API stops assuming one per user; the engine and the process model decide whether a deployment allows it (#163, #168). Until then `search_already_running` may be answered more often than the contract requires, which is compatible in both directions.
2. **What the credit ceiling should actually be.** #275 derives it from the embedding stage's real capacity rather than guessing, and reports it as `limits.maxIngestCredits`. The contract fixes the unit and the mechanism, not the number - and since #326 the number is a fact about the machine PDM is running on rather than about the build, so a caller that reads it off `GET /v1/capabilities` gets a different answer from a four-vCPU container and from a sixteen-core desktop. That is what the field was always for; what changed is that it now varies.

Closed since this document was written:

- **A media token in a query string, rather than a scoped cookie.** Both are gone with the operation (#253, section 4). The reasoning is kept in the changelog because the question - how a page gets read access to bytes without ambient authority - is one somebody will ask again.
- **Whether the first-party shell keeps a results view of its own.** It does not, and it has no working area either (#251). Each manager draws its own screens on its own origin (#248, #250), and the app host mounts them in a frame it authorises by origin (#249).

---

## 17. Deliberately not in this API

| Not here | Why | Where |
| --- | --- | --- |
| image bytes, previews, names, locations, whether a file still exists | the manager's own facts, served by the manager to its own frontend | sections 8 and 10, #254 |
| recycle, delete, move | the manager acts on its own resource and reports it | section 10, #254 |
| browsing, folder names, drive lists, connecting an account | the manager's own frontend and backend | #248, #250 |
| the notification channel itself, and what a browser receives on it | a channel, not a request/response API - what a *backend* receives is section 20 | #161, #246 |
| registry administration, health aggregation | core-side model and first-party UI | #144, #153 |
| creating registrations, granting scopes, rotating secrets, revoking | administrative, and never reachable by a caller of this API | #145 |
| iframe sandboxing and a capability broker | never built. A manager's frontend is embedded by origin, authorised as a grant, and calls only its own backend | #249, #251 |
| settings of any kind | first-party, and never rendered by a resource manager | #167 |
| the support window and the desktop update story | policy | #174 |
| an interface a resource manager implements | there is none left. A manager calls this API and serves nothing to PDM | #278 |

---

## 18. What is served today

**3.0, and the document is the whole story.** This section used to exist because the two differed: #274 wrote the major before anything answered it, and for four issues this was the only place that said what a caller could actually reach. It stays because the question it answers - what does *this* deployment do, as opposed to what does the contract permit - is worth a section of its own, and because two behaviours below are still narrower than the document describes.

The enforcement chain is `CoreApi/Security/`, the ownership predicate is `CoreApiReadStore`, and the acceptance criteria are asserted over a real HTTP pipeline in `PDM.API.Tests`.

**The major is closed on both sides.** Everything 2.0 adds is served - the three ingest operations of section 9, `POST /v1/images/gone`, the `searches:ingest` and `images:forget` scopes, the ingest limits, and `ingest_credits` on the stream. Everything it removes is gone from the code as well as from the document: the image, file and token operations of section 10, the four scopes behind them, the four readable fields on `ResultImage` with `availabilityChecked` beside them, `recursive` and `cacheFolderRef` on a search, `connectionState` on a result, and everything on a resource manager descriptor except an id, a name and an icon. The two scope vocabularies agree exactly, in both directions, which is what `CoreApiContractDriftTests` asserts.

**2.1 adds one operation and one event, both additive** (#295): `GET /v1/searches/{searchId}/groups` and `search_groups`, under scopes that already existed. Section 8 has the reasoning and section 20 the message.

**2.2 adds one response field and one per-item code, both additive** (#207): `formats` on `GET /v1/capabilities`, and `unsupported_format` on a `rejected` declaration. No operation and no scope. The subsection in section 9 has the reasoning; what it is worth to a caller is that the list now holds HEIC, TIFF and ten RAW families, which between them are most of what an ordinary phone and every camera produce.

**2.3 adds one request field and one response field, both additive** (#227): `expectedTotal` on a declaration, which is how a progress dialog gets a denominator before the walk ends, and `itemsPerSecond` on a reading. `totalItems` and `percent` are present as soon as such a count arrives, and `isTotalKnown` keeps its narrower meaning - the walk is over, this is final. No operation and no scope, and all four built-in managers count for themselves. Section 8 has the rules and [manager-scan-driver.md](manager-scan-driver.md) has what it costs each of them.

One thing in that minor is a defect fixed rather than a field added, so it is worth naming here: `percent` is now the stage-weighted figure the pipeline has always computed, where it used to be `processedItems` over `totalItems` - the running stage's own counters. The bar filled to the end while embedding and started again from nothing when comparing began.

**2.4 adds two response fields and one enum, all additive** (#226): `totalUnsupportedImages` beside the `totalBrokenImages` a result has declared since 1.0, and `errorCode` on a `ResultImage`. Nothing about the surface moved - both of the places this fact belongs in were already declared and both were always empty. Section 8 has the reasoning and the two codes.

**3.0 removes the one thing a caller could say, and adds nothing** (#318): `POST /v1/events`, `events:publish`, the four kinds and the two limits that bounded them. The operation was a relay into a notification channel that had had no subscriber since #251, and only one of its kinds did anything besides relaying - a number 2.3's `expectedTotal` already carries on a call the caller is making anyway. Section 19 has the record and history.md has the decision.

**`PDM-Core-Version` negotiates inside the 3.x major only.** A client asking for a 2.x or 1.x minor is refused with `400 unsupported_core_version` naming both sides, rather than served an API that has none of the operations it was written against. `CoreApiVersion.OldestSupported` is `3.0` and does not follow the served minor: a major starts its own window, and within one the floor moves only for a defect that cannot be fixed additively.

**There is one direction, and it is the grant that says so.** Feeding a search needs `searches:ingest`. Until #278 a registration without it got a walk PDM performed over the gateway, which was the transitional switch #276 threw for all four built-ins; there is no gateway and no alternative, so `credits` on an opened search is present for every search that can be fed and the branch that read it is gone from the pipeline. See [manager-scan-driver.md](manager-scan-driver.md).

One thing the plan for #254 guessed wrong, recorded because it reads as a slip: an abandoned upload does **not** expire in the job cleanup worker. What expires is a pooled buffer and a credit in the API process's memory, and a worker with its own `Program.cs` and its own connection to the database can see neither. It is `SearchIngestSweeper`, a hosted service in `PDM.API`, on a fifteen-second tick.

### Behaviours narrower than the document describes

Two are left. Each answers honestly rather than pretending, and neither is a shape a client has to write a second code path for.

| Behaviour | Today | Arrives with |
| --- | --- | --- |
| **Progress across a restart** | The reading a search broadcasts is kept in memory and does not survive a restart (#228). The job record does, so a read after one answers "running, no numbers yet" rather than zero of zero, which is the honest shape and the reason no client needs a second code path. | not scheduled |
| **A move that keeps a photograph in its group** | A manager whose references are paths reports the old one gone (#277) and the photographs leave their duplicate groups until a rescan. The `newImageRef` half of the retired move operation has no replacement; closing it is additive. | not scheduled |

The other row here was full-size image bytes: `GET /v1/images/{imageId}/content` buffered rather than streamed, so `Range` was not honoured. It was honoured once, for a file on this host that the core opened directly; #273 removed that path and #277 removed the operation, so it is a gap that closed by removal rather than by being fixed. A manager serves its own bytes now, off its own source, with its own range handling.

`folderRef` and `displayLabel` are stored apart, which they were not. A search records the reference its picker supplied and the label beside it; a result records the reference it was scanned from and keeps the readable label beside that. For a cloud manager those two genuinely differ, and conflating them meant a cloud search could never be matched to the result it had just written. 2.0 does not change any of that; it is the reason `displayLabel` is worth sending.

Two obligations this document used to place on the implementation are **gone with the credential that owed them** (#277). A media token travelled in a query string, because its whole purpose was to be usable from an `img src` where no header can be set, and that cost a redaction rule in PDM's own request logging and a `Referrer-Policy: no-referrer` on every image response. No operation here accepts a token in a URL now, so there is nothing in a request line to redact and no image URL to keep out of a `Referer`.

The private `/tasks/*`, `/files/*`, `/export/*` and `/import/*` routes were untouched, were not aliases of these and were never reimplemented on top of them. They were expected to go with #165 and did not, because that turned out to be a different piece of work; #251 is the issue that owned their retirement and they are gone. What is left under `/tasks` is the admin diagnostics sub-group, which was never part of this mapping.

### 18a. A result whose resource manager is gone

[#172](https://github.com/Iskandarus/photo-duplicates-manager/issues/172). Removing a registration does not delete the results that name it - it retains them, so that a registration made again under the same id reattaches them. Those results are **not representable in this contract, and that is not a gap**.

Every call here is scoped to a registration, and an app token is minted from that registration's client credentials. A manager with no registration has none, so no token can be minted for it, `CoreApiAuthority` never covers its id, and the ownership predicate never matches its rows. There is nothing this API could be asked that an orphaned result would be the answer to.

That is the same conclusion from the other direction: **nothing renders an orphaned result**. Its names, its previews, its locations and its file operations are all the manager's - which 2.0 makes literal rather than merely true in practice, since PDM no longer holds even a stale copy of any of them - so a page about it cannot be drawn by anybody, least of all by a frontend belonging to a manager that is gone.

What is answerable is the *fact* of the result, and that lives on PDM's own private surface, which is the host's: `GET /resource-managers/orphans`, `DELETE /resource-managers/orphans/{id}`, and `DELETE /admin/resource-managers/{id}/orphaned-data` across every user. See resource-manager-orphans.md.

### 18b. The two operations with no caller, and the trigger that fired

[#253](https://github.com/Iskandarus/photo-duplicates-manager/issues/253) recorded that `POST /v1/tokens/capability` and `POST /v1/tokens/media` were served, tested and called by nothing. Both were written for one consumer: a workspace running cross-origin in a frame, brokered by a first-party shell, fetching its own image bytes from PDM. There was no such consumer and none was planned. [#217](https://github.com/Iskandarus/photo-duplicates-manager/issues/217) had given a manager's frontend exactly one correspondent, its own backend, which already holds an app token.

They were kept, because removing an operation from a published document is a major under [version-compatibility.md](version-compatibility.md) (#174) and that was the wrong price for two operations that cost nothing to serve while a caller was still possible. And the caveat named its own end: **"re-read when #160 ships, or at the next major of this API, whichever comes first."**

**2.0 is the second of those, so both are removed.** The condition was the whole point of writing it down. "For now" with nothing attached is exactly how the in-process OAuth path survived - it lost its caller in #150 and compiled quietly for two releases until #229 counted the tables, and deleting it then cost one compile error. MCP has not taken the media token up, so it is dead with evidence rather than by argument, and a major is the moment removal is cheap.

The reasoning behind them is not deleted with them. A surface that brokers calls on a user's live session still needs exactly the seam a capability token was, and a page that wants real URLs in `img src` without ambient authority still needs something shaped like a media token. Both are described in the changelog entry for 2.0 and in section 4, so the next person to need one finds a design rather than an absence. What changed is not that the idea was wrong - it is that the topology moved: the browser talks to the manager, the manager talks to PDM, and nothing in the browser holds anything of PDM's.

**#277 removed them**, together with the capability and media token classes, the query-string credential position, the log redaction enricher and the `Referrer-Policy` on an image response there no longer is. `CoreApiTokenKind` is a one-member enum rather than nothing at all, and the hole in its numbering is the point: the `kind` claim is parsed by name, so a capability or media token minted by a 1.x build fails to parse instead of verifying as a full app credential.

## 19. The event channel, and the record of its removal

Added by [#161](https://github.com/Iskandarus/photo-duplicates-manager/issues/161) in core API 1.1 and **removed at 3.0** by [#318](https://github.com/Iskandarus/photo-duplicates-manager/issues/318). `POST /v1/events` was the only operation in this document that accepted a statement rather than a question, and the only path by which code we did not write could put something on a user's screen.

This section is kept rather than deleted because the question it answered - can a resource manager tell a user something, and through what - is one somebody will ask again, and because the shape of the answer is what decides it.

### What it was

Four kinds, each carrying a closed block with every string capped: `account_connected`, `account_disconnected`, `enumeration_progress` and `resource_manager_unavailable`. The user was never in the request: an app token is minted from a `userRef` PDM sealed for one registration and no other (#196), so addressing somebody else was unrepresentable rather than refused. It had a rate bucket of its own, charged per registration and before the body was read, and a size cap enforced during the read rather than after it.

The relay itself was never in any contract. The channel was SignalR, in a service of its own (`PDM.SignalR`), authenticated with the user's own session - so what this document specified was the manager-to-PDM half, and it was an ordinary POST with the same credential, the same problem vocabulary and the same enforcement chain as everything else here. That was the whole argument for its being in this document at all.

### Why it went

**The channel had had no subscriber since [#251](https://github.com/Iskandarus/photo-duplicates-manager/issues/251).** The app host draws no working area, and the screens that watch a scan belong to each manager's own frontend, on that manager's own origin, whose served document carries `connect-src 'self'` (#248) - so PDM's hub is not something those pages may reach even in principle, and they do not need to: their own backend holds `GET /v1/stream`.

So the operation relayed into nothing. That is a published promise that had quietly stopped being true, and the two honest ways out were to say in this document that three of the four kinds are accepted and dropped, or to take the kinds with the channel. The second was taken.

**Three of the four had nowhere else to arrive.** The fourth, `enumeration_progress`, also refined the stored progress reading - and 2.3 had already given that number a better door: `expectedTotal` on a declaration (#227), counted by the same walker and carried on a call it is already making. Sending both is one number twice, which is why #276 sent neither: none of the four built-in managers was ever granted `events:publish`.

**No overlap was owed**, on the ground #197 and #278 stood on: no `sdk-v*` tag has been cut, so the bundle #192 built has never left this repository and no 2.x client can exist outside it. It is the last version that will be true of. The changelog row says so rather than leaving it to be inferred, which is the rule section 3.2 of [version-compatibility.md](version-compatibility.md) sets.

### What answers each kind now

| Kind | What answers it |
| --- | --- |
| `account_connected` / `account_disconnected` | the manager runs the flow on its own origin and tells its own frontend (#250); whether an account is connected is asked of the manager, the only side that knows (#229) |
| `enumeration_progress` | `expectedTotal` on `POST /v1/searches/{searchId}/items` (#227), and `search_progress` on the stream carries the reading it refines |
| `resource_manager_unavailable` | nothing here, and nothing sent it: a manager tells its own frontend, which is the page that would have rendered it |

### One thing that did not change

`isFinal` on the enumeration block was relayed and folded into nothing, and the distinction it drew outlives it: **the caller finishing its walk is not the same fact as PDM's total being known.** PDM can be several batches behind, so a total declared final while images are still being embedded is `processedItems` past `totalItems` and a percentage over 100 - the exact failure `isTotalKnown` exists to prevent. What makes the total known is `POST /v1/searches/{searchId}/complete`, and even then only once the queue behind it has drained.

### The direction this section never covered

What a manager's backend is *told*. That is section 20 (#246), and it is the only push there is now. The two were never the same thing and must not be confused in whatever replaces either: this one was a statement a caller made and PDM relayed to a browser, that one is PDM answering a caller about work it started.

The full reasoning, including why giving the channel a consumer was refused, is history.md.

---

## 20. The answer path

Added by [#246](https://github.com/Iskandarus/photo-duplicates-manager/issues/246), in core API 1.3. `GET /v1/stream` is the direction PDM had none of. Section 19 is a caller telling PDM something and everything above it is a caller asking PDM something; this is PDM telling the caller.

### Why there has to be one

A manager's backend submits work and gets an acknowledgement. Everything after that - the progress, the result, the failure - arrives here or nowhere, because **the core only answers and never dials**. It has no address for a resource manager, and under [#254](https://github.com/Iskandarus/photo-duplicates-manager/issues/254) it will not even expect one to serve HTTP. The alternative was a backend polling for a job that runs for hours, which is the design this replaces rather than supplements.

### Server-sent events, and deliberately not a hub

PDM used to run a realtime channel of its own - SignalR, in a service of its own, authenticated with the user's session - and none of it was reachable with a core API credential or named in any contract, so "reuse the hub" was never one of the options here: it was "build a second hub, with its own handshake, its own authentication and its own framing library, for one direction of one flow". That service is gone (#318) and the argument survives it intact, because what it turned on was the cost of a second protocol rather than the existence of the first.

What is here instead is a plain `GET` that stays open. It carries the app token the caller already holds, it is refused by the same enforcement chain in the same `application/problem+json` vocabulary as every other operation, and a backend written in anything can hold it open with the HTTP client it already has. That last point is the whole argument: the audience for this contract is somebody else's program in somebody else's language, and a framing library is a dependency we would be choosing on their behalf.

### Four things it gets right, by construction

**The scope is the credential's.** Messages are about one (user, registration) pair - the user the app token was minted for, and the registration it names. `resourceManagerId` narrows a token that acts for several and can do nothing else; there is no field in which a caller could name somebody else. That is the same property the event channel had, for the same reason: an app token is minted from a `userRef` PDM sealed for that registration and no other (#196), so addressing a stranger is unrepresentable rather than refused.

**A subscriber is not a publisher.** A `GET` carries no body and the connection has no client-to-server message, so this cannot quietly become a way of putting something on a user's screen. There is no such way at all since 3.0: `POST /v1/events` was the one and it went with the channel it relayed into (#318, section 19).

**Reconnection is ordinary, and the first frame says whether it was clean.** A deploy rolls, a container restarts, a socket drops, and a scan outlives several of those. Nothing is buffered for a subscriber that is not connected - a queue held for an absent backend is a leak with a manager's name on it - so the `hello` frame states plainly whether the caller may trust what it already believes. `resync: false` means the scope's sequence has not moved since the id the caller presented; everything else, including every first connection, is `true` and means "read `GET /v1/searches` and carry on from what it says". That reconciliation is #228's, and it is why replay was not built: the read already exists and it is authoritative where a replayed message would only be recent.

**A client that is not subscribed costs nothing.** Publishing looks a channel up and returns when there is none; it never creates one. A deployment whose managers never subscribe pays one dictionary miss per progress tick, which matters because most deployments run one or two managers and neither may want a live channel at all.

### What it carries

| `event` | Means | What a caller would pull instead | Scope behind it |
| --- | --- | --- | --- |
| `hello` | First frame of every connection. No `id`, because it is not part of the sequence. | - | `stream:subscribe` |
| `resync` | This connection lost messages. Reconcile. | - | `stream:subscribe` |
| `search_progress` | A running search has a new reading. | `GET /v1/searches/{searchId}` | `searches:read` |
| `search_ended` | A search completed, failed or was cancelled. | `GET /v1/searches/{searchId}` | `searches:read` |
| `ingest_credits` | PDM will take the bytes of this many more images (2.0). | `GET /v1/searches/{searchId}` | `searches:ingest` |
| `search_groups` | A running search has formed more duplicate groups (2.1). | `GET /v1/searches/{searchId}/groups` | `searches:read` |

**`stream:subscribe` opens the channel; the fourth column is what travels on it** (#306). Each of the four messages is a reading the same caller could have asked for, and it is delivered only to a connection whose caller holds the scope that read is behind - so nothing here is a way of learning something an ordinary call would have refused. That is #246's own rule, and until #306 it was applied to `ingest_credits` alone: a registration granted `stream:subscribe` and not `searches:read` was pushed readings and counts it could not have asked for. All three search messages move together rather than one at a time, which is also why #295 filtered none of them when it added the third: filtering the newest of three kinds of one thing is a rule the next reader cannot infer.

Not holding a scope is **not an error and not a refusal**. The connection opens, `hello` and the heartbeat arrive, and the kinds this caller may not be told simply never do - including for a caller that may be told nothing at all, which is held open and quiet on purpose. A grant is something an administrator may be part-way through making, and a channel that answers and says nothing is a better diagnosis of that than a `403` whose cause a caller has to guess.

The user half of the question was already right and is worth stating so nobody re-solves it: `searches:read`, `searches:ingest` and `stream:subscribe` are each narrowed by a `Permission` in section 4's table, so no user was ever pushed what their own role would refuse them on the read. What #306 narrowed is the registration's grant.

`search_progress` carries the numbers `GET /v1/searches/{searchId}` answers with, because they are the same reading: every reading a scan broadcasts is kept (#228), and this is that entry pushed rather than a second measurement of it. A subscriber that also polls can never see the two disagree.

`search_ended` **names the search and does not carry its result**. The result id is written by the transaction that persists the grouping, so a message that carried one would be reporting a row rather than an event - and #130 is the memory of what happens when a completion announcement races that write. Read the search; the result id is on it.

A cancellation is the ending nothing else reports. A cancelled run unwinds on an `OperationCanceledException` and broadcasts nothing on its way out, so the store forgets its reading (#228) - and to a subscriber that is news, not silence.

`search_groups` is the same division one step earlier, and section 8 has the reasoning. It says how many groups and how many images a running search has formed; the groups are read with `GET /v1/searches/{searchId}/groups`. A message carrying the grouping itself would put a scan's worth of payload on the one channel whose whole design is that nothing is buffered for a subscriber that is not connected.

### Backpressure, and what a slow subscriber is told

The writer is a scan's progress tick. A queue that blocked would let a manager which stopped reading its socket slow somebody's scan down, so the queue in front of each connection is bounded and a full one refuses the write. What was lost is then stated: a `resync` arrives ahead of the next message that does get through, and the answer is the same one as on a reconnect.

That is the small, local half. **The other half is `ingest_credits`, and 2.0 decides it** (section 9). It is worth being clear that the two are unrelated mechanisms that happen to share a socket: the bounded queue protects *this connection's writer* from a slow reader, and a credit protects *PDM's memory* from a fast one. A subscriber that falls behind loses messages; a caller that ignores its credits is refused an upload.

Three properties of a credit follow from riding this channel, and each is why it rides it rather than being answered on demand. It is **absolute, never a delta**, because this channel drops messages and a lost delta stays wrong for the rest of a scan. It is **also readable**, on `GET /v1/searches/{searchId}` and on every batch answer, because nothing here is authoritative and a caller must be able to ingest with no stream at all. And it is **only delivered to a caller that holds `searches:ingest`** - a subscriber without that scope simply never sees the event, which is not an error, it is a message about work it cannot do. That is the same rule the table above states for every kind; credits were only the first place it was applied.

**The bytes do not ride this socket**, and that was settled on #254 rather than here. It is a message bus and not a bulk transport: no resume, one connection is one queue, gigabytes through a corporate proxy give back the deployment win the reversal buys, and it would weld a cheap channel held for hours to a heavy one that churns. It would also be a second inbound door with a second authorisation story, where an ordinary `PUT` falls into the chain this API already has.

### The connection never outlives the credential

Authority is re-derived per call (section 4) and a stream is one call, so a socket held for a day would be a grant checked once yesterday. The response ends when the app token expires, `hello` says exactly when, and the caller re-exchanges its client credentials and reconnects - which is the same thing it does after a deploy. Two ceilings bound the rest: `limits.maxConcurrentStreams` per (caller, user), reported because a subscriber is the whole of its own allowance, and a registration-wide one that is deliberately not reported, on the same reasoning as `RegistrationRequestsPerMinute`. Over either is `429` with `Retry-After`.

A subscription is per (user, client) because that is what the credential names. A manager therefore subscribes for the users whose work is actually in flight, not for everybody it has ever been linked to - and the registration ceiling is what makes that a rule rather than a suggestion.

### What it does not carry, and why

The live duplicate-group preview and the "you have an interrupted search" prompt both reach the browser and neither is here. The first is a full snapshot of every group found so far, sent per tick, which is a rendering aid rather than an answer; the second is a fact a subscriber gets from `GET /v1/searches?status=interrupted`, which it reads on reconnect anyway. Adding either would be putting something on this channel that a reconcile already answers better.

Server-side prose - the broker, the observer the progress store hands readings to, and the ceilings - is `core-answer-path.md`.
