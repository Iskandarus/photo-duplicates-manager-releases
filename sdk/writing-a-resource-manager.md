# Writing a resource manager

A resource manager is a program that owns a collection of photographs - a folder on a disk, a cloud drive, a photo service - and wants Photo Duplicates Manager to find the duplicates in it.

**You implement nothing.** There is no contract you serve, no endpoint PDM calls, no address you have to expose. Your manager is an API client: it holds credentials, it walks its own resource, it sends PDM what it finds, and it shows the user whatever it wants to show them on its own screens.

That is the shape [#254](https://github.com/Iskandarus/photo-duplicates-manager/issues/254) settled and [#278](https://github.com/Iskandarus/photo-duplicates-manager/issues/278) finished. There used to be a second contract - a manifest, a walk, a read, a thumbnail, file operations - that PDM called into your service. It is withdrawn; [`resource-manager-api.md`](resource-manager-api.md) is the record of what it was and why it went.

The one document is [`core-api.v1.yaml`](core-api.v1.yaml), and the prose beside it is [`core-api.md`](core-api.md).

## Where to get this

This document, the contract and the rest of the prose ship as one zip, from the releases of **`Iskandarus/photo-duplicates-manager-releases`**, under a tag that begins `sdk-v`:

```text
pdm-resource-manager-sdk-3.0.0.zip     core API 3.0, first publication
pdm-resource-manager-sdk-3.0.1.zip     the same contract, a clearer guide
```

The first two numbers are the **core API version**, which is what your client negotiates with the `PDM-Core-Version` header on every request; the third counts re-publications of the same contract and means nothing to a client. A `.sha256` sits beside the zip.

The same documents are readable without downloading anything, at [github.com/Iskandarus/photo-duplicates-manager-releases/tree/HEAD/sdk](https://github.com/Iskandarus/photo-duplicates-manager-releases/tree/HEAD/sdk). That copy is pushed by the same script that assembles the zip, so it cannot say something different; it is always the newest publication, and the version it is at is on the `README.md` beside it. Take the zip when you want the contract on your disk to generate a client from.

PDM's own source is private, so what is in that zip is the whole of what an implementer is given - which is why the guide is written to be enough on its own, and why every document it links to travels with it. Sentences here sometimes name a file in PDM's repository; in the published copy those are plain text rather than links into a tree you do not have.

There is nothing to install in order to *write* a manager: it is an HTTP client of the contract, in whatever language you like. There is one tool worth having in order to *check* one, and it is **on the same release, under the same number**:

```text
pdm-harness-3.0.1-win-x64.zip          one self-contained file, nothing to install
pdm-harness-3.0.1-linux-x64.tar.gz
pdm-harness-3.0.1-osx-arm64.tar.gz
pdm-harness-3.0.1-osx-x64.tar.gz
```

`pdm-harness` plays the core on a loopback port and tells you what your manager got wrong; section 11a below says what it is for and [`manager-harness.md`](manager-harness.md) is the whole of it. One version covers both halves on purpose - a tool and a guide out of one release cannot disagree about which contract they describe. On macOS and Linux, `chmod +x pdm-harness` after unpacking. (There used to be a different tool, `pdm-conformance`, which checked a manager against the contract it *served*. It went with that contract - see [`resource-manager-api.md`](resource-manager-api.md).)

---

## 1. What you are building

Three things, and only the first is required.

| | What it does |
| --- | --- |
| **A scan driver** | Walks your resource, declares what it found, uploads the images PDM asks for. This is the whole of the integration. |
| **A frontend** (optional) | Whatever you want the user to see: your library, your account screen, the duplicate groups PDM found. It talks to your own backend and nowhere else. |
| **An account link** (optional) | If your resource belongs to somebody - a cloud account - connecting it is yours end to end. PDM never sees a token of yours and never runs your OAuth. |

What PDM contributes is the part that is hard: embeddings, similarity, grouping, job orchestration, resume and cancellation, and the storage behind all of it.

What PDM never has is anything readable about a photograph. It holds an identifier, a reference you minted, and the manager the two belong to. Its name, its preview, its location and whether it still exists are yours, and your own screens are where a person reads them.

---

## 2. Ten minutes

```text
1  POST /core/v1/tokens/app        exchange your client credentials for an app token
2  POST /core/v1/searches          open a search over a folderRef of yours
3  count your own resource, then walk it
4  POST /core/v1/searches/{id}/items          declare a batch, with expectedTotal; PDM answers per item
5  PUT  /core/v1/searches/{id}/items/{ref}/content    send the ones it asked for
6  POST /core/v1/searches/{id}/complete       the walk is over
7  GET  /core/v1/searches/{id}     watch it, or hold GET /core/v1/stream open
8  GET  /core/v1/results/{id}/groups          the duplicates, as references you recognise
```

Steps 4 and 5 are the loop, and step 4 is where the cost is decided: PDM answers each declared item with `cached`, `needsContent` or `rejected`, and an unchanged library uploads nothing at all.

### Count first, because nobody else can

`expectedTotal` on a declaration is how many images your whole walk will find. It is the denominator of the progress dialog somebody watches for the length of a scan, and **PDM has no way of producing one**: it is handed one batch at a time and never sees a folder, so a scan of a photo library used to show a number that only climbed with nothing to read it against.

None of the four managers PDM ships gets it for free either - no cloud service here has an operation that answers how many images a folder holds - so each does a listing pass with nothing opened, nothing hashed and nothing downloaded, in front of a scan that downloads and embeds every photograph it finds. Write yours for counting rather than performing the walk twice: ask for no per-item data beyond what tells you whether an item is a photograph, and page at your service's own maximum rather than at the batch size PDM accepts.

Four rules, and none of them is a burden:

- **Count only what you would declare.** An item you are going to filter out - a format `GET /core/v1/capabilities` does not list, a file that is not a photograph - is one that will not be in the result. The number at the end of the walk has to be the number actually processed.
- **Send it on every declaration.** It rides a call you are making anyway, and repeating it is what gets the number to a PDM that restarted mid-scan.
- **Being wrong is not an error.** Nothing loops until your count is reached and no search completes because of it, and PDM never reports it below what you have actually declared. Send a corrected number whenever you have one; it may move in either direction.
- **Sending nothing is allowed.** Omit the field and the screen says "found N so far", which is what every 2.2 client does. It is a percentage a person can read, not a requirement.

---

## 3. Register with PDM

A registration is a grant of authority to your code, so it is made by an administrator in PDM's admin panel rather than by you. What it carries:

| | |
| --- | --- |
| **id** | The short name everything keys on: `googledrive`, `dropbox`, yours. |
| **display name, description, icon** | What a person reads when PDM names your manager on a screen. Since #278 these are on the registration, because there is no manifest for PDM to fetch them from. |
| **client credentials** | An app id and a secret, issued once. The secret is shown at issuance and never again. |
| **granted scopes** | What you may call on the core API. Empty until somebody writes them down. |
| **frame origin** | Where your frontend is served from, when an administrator wants PDM's own start page to mount it. Optional, admin-granted, and never given to a manager the deployment did not ship. |

### One secret, and what it is not

The client secret authenticates **you to PDM** on `/core/v1`. PDM stores it as a one-way hash and cannot use it for anything else.

There used to be a second, a signing key PDM proved itself to you with on every outbound call. PDM makes no outbound call. The key survives for exactly one purpose and only where an administrator mounts your frontend inside PDM's own window: PDM signs a short-lived token the **browser** carries to your origin, saying which user is looking at the page. If nobody frames your frontend, you never see it.

---

## 4. Refs: opaque, URL-safe, stable, and yours

`imageRef`, `folderRef` and `accountRef` are opaque tokens you mint. PDM stores them and hands them back verbatim, and PDM never parses one.

They are restricted to `A-Z a-z 0-9 . _ ~ -`, and may not be `.` or `..`. If your natural identifier is not URL-safe - a Windows path, a Dropbox path, anything with a slash - encode it, for example base64url. Encoded slashes in path segments are a reliable source of proxy and framework bugs, and the restriction also removes any temptation for PDM to read structure into a ref.

**Refs must stay stable for the lifetime of the item**, because PDM stores them in search results that outlive the search. When one does change - a move, on a source that addresses a file by its path - tell PDM the old one is gone (section 7) and declare the new one on the next scan.

`userRef` is the exception: it is **PDM's**, not yours. It is the opaque per (user, registration) reference PDM mints, and it is how you say which of your users a call is about without either side learning the other's identifiers.

---

## 5. The walk

Your walk, your cursor, your batch size. PDM has no concept of a directory and never asks you to recurse: how deep to go is your decision and needs nobody's permission.

Per batch:

1. **Declare** what you found: for each image, its `imageRef`, `sizeBytes`, and whichever of `contentHash` and `etag` you can produce cheaply.
2. **Read the answer.** Each item comes back `cached` (PDM already has an embedding for these exact bytes - send nothing), `needsContent` (send the bytes) or `rejected` (with a reason).
3. **Send only what was asked for**, one `PUT` per image, and never before checking you have a credit.

`contentHash` and `etag` are what make a repeat scan free. Either is enough; a manager that supplies neither is correct, and pays for it by uploading a library it has already uploaded once.

### Enumerate what PDM can read

`GET /core/v1/capabilities` answers `formats`, with `mediaTypes` and `extensions`. It is what this deployment can turn into an embedding, and it is the only list worth filtering your walk against: your service knows what it can serve you, and what PDM can decode is PDM's to say. Use whichever spelling suits your source - names or types - and match case-insensitively.

**Read it per scan and do not pin a copy.** The list grows in minors, and a manager holding the copy it read in 2.2 is one whose users cannot scan whatever a later minor adds. A deployment older than 2.2 answers no `formats` at all, which means it did not say rather than that it can decode nothing - enumerate as you would have.

An item whose `mimeType` names an image format absent from the list is `rejected` with `unsupported_format`, before any bytes are asked for. An item that names **no** media type, or `application/octet-stream`, is never refused for its format: silence is not a claim, and PDM identifies those from their own leading bytes. So declaring nothing is safe and guessing is not.

### Credits are the backpressure

PDM holds an image in memory from its first byte until the pipeline is done with it, so it tells you how many it will take at once. A credit is **one image**, not a rate and not bytes.

The balance is on the answer to every batch, on `GET /core/v1/searches/{id}`, and pushed as `ingest_credits` on the stream. Read it **before you open an image**, not after: nothing is worse than pulling a 20 MB photograph off somebody's cloud account to find out at the last byte that there was no room for it.

`GET /core/v1/stream` is an optimisation, never a requirement. A driver that reads the balance off the batch answers alone is correct with no stream at all.

### Ending the walk

`POST /core/v1/searches/{searchId}/complete` says the walk is over. It is the only thing that gives PDM a denominator - until then a scan honestly reports "found N so far" rather than a percentage against a number nobody knows.

### Resuming, and what PDM cannot hand back

A search whose run was lost to a crash or a restart is `interrupted`, and `POST /core/v1/searches/{searchId}/resume` opens it again. Three things about the answer are worth knowing before you offer a Continue button.

- **It is a new search.** The response carries a new `searchId` with `resumedFromSearchId` naming the old one, so the walk you start is keyed by the new id. There is no cursor to pick up: PDM keeps none, because the walk is yours. Walk your resource again from the top - an unchanged library re-declares its contents and uploads nothing (`cached`), which is what makes that cheap.
- **`folderRef` and `accountRef` come back on it**, and they are the search's own. Take both from the answer rather than from whatever your screens are set to now: a person can have moved on to another account between the interruption and the Continue, and their scan did not.
- **Anything else you walked with is yours to remember.** PDM never hears whether your walk descends, so it cannot hand that back either. If your walk has parameters of its own, store them against the `searchId` somewhere that survives a restart - a restart being exactly what made the search resumable. PDM's own four managers keep a row per search for this; see `manager-scan-driver.md` section 3c.

---

## 6. Showing the user

Everything readable is yours, and you already have all of it.

- **Names, locations, previews.** Your source knows them; PDM never did without asking you.
- **Whether a file still exists.** Same.
- **Recycle, delete, move.** Your file operations, on your resource, with your credentials.

What you read from PDM is what only PDM has: which of your references are in which duplicate group, and how similar they are. `GET /core/v1/results/{resultId}/groups` answers in references you minted, so joining them against your own state is a dictionary lookup.

A tile costs you one preview from your own source and a page costs you one metadata call to your own backend. Neither goes through PDM, which is the whole point: PDM relaying a request to fetch a file from the program that started the request was three hops to do nothing.

### Do not make them wait for the end

A scan of a hundred thousand photographs runs for a long time, and duplicates exist long before a result does. `GET /core/v1/searches/{searchId}/groups` answers with the groups the running search has formed so far, in the same paged shape a finished result's page has, and `search_groups` on the stream says when there is more to read.

Three things to know before you draw it. It is a **snapshot and not an accumulation**: each one replaces its predecessor, so re-reading never double-counts and a dropped message costs nothing. Its `groupId` is **stable across the snapshots of one search and is not the identifier the saved result will carry** - use it to key your rendering and for nothing else. And an **empty page is an ordinary answer**: before the grouping stage, after it, and once the result is written there are no live groups, so when the search ends you switch to `GET /core/v1/results/{resultId}/groups` rather than treating the empty page as a scan that found nothing.

---

## 7. When a file goes away

You deleted it, the user deleted it elsewhere, it moved and your reference changed. Whatever the cause, `POST /core/v1/images/gone` reports the references, and PDM drops them from every stored result that names them - which is the part you cannot do, because you do not know which results those are.

Two rules worth stating:

- **Report a move only when the reference actually changed.** A source that addresses a file by an id keeps that id across a move; reporting it gone would cost the photograph its duplicate groups for nothing. A source that addresses a file by its path has a new reference and must report the old one.
- **`images:forget` is narrowed by no user permission**, deliberately. The deletion already happened, and gating the report would leave a read-only user's results naming files that do not exist, for ever.

---

## 8. Your own frontend

Optional, and yours whatever you build it in - a web page, a desktop window, a phone app. It talks to your backend, and your backend holds the app token.

If an administrator grants your **frame origin**, PDM's start page can mount your page in a frame. Two documents have to agree for that to work: PDM's `frame-src`, composed from the grant, and your own `frame-ancestors`. When the frame boots, PDM hands it a short-lived token naming the user; you exchange it for a session of your own, on your own origin, and PDM's credentials never leave PDM's origin.

Nothing about that path is required. A manager whose frontend is opened somewhere else works exactly the same way; it simply learns which user it is acting for by its own means.

---

## 9. Errors

`application/problem+json`, with a machine-readable `code` you branch on and a `title` and `detail` you do not. The vocabulary is in `core-api.v1.yaml`; the two pairs worth telling apart:

- `insufficient_scope` against `insufficient_permission`: "an administrator has to grant you this" against "this user may not do it".
- `rate_limited` against `ingest_credit_exhausted`: "wait the stated seconds" against "wait to be told there is room". The second carries no `Retry-After`, because what has to happen is an image finishing being embedded and there is no honest number of seconds for that.

---

## 10. Versioning

`PDM-Core-Version` on every request says what you were built against; PDM answers with what it served. A minor is additive and is negotiated down to the older side. A **major** is refused rather than degraded, with both versions named - so a client written against an earlier major gets a diagnosis and not a mystery. This deployment serves 3.0; a client asking for 1.x or 2.x is answered `400 unsupported_core_version` naming both sides.

The rules are [`version-compatibility.md`](version-compatibility.md), and every change is in [`contracts/CHANGELOG.md`](CHANGELOG.md), whose second column is written for an implementer who does nothing.

---

## 11. Before you ship

- Your refs are stable, URL-safe, and something your own code can resolve back to an item.
- Your walk declares `contentHash` or `etag`, so a second scan of an unchanged library uploads nothing.
- Your walk is filtered by `formats` from `GET /core/v1/capabilities`, read per scan rather than compiled in.
- You check the credit balance **before** opening an image, and you never send without one.
- You call `complete` when the walk ends, including when it ends badly.
- If you offer a Continue button, a resumed walk uses the search's own `folderRef` and `accountRef` and whatever else you recorded against it - not the state of your screens today.
- You report a deleted reference with `POST /core/v1/images/gone`.
- Your secret is in a secret store, not in a file you commit.
- Your frontend, if you have one, talks to your backend and nowhere else.

### 11a. And check it against something that answers back

Every line above is a sequence you have to get right, and a sequence is a poor thing to check by reading. `pdm-harness` is a program that **plays the core**: it serves the caller's half of this contract on a loopback port, records what your manager sends, and says which of eighteen rules it broke and at which call.

```text
pdm-harness --ui
```

It prints a root, an app id, a secret and a `userRef`. Point your manager's core API configuration at those, start a scan the way a person would, and watch the timeline. It ends by itself and exits 0, 1 or 2, so the same command fails a build.

It does not merely watch. It answers a smaller deployment than any real one, so a hard-coded batch size is wrong in the first thirty seconds; it answers the first item of every batch `cached`, so a manager that sends the bytes anyway is seen doing it; and once, mid-scan, it cuts the event stream and withholds the credit balance from every answer - because a manager that waits to be told about a credit rather than reading the search waits for ever, and that is a defect nobody finds by reading their own code.

PDM's own four managers are driven against it in CI, which is the only thing that stops the implementations we ship drifting from what this guide tells you to do. [`manager-harness.md`](manager-harness.md) is the whole of it.

## Where to read further

| | |
| --- | --- |
| [`core-api.md`](core-api.md) | The one contract, in prose. Section 9 is the ingest path and the format list; section 18 is what a deployment actually serves. |
| [`manager-scan-driver.md`](manager-scan-driver.md) | How PDM's own four managers drive a scan, including the parts that are only obvious after you have written one. |
| [`resource-manager-frontend.md`](resource-manager-frontend.md) | What a manager's own frontend looks like, and the rules a framed one has to keep. |
| [`manager-harness.md`](manager-harness.md) | `pdm-harness`: a program that plays the core so you can check your manager against a socket rather than against a document. |
| [`resource-manager-api.md`](resource-manager-api.md) | The record of the contract that was withdrawn, if you found a reference to it somewhere. |
