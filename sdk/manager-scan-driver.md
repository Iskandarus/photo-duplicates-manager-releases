# The manager drives the scan

> Piece D of [#254](https://github.com/Iskandarus/photo-duplicates-manager/issues/254), shipped as [#276](https://github.com/Iskandarus/photo-duplicates-manager/issues/276). Piece C ([#275](https://github.com/Iskandarus/photo-duplicates-manager/issues/275)) built the core's half - the three ingest operations and the credit window behind them. This is the other end of the same socket: the loop that walks a manager's own resource and feeds them.

## 1. What changed

Before this, a scan was PDM walking a manager:

```text
core --- POST /v1/images:nextBatch ---> manager        (a page of what is there)
core --- GET  /v1/images/{ref}/content -> manager       (one photograph, pulled)
```

After it, a scan is a manager feeding the core:

```text
manager --- POST /core/v1/searches                     ---> core   (open, and start nothing)
manager --- POST /core/v1/searches/{id}/items          ---> core   (declare what the walk found)
manager <-- cached | needsContent | rejected, + credits ---
manager --- PUT  /core/v1/searches/{id}/items/{ref}/content -> core (only the bytes it asked for)
manager --- POST /core/v1/searches/{id}/complete       ---> core   (the walk is over)
```

The deployment consequence is the whole point of #254: **a manager makes outbound calls only.** No hostname the core has to reach, no certificate, no hole in a firewall. That was the single largest requirement [#139](https://github.com/Iskandarus/photo-duplicates-manager/issues/139) imposed on anybody who wanted to write a manager, and it is gone.

**The gateway is still there and still able to walk.** Which direction a search takes is decided by one thing, the `searches:ingest` grant, when the search opens - see section 6. #278 removes the alternative.

## 2. Where the code is

| Piece | SDK (the three cloud managers) | `PDM.ResourceManager.Local` |
| --- | --- | --- |
| The three operations, as a client | `Ingest/CoreIngestClient.cs` | `Ingest/CoreIngestClient.cs` |
| Wire shapes | `Ingest/IngestContracts.cs` | `Ingest/IngestContracts.cs` |
| The credit window | `Ingest/IngestCredits.cs` | `Ingest/IngestCredits.cs` |
| The loop | `Ingest/ScanDriver.cs` | `Ingest/ScanDriver.cs` |
| The scans in flight | `Ingest/ScanDrivers.cs` | `Ingest/ScanDrivers.cs` |

Two copies, and the copy is deliberate. `PDM.ResourceManager.Local` may reference no PDM assembly at all (#149, pinned by `NoProjectReferencesTests`), which is what makes "the published core API is enough" a tested claim rather than an intention. It is the same rule that already gives it its own `CoreApi`, `CoreStreamRelay`, `WorkspaceSessions` and `/workspace` layer.

**The walk itself is not new and is not in either of them.** Every manager already implements `IResourceManagerSource.NextBatchAsync` - the same cursor, the same `isLast`, the same batch of `ImageItemDocument` - because that is precisely what PDM used to call over `POST /v1/images:nextBatch`. What #276 adds is the loop around it and the uploader below it, which is why the diff is a driver and not four rewritten managers.

## 3. The loop

```text
limits   <- GET /v1/capabilities            (once per scan: maxItemsPerBatch, maxImageBytes, formats)
expected <- source.CountAsync(folderRef, recursive, formats)   (else: count by walking)
repeat
    batch <- source.NextBatchAsync(folderRef, cursor, maxItems, recursive)
    for each chunk of maxItemsPerBatch items:
        answer <- POST /v1/searches/{id}/items   (carrying expectedTotal)
        credits.Report(answer.credits)
        for each item answered needsContent, at most MaxParallelUploads at once:
            credits.Acquire()
            open the image, PUT its bytes
            credits.Settle(what the answer said)
until batch.isLast
POST /v1/searches/{id}/complete
```

Six things in it are decisions rather than mechanics.

**The walk is counted before it is made** (#227). PDM's progress dialog needs a denominator and PDM cannot produce one - it is handed one batch at a time and never materialises a folder (#163) - so a scan of a photo library ran for hours showing a number that only climbed. The count goes on every declaration afterwards, which costs nothing because it rides a call being made anyway and is what gets the number to a PDM that restarted mid-scan.

*Before* the walk rather than beside it, on two grounds: the acceptance is that a total is on screen before the first embedding is computed, which counting concurrently does not guarantee; and a count racing the walk against one upstream service is two passes competing for one quota. What it costs is a listing pass with no file opened, nothing hashed and nothing downloaded, in front of a scan that downloads and embeds every photograph it finds.

**A count is never a bound.** Nothing loops until it is reached, no scan ends because of it, and PDM never reports it below what has actually been declared - so a count that turns out low slows the bar down rather than taking it past the end, and one that turns out high leaves the bar arriving late. Every way of failing answers null, and null puts the dialog back to "found N so far": a count that could end the scan it was going to describe would be worse than no count at all.

**Only what will be declared is counted**, by exactly the two tests the walk applies - the manager's own idea of an image, then the format list this deployment publishes (#207). A folder of photographs with a spreadsheet in it counts the photographs, and a file PDM cannot decode is left out, because the number at the end of the walk has to be the number of photographs actually processed.

**The declaration comes before anything is opened, and that is the repeat-scan cache** (#230) asked from the other side of the socket. An item answered `cached` costs no credit, no upload and no further call, so **a library nothing has changed in declares its contents and reads no file at all**. A driver that uploaded first and let PDM decide afterwards would re-send an entire library every time somebody pressed the button. It is the property both test suites pin first.

**`cached` and `rejected` are opposite answers, not two flavours of "skip".** The first means the image is in the search and finished; the second means it is not in the search at all. The driver acts the same way on both - it sends nothing - and says which in its log, because sending bytes for either is a conflict and retrying one is a loop.

**A batch that is not the last and names no cursor fails the walk.** There is nothing to ask the next question with, and asking the same question again would answer the same thing forever, so the walk is completed as `failed` with a line somebody can read rather than spun on.

**The walk is completed even when it went wrong.** `POST .../complete` with `outcome: failed` and a detail is the difference between a search that ends and one that hangs until a cleanup worker calls it interrupted. Two paths complete nothing, and both are searches PDM has already settled: cancellation, and a `search_not_ingesting` anywhere in the feeding. The completion is made on a token of its own with a fifteen-second deadline rather than on the scan's - it is owed even while the process is stopping, but the client it goes on carries the event stream and so has no timeout, and an unbounded wait there would hold the host's shutdown open.

**Progress is not reported at all, and there is nothing left to report it on.** PDM's own reading already counts what has been declared, so `enumeration_progress` would have been saying the same number twice; it was sent only where the registration held `events:publish`, which none of the four built-ins does. Core API 3.0 removed that operation with the notification channel it relayed into (#318), and the driver's announce path went with it. Nothing is owed in its place: the total rides the declaration as `expectedTotal`, so a manager granted nothing beyond ingest still gives the dialog a denominator.

### 3a. How each manager counts

`IResourceManagerSource.CountAsync` answers null by default, which means "nothing cheaper than the walk" - and the driver then counts by walking the source itself, one listing pass through the same `NextBatchAsync` the feeding loop uses. That is what a third-party manager gets for free, and it is already that manager's own paging.

The four built-ins each answer for themselves, because none of the three cloud services has a count operation and the pass is worth writing for counting rather than performing the walk twice:

| Manager | How it counts | What it saves over the walk |
| --- | --- | --- |
| `local` | `ImageWalk.Count` - the same directory walk with no `ImageFacts.Describe` per file | The only expensive part of the walk. A directory enumeration and nothing else |
| Google Drive | The walk's own `files.list` query, `fields=nextPageToken,files(id,name,mimeType)` | Seven fields, including `md5Checksum` and `imageMediaMetadata` - the two that make Drive read more than its index. Pages at `MaxDrivePageSize` |
| OneDrive | `/children?$select=id,name,folder` | Six fields, including `cTag` and the `image` and `photo` facets Graph has to assemble. Pages at `MaxGraphPageSize` |
| Dropbox | `files/list_folder` with `recursive: true` and `include_media_info: false` | The media info the walk turns on to learn dimensions and capture date, which Dropbox documents as the slower answer. Pages at `MaxDropboxPageSize` |

Graph's `folder.childCount` is deliberately **not** used. It counts every child of one folder - subfolders and files that are not images alike - and says nothing about a recursive scan, so it answers a different question: a number that is wrong in a way nobody could see, which is worse than a listing pass.

The page size is the second saving and the less obvious one. A walk's batch is sized for how much PDM will accept in one declaration; a count declares nothing, so it asks for the largest page its own service allows.

**What it costs, measured on a desktop:** 2 to 3 seconds for a local library of 66,000 photographs. That is not merely small, it is **free in wall clock**, and the reason is that the two processes overlap: PDM opens the search and starts loading the embedding model while the manager is still counting, so the count finishes inside a window the scan was going to spend anyway. It is the number that settles the question of counting before the walk rather than beside it - a concurrent count would have bought nothing and cost a second pass competing with the walk for one service's quota.

### 3b. A walk is never asked to descend where the manifest says it cannot (#212)

`recursive` reaches the driver from whoever opened the search, and `canListRecursive` is what this build can actually do. The driver takes the narrower of the two, once, at the top of the run - so both the count and the walk below it are made against the same answer.

The contract has always said a manager without the capability ignores the flag on an enumeration turn, so narrowing it changes what `NextBatchAsync` does for nobody. **The count is why it is done anyway.** `CountAsync` is a separate implementation of a separate question, and a source that counted what is under a folder while walking only the top of it would hand the progress dialog a denominator its own walk can never reach: a bar stuck short of the end for the whole of every scan, on the one screen that exists to say how far along things are (section 3a).

One place rather than at the door, because there are two doors - the route that opens a search and the one that resumes it - and both run through here.

What a person is told is the other half, and it is on the screen before the scan rather than in a log: see section 3e of [resource-manager-frontend.md](resource-manager-frontend.md).

### 3c. A resumed scan is walked with the search's settings, not today's (#312)

Resuming used to re-derive the walk's parameters from **the request that resumed it** rather than from the search being resumed. Of the four things that decide what a walk reads, two came from the right place and two did not:

| | Where it came from on a resume | Right? |
| --- | --- | --- |
| `folderRef` | `opened.folderRef` - PDM's own answer | yes |
| `threshold` | never re-sent; PDM keeps it on the job | yes |
| `recursive` | a literal `true` on the resume route | **no** |
| `accountRef` | the session's *current* choice | **no** |

Two symptoms, one cause. **A deliberate "do not descend" was lost**: untick "include folders inside it", let the scan be interrupted, press Continue, and the walk descended - so the result held photographs from folders somebody had excluded on purpose, on a page that exists to select photographs and delete them. And **a scan could change account mid-flight**: started under account A, resumed while the account bar had moved to B, the walk read B's files into a search PDM had tagged `accountRef: A`, whose stored `imageRef`s then resolved to nothing under either.

The frontend sends no body at all on a resume, so there was nothing it could have passed instead.

**The two halves want different fixes, and the difference is the point.**

`accountRef` is **a read that was skipped**. The answer to a resume is a `Search`, and `Search.accountRef` has been on it since 1.4 - PDM keeps the account on the job, copies it to the result and hands it back on every later call about those images. So the walk's `SourceCall` takes `opened.accountRef` and not the session's, on the way in as well as on the way back: for a scan being started the two are the same value, which is what makes this one rule rather than a branch.

`recursive` **needed somewhere to live**, because no other process has it. It is an instruction to a walk PDM does not perform (#276) and has no concept of a directory to hold anyway (#163) - so the manager remembers it. That memory has to survive a restart, since a restart is what makes a search resumable in the first place: a row in the manager's own database (`ScanWalks`, #229) for the three cloud built-ins, and a small file in this manager's own data directory for `PDM.ResourceManager.Local`, which keeps no database and had never written anything until this.

Four things to keep.

- **A missing record is today's behaviour, not an error.** A search opened before there was anywhere to record this, or one whose record has aged out, resumes descending and under whichever account PDM answers with. Nothing here may be a reason a scan cannot be continued.
- **It is recorded on every drive, not only when a scan is first started.** A resume opens a *new* search - PDM answers with a new `searchId` carrying `resumedFromSearchId` - so the walk that starts is keyed by the new one, and a scan resumed twice would otherwise lose its settings on the second.
- **Resuming under a different account is refused rather than corrected**, and refused *before* PDM is asked to resume anything: a search PDM has already moved back to running with no walk behind it hangs until a cleanup worker calls it interrupted again. Walking the search's own account would also have been correct - it is what the missing-record case does - but somebody who moved the account bar and then pressed Continue is asking for something this cannot give them, and quietly ignoring the control they just used is the worse of the two answers.
- **Null is not a value in that comparison.** On either side it means "the default account, whichever that is now", so a session that has since named the very account the search already ran under is not a disagreement: only two present refs that differ are.

## 4. Credits, from the caller's side

The core says how many images it will take the bytes of now. A caller that honours the number never has an upload refused; a caller that ignores it gets `ingest_credit_exhausted` at byte 19,000,000 of a 20 MB photograph, which is the whole reason credits exist and are not simply a 429 with better manners.

`IngestCredits` keeps two numbers per search: **the last balance PDM reported**, and **how many uploads this process has started and not been answered about**. What it will issue is the difference, floored at zero.

That is deliberately a *lower* bound. A balance taken while some of our own uploads were already counted, minus those same uploads, can only understate what is free - and understating is the point, because the cost of overstating is discovering it at the last byte.

Three properties are worth keeping.

- **Absolute, never a delta.** Every number that arrives replaces the one before it. The channel a pushed balance arrives on drops messages for a subscriber that has fallen behind, and a lost delta is a balance that stays silently wrong for the rest of a scan.
- **A credit is taken before the image is opened.** Opening first would mean reading a photograph off a cloud service in order to find out there was no room for it.
- **The balance arrives three ways and all three are used**: on the answer to every batch, on the answer to every upload, and pushed as `ingest_credits` on `GET /v1/stream`. The first two are what make a driver correct with no stream at all; the third is what stops a starved uploader waiting out a poll interval it did not need to.

`ScanDrivers` therefore **holds the core's event stream open for the length of a walk**, not only while a frame is watching. `CoreStreamRelay.Hold` is that hold, counted rather than a flag, because two searches can be running for one user and the first to finish must not take the second's connection with it. `ingest_credits` is dispatched to the credit window and to no frame: a page has nothing to draw with it, and one that received it would be reading this manager's own pacing.

When the window is empty the driver waits for a change and, if nothing changes within a couple of seconds, **reads the balance off `GET /v1/searches/{id}`**. The push is the optimisation and the read is authoritative, which is the same relationship a progress reading has (#228).

Two bounds apply to an upload and they limit different things: PDM's credit window is what the core will hold, and `MaxParallelUploads` (four by default) is how many files this manager is willing to have open on its own upstream service at once. The smaller of the two is what runs.

## 5. Sending the bytes

An ordinary send is one `PUT` with a `Content-Length` and no `Content-Range`. It carries the whole image, and it replaces any prefix the core happens to hold - which is what makes "retry it whole" a correct recovery and not a special case.

**A length is required.** The core writes into a pooled buffer and closes the content hash only once the declared number of bytes has arrived (#260), so a chunked body is `invalid_request` rather than something it works out as it goes. Three cases, and the first two copy nothing:

| The source answers | What is sent |
| --- | --- |
| a seekable stream (a manager reading its own disk) | the stream, at its own length |
| a stream whose length the source stated (every cloud download) | the stream, at that length |
| neither, or a resume of a stream that cannot be seeked | a buffered copy, at its length |

The buffer is bounded by the `maxImageBytes` the deployment declares, because a source that could make this process allocate without bound is exactly what that ceiling exists to refuse. It is not the ordinary path: **a 4 MB photograph from Drive, Graph or Dropbox is never held in memory on its way past**, which is the same property #260 gave the core's own side.

**Resume is the exception, not the shape.** After a transfer that did not finish, the driver probes with `Content-Range: bytes */<total>` and no body, learns the length of the prefix the core holds, and continues from it where the bytes can be advanced to that offset. Against this disk that is a seek and the second request carries the tail alone; against a cloud service the body is re-read and re-buffered, so the saving is the upload rather than the download. Where a total is not known, the image simply goes whole again.

**Three refusals mean something other than "this photograph is not in the search", and each has its own next step.**

| Answer | What it means | What the driver does |
| --- | --- | --- |
| `401` | the app token lapsed or was voided | forget it, re-open the image, send again |
| `content_mismatch` | the bytes or the range did not agree | continue from the prefix the `Range` header names |
| `search_not_ingesting` | the search takes no more items | end the **walk**, and report nothing |

The first is the one refusal worth retrying that the client underneath cannot retry for itself: an upload's body is a stream already read, so a 401 comes back as an answer rather than being replayed, and without acting on it one token lapsing mid-scan would quietly drop every image left in the batch while the scan finished looking successful.

The second is the refusal the contract *builds* a next step for, and treating it as final would make it the one refusal that never gets one.

The third is not about the image it arrived on. The contract's own words are that a caller which keeps sending after it "is walking a source nobody is waiting for", so the walk stops - and stops **silently**: the search was cancelled or its walk was already closed, and completing it as failed would be a second statement about a fact PDM has settled. A declaration refused the same way says so too, so the distinction reaches a batch as well as an upload.

An image that cannot be delivered after three attempts is **left out of the search and the walk goes on**. A file deleted since the walk saw it, a permission somebody changed, one object an upstream service refuses - none of those is a reason to stop, and the core's own sweep gives up on what never arrives, returning the credit with it. A refusal for want of a credit is a wait rather than an attempt and is bounded separately: a caller that honours its credits never sees one, so twelve in a row means the window and the core disagree, and a disagreement that does not resolve must not hold a credit and an image open for the life of the process.

## 6. Which direction a search takes

**One thing decides it, and it is a grant.** `POST /core/v1/searches` reads `searches:ingest` for the registration when the search opens (#275). A registration that holds it gets a search that is waiting to be fed; one that does not gets the walk PDM still performs over the gateway.

What says so on the wire is `credits`: it is present on the opened search exactly when the search takes items, and absent - `null`, since PDM's serializer writes nulls - when it does not. The manager's `/workspace/searches` route reads that field and starts a walk or does not. A configuration switch here would have been a second answer that no administrator can revoke, and a grant is per registration, which is the granularity that lets one manager be moved back without touching the other three.

All four built-ins are granted it in `PDM.API/appsettings.json`. Taking it off one registration puts that manager back on the gateway's enumeration path with no redeploy - a state to pass through while #277 and #278 land, not one to settle in.

## 7. What did not change

- **`userRef` is PDM's; `imageRef`, `folderRef` and `accountRef` are the manager's** (#170, #229). Reversing the call direction moves the identity boundary nowhere.
- **Nothing readable is declared.** There is no `name` and no `displayPath` on an ingest item, because PDM keeps neither: a caption is asked of the manager on read, and a caption is never a key.
- **A search is keyed by (user, resource manager)** (#163). Pushing does not change what a search is.
- **A manager still never reaches a user's screen through PDM.** `POST /core/v1/events` was the one way in (#161) and ingest was never an exception to it; 3.0 removed that operation with the channel it relayed into (#318), so what a manager tells a person it tells its own frontend, on its own origin.
- **`recursive` stopped travelling to PDM**, and that is a removal rather than a change: it was an instruction to a walk PDM performed, and the walk is the manager's now. It is a parameter of `NextBatchAsync` and of nothing else - which is why a manager that wants to resume a walk the way it was set up has to remember the field itself (section 3c). Sending it to PDM so that PDM could hand it back was considered and refused: a field the core stores and cannot interpret is one more thing on the wire that means nothing to the side holding it.

## 8. What is still owed

- **Reads still go the old way.** A tile's preview, an image's name and a file operation still travel manager -> its own backend -> core -> gateway -> manager. #277 turns that loop inside out.
- **The gateway is gone** (#278), and with it the published resource manager contract and the registry columns that only existed so the core could dial. A driver is the only way a scan happens.
- **`pdm-conformance` retired with the document it checked**, and what replaces it has landed: `pdm-harness` (#194) plays the core on a socket and checks a manager's *calls* rather than its answers. It was a piece of work of its own and deliberately not folded into the retirement. All four built-ins are driven against it in CI - see [manager-harness.md](manager-harness.md).
- **No numbers, and now none to be had against the old shape.** #255 measured the transfer and found it immeasurable (+0.29 ms a photograph at 4 MB against 121 ms for the model), and #260 removed the staged copy that was the real cost. What was never measured is a full library scan driven from this side against one pulled, on a server - and the pulled arm no longer exists to measure, so the claim that the reversal is free rests on #255's per-image numbers rather than on an end-to-end run.

## See also

- [core-api.md](core-api.md) section 9 - the ingest operations, and why a credit is one image
- history.md - the record of what the reversal replaced
- [writing-a-resource-manager.md](writing-a-resource-manager.md) section 5 - the walk a driver drives
- [resource-manager-frontend.md](resource-manager-frontend.md) - the `/workspace` layer the driver starts from
- [manager-harness.md](manager-harness.md) - the program that plays the core, and what it checks a driver against
