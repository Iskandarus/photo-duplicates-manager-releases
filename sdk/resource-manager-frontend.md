# A resource manager that serves its own frontend

What the four internal managers draw, what stands behind it, and the boundaries that make each of them a microfrontend rather than a second copy of the app.

Built for [#248](https://github.com/Iskandarus/photo-duplicates-manager/issues/248), which did the first one, and [#250](https://github.com/Iskandarus/photo-duplicates-manager/issues/250), which did the other three and turned the first one into a consumer. Both are part of the microfrontend architecture [#217](https://github.com/Iskandarus/photo-duplicates-manager/issues/217), itself part of the platform epic [#139](https://github.com/Iskandarus/photo-duplicates-manager/issues/139).

Companion documents:

- `shared-frontend-package.md` - what the app host and a manager's frontend share, and the three things that may never be in it.
- `app-host-frames.md` - the other side of the seam: how the host mounts these four, and where the frame origin comes from.
- `resource-manager-local.md` and `resource-manager-cloud.md` - the four services themselves.
- `history.md` - the folder tree before it moved, the credential it was reached with, and what the microfrontend split left behind.
- `core-answer-path.md` - the stream this progress comes down.
- [`core-api.md`](core-api.md) - what a backend may ask of PDM, and with what.

---

## 1. What was built

All four internal resource managers serve a frontend of their own, each on its own origin, carrying the whole of what a person does with that resource:

- **the search page** - the account where there is one, choosing a folder including the tree, which moves here from PDM; starting a search; watching it; and the saved results of this manager;
- **the result page** - the duplicate groups, selection, hiding, recycling, deleting, moving, and the full-size viewer. The hidden groups are this page with the filter flipped.

Each talks to exactly one thing: the backend beside it. That backend talks to the core API with an app token exchanged from the client credentials its registration was issued (#145), and to nothing else.

```text
browser  ->  PDM.ResourceManager.<X>  ->  PDM core API
   (one origin, one session)     (client credentials, app token per user)
```

There is no second arrow out of the browser, and there is no arrow into a manager from PDM that its frontend uses.

### 1.1 Why `local` first, and what the other three added

`PDM.ResourceManager.Local` is the reference implementation of the contract and the one built-in with no OAuth, no accounts and no upstream service. Everything that is hard about a microfrontend is hard there too - its own origin, its own session, its own realtime, its own result page - and none of the things that are hard about *cloud* are. If that one could not be built without reaching for the core API from the browser, the rule would be wrong, and it was cheap to find that out there.

The other three then prove the same shape against a resource somebody else owns. What is genuinely new in them is the account: a person connects one, may hold several, chooses between them, and has to be told plainly when a saved result belongs to one that is no longer linked. Section 5 is that.

### 1.2 Four frontends, one package

Three frontends that are almost the same is exactly how the four backends looked before `PDM.ResourceManagers.Sdk`, and the answer is the same on both sides of the boundary.

| Side | Shared | Per manager |
| --- | --- | --- |
| Backend | `PDM.ResourceManagers.Sdk` - the contract surface, the two gates, and since #250 the whole `/workspace` layer | the source: what a folder is, what a ref means, which upstream service |
| Frontend | `@pdm/manager-frontend` - the two screens, the session, the API client, the stream, the account bar, the dictionary | `ManagerProfile`: an icon, a fallback title, and the sentences that name the resource |

**They are still four applications.** Four origins, four documents, four content security policies, four frames in the app host, four bundles built and shipped separately inside four services. The package is compiled into each of them the way the SDK is compiled into each service; nothing about it is shared at run time, and a manager that needs something the profile cannot express puts it in its own bundle, which is what four bundles are for.

**Nothing in the package may name a manager**, and that is checked rather than reviewed: `front-end/packages/manager-frontend/test/boundaries.test.ts` fails the build on a literal naming one, on a path into `/core/v1`, on an absolute address, on an import that climbs out of the package and on a runtime dependency beyond what all four already ship. The first rule bites hardest here: a name in this package is a name compiled into the frontends of the three managers it is false for, and the one it is true for renders correctly, so nobody notices.

`PDM.ResourceManager.Local` is the exception on the backend side and only there. It may reference no PDM assembly at all - `NoProjectReferencesTests` fails the build when one appears - so it keeps its own copy of the workspace layer, exactly as it already keeps its own token gates and problem documents. On the frontend side there is no such rule and no such copy: an npm package is not a PDM assembly, and `@pdm/manager-local` consumes both packages like the other three.

---

## 2. The four boundaries

### 2.1 One correspondent, and it is written as a header

Rule 2 of #217: a manager's UI never talks to the core API. Everything the page needs is on its own origin, so the served document carries `connect-src 'self'` and nothing else. A frontend that grew a call to PDM is then stopped by the browser rather than by a review.

That includes the two things it would be most tempting to except:

- **Image bytes.** The core API used to mint a media token precisely so a workspace could put real URLs in `img src` and let the browser do what it is good at. These frontends never did: `/workspace/images/...` is the manager's own route, because a page that reaches two origins has two correspondents. Until #277 that route proxied the bytes through PDM and back to this same manager; it reads the manager's own source now, by the `imageRef` on every image of a result, and the media token is gone from the core API altogether. The entity tag is the file's own either way, so a second visit to a results page is still a page of 304s.
- **Realtime.** Progress arrives on the manager's own stream, fed by the core's (section 4).

There is one thing on the page that is deliberately not a request at all: the OAuth consent screen a connect flow opens (section 5). It is a window, not a frame and not a fetch, so no policy on this document has anything to say about it - which is right, because it is not this page talking to anybody.

### 2.2 The session is PDM's answer, held here

A manager holds a user-facing session, and it is worth being exact about what that is. It is **not** a second identity: PDM is the only thing that can say who somebody is, and a session here exists only because PDM already said so. It is minted from a token PDM signed, naming one user and this registration, and it carries no more authority than that token did.

The handle is **opaque and is not a JWT**. A signed token would need a second secret in the process, and the one secret there is shared with PDM - signing with it would produce something indistinguishable from PDM's own. A random handle over an in-memory table needs no key, is revocable in one line, and cannot be replayed anywhere else.

It is **in memory, and that is the whole of it**. A restart drops every session, every frame re-bootstraps, and authority is re-derived rather than carried across a restart of the process that was holding it.

Three credential classes reach each of these services, and each surface refuses the other two:

| Surface | Credential | Refused there |
| --- | --- | --- |
| `/v1` - the published contract | a token PDM mints per call, no `use` claim | a frame token, a session handle |
| `/workspace` - this frontend | a session handle this manager minted | a contract token, and any token at all on the routes behind the gate |

`POST /workspace/session` is the one route that takes a PDM-minted token, and it accepts two classes: `use: frame`, which is what the app host mints when it mounts a frontend (#249). It used to accept `use: picker` beside it, which is what PDM mints for the folder API it still calls. Both name the same user and are signed with the same registration key. `picker` went with the rest of #211 in #251, and `frame` is the one that remains.

### 2.3 The host-disk gate was asked again, and then stopped existing

The earlier reading of this was that a frontend *inherits* `CloudLocalBrowse` plus the registration's host-filesystem grant. It does not inherit anything by being ours, so the gate was asked again, of the session, on `/workspace/roots` and `/workspace/folders`.

**#254 removed the question.** What protects the host disk was already that there is no path to it for a manager somebody else wrote: such a manager is a separate program with its own storage and no field in which to name a path on our machine. The only thing that read this host's disk was a provider compiled into `PDM.API`, and that is gone; the only thing that reads *a* local disk is `PDM.ResourceManager.Local`, which reads its own machine and is not built for a server at all - no image, no compose service, no publish entry.

So the answer to "should this deployment be running it" is not a permission and not a profile: a server does not have it. That is better than a permission, because a permission can be granted by mistake, and better than a profile, because a profile can be enabled by one.

The scope it used to carry, `picker:host-filesystem`, is minted by nobody and checked by nobody. It was never asked of the three cloud managers either - those routes list the user's own folders behind their own account link (#250) - so the two halves now agree by there being one.

**`picker:browse` was checked beside it and is not (#249).** That one is the administrator's `Browse` grant - a ceiling over what the core would let a manager do to a resource *the core mediated access to* - and these routes are a manager showing its own resource to the person PDM named. A frame token carries no such scope and #254 removes the grant outright, so keeping the check would have blanked every folder tree the day the app host mounted them. `/picker` required it and went with #251, so nothing mints the scope and nothing reads it.

### 2.4 The shapes are the core's, and the words belong to whoever draws the screen

A search, a result, a group and an image come back from `/workspace/*` exactly as `core-api.v1.yaml` describes them, because the backend relays them rather than restating them. Restating would be a third spelling of the same objects to keep in step with two others.

Those types live in `@pdm/manager-frontend` and may never live in `@pdm/frontend-shared`. Rule 4 of #247 is that no wire shape goes in the latter, because two frontends sharing a DTO are two frontends calling the same server - and the app host is not one of these four. Here that is exactly what they are: four frontends and four backends, one relayed contract, and the two halves of each pair written together.

The **dictionaries** split the same way, one level further down. `common` - Cancel, Close, Retry, and the two sentences the shared API client produces - is `@pdm/frontend-shared`'s, because those read identically in every frontend. The search page's strings are `@pdm/manager-frontend`'s, because four managers draw that page. The sentences that name the resource are each manager's own, because "Choose a folder on this computer" is false in three of the four.

---

## 3. What is behind each route

`/workspace` is a named route per core operation. There is no pass-through, no route that takes a core path as a parameter, and no way for a page to reach an operation nobody wrote down.

| Route | What it is |
| --- | --- |
| `POST /workspace/session` | Exchanges a PDM-minted token for a session, and answers what this user may do - the core capabilities, **what this manager can do to its own files** (section 3d), whether this manager reads the host filesystem, and whether there is an account to connect |
| `DELETE /workspace/session` | Ends it |
| `GET /workspace/accounts` | This user's links here, and which one the session is working in |
| `POST /workspace/accounts/select` | Chooses between several |
| `POST /workspace/accounts/connect` | Starts a flow and answers the consent URL |
| `DELETE /workspace/accounts/{ref?}` | Drops one link, or every one of this user's |
| `GET /workspace/connect/done` | Where the browser lands when a flow ends. No session, no claim (section 5) |
| `GET /workspace/roots`, `/folders` | This manager's own folders - the listing `/picker` used to serve, and since #251 the only one |
| `GET/POST /workspace/searches`, `/searches/{id}`, `.../cancel`, `.../resume` | `/v1/searches`. The two that open a search also **start the walk that feeds it** (#276) |
| `GET/DELETE /workspace/results`, `/results/{id}`, `/results/{id}/groups` | `/v1/results` |
| `PATCH /workspace/groups/{id}` | `/v1/groups/{id}` |
| `GET /workspace/images/{imageRef}/thumbnail`, `/content` | **This manager's own source** (#277). Streamed, entity tag from the file itself |
| `POST /workspace/images/describe` | **This manager's own source.** What every image on a page is called and whether it is still there, one call per page |
| `POST /workspace/files/recycle`, `/delete`, `/move` | **This manager's own resource**, then `POST /v1/images/gone` for the references that stopped existing |
| `GET /workspace/stream` | This manager's own event stream (section 4) |

**The answer is relayed as it arrived**, problem documents included, for every route that is a relay. A client that branches on a machine code branches on the core's, which is the only vocabulary that describes what actually happened; a second error table here would say less and drift. The five rows in bold above are not relays at all - they are this manager answering about its own resource, in this manager's own error vocabulary, which is the same table because both documents share it.

### 3a. The images and the deletions stopped being relays (#277)

Three things moved here at core API 2.0, and each one was a loop through PDM back to this same process.

**A tile.** `/workspace/images/{imageId}/thumbnail` called `/core/v1/images/{imageId}/thumbnail`, which dialled `/v1/images/{imageRef}/thumbnail` on this manager - three hops to fetch a file from the program that started the request, and since #254 the core cannot dial anything at all. What made the loop removable rather than merely wasteful is that `imageRef` was on every `ResultImage` already, "so a workspace can join against its own state".

**A caption.** PDM used to make one `images:batchMetadata` call per page of groups and hand back `name`, `displayPath` and `available` on each image. `POST /workspace/images/describe` is that call, made by the page against the backend beside it. It is **per page and never per tile**, which is the whole economy of it: a hundred tiles asked one at a time is a hundred round trips against somebody's API quota, slower than the page they annotate. A tile with no description has **no caption rather than a wrong one** - a name stored anywhere but the manager stops being true the moment somebody renames the file (#229) - and "no description" and "the manager says it is gone" are different states that the screen says different things about.

**A deletion.** The three-way authority check of `docs/core-api.md` section 10 collapsed to one, held by the program that can actually check it. What is left for PDM is the part nobody else can do: a deleted photograph is gone from *every* stored result that names it, and a manager cannot know which those are, so it reports the references with `POST /v1/images/gone` and PDM drops the rows, dissolves any group left below two images and recomputes the aggregates.

Two decisions in that last one are worth keeping.

**A move reports the old reference gone only when the reference actually changed.** 2.0 has no operation that renames a reference, and section 10 records the two honest options. Drive's file ids survive a move, so reporting unconditionally would drop its photographs out of their groups for nothing; `local` and Dropbox address a file by its path, so reporting nothing would leave stored references resolving to nothing and tiles that stop opening with nothing in any log. The manager's own `newImageRef` is the only thing that can tell the two apart.

**A report PDM will not accept is a field on a successful answer, not a status.** The photographs are gone whatever PDM says. What is left wrong is a stored result naming files that do not exist, which without `forgetFailed` on the answer is a page of tiles that cannot open and nothing anywhere explaining why.

There is no `Idempotency-Key` and no `dryRun` on these three. The key belonged to a relayed call that could be retried after a lost response; a person pressing delete twice deletes a file that is already gone, which every source answers `skipped` - and a `skipped` counts as gone, because the requested end state is what is being reported. The dry run existed for an agent with a delete tool and a confused model (#158) and moved to where the delete happens, which is a decision for whoever writes a manager rather than a shape PDM imposes.

**Opening a search is the one route that reads the answer before relaying it** (#276). `POST /v1/searches` opens a search and starts nothing; whether this manager is the one that feeds it is said by `credits` on that answer - present exactly when PDM is waiting to be fed - and a walk is started behind the response rather than in front of it, because a route that waited for a scan would be a request held open for hours. The relayed body is byte for byte what the core answered either way. Cancelling stops that walk before it relays the cancel, since PDM stops accepting items the moment it cancels. See [manager-scan-driver.md](manager-scan-driver.md).

**The app token is per user and cached.** The exchange verifies a secret, which is deliberately slow and deliberately rate limited, so a page of tiles must not spend one exchange each. It is re-exchanged a minute before expiry, because a token that lapses between the check and the call is a 401 on somebody's delete - and a 401 that arrives anyway is retried **exactly once** with a fresh token. A grant change or a restart voids a token before its expiry, which no margin can cover; a second retry would turn a credential that is genuinely refused into a loop against a limiter.

---

### 3b. Choosing which duplicate to keep, which is the manager's and not the core's (#271)

The result page's one bulk gesture used to be "select everything but the first image of each group", where *first* is the order the page came back in. The app host had a panel instead - an ordered list of criteria narrowing each group down to one image - and it was not carried across the split; `history.md` section 2 is the record of that. `SelectionFilters.vue` over `selection.ts` is what replaces it, in `@pdm/manager-frontend`, so all four managers get it at once.

**It is here because the core has no opinion about it.** PDM keeps identifiers and grouping; which of several identical photographs is the one worth keeping is a *policy*, and a policy belongs to whoever draws the screen. Nothing was added to `core-api.v1.yaml` for this and nothing here reaches `/core/v1` - the rules read the page this frontend has already loaded, and the descriptions it has already asked its own backend for.

Four things about a run, each of which is a decision rather than an implementation detail.

**The rules apply in order, and running out is a different answer from choosing.** Each rule narrows what the one before it left. A rule that would empty the set is skipped rather than allowed to decide nothing - the three size filters are filters and not comparisons, so one that matches nobody would otherwise settle the group by matching nothing and the rule after it would never run. A list that leaves more than one candidate is settled by the group's own order, and the panel says in how many groups that happened, because a person who can see it can add one more rule instead of trusting a tie-break nobody chose.

**A group that continues on another page is left out.** Paging counts images, not groups, so the tail of a large group is on the next page (`isPartial`). The largest image of such a chunk is the largest of the part that happened to fit, and selecting the rest of that chunk is a list of photographs to delete chosen by where a page boundary fell. The old panel never had this hazard because the host never had partial groups; the run skips them and says how many it skipped.

**A run covers the page and says so.** That was already true of the host's panel and it said so nowhere, so "keep the oldest in each group" quietly meant the fifty groups on screen. The sentence is drawn from what the run actually did rather than written as a caption, so a page with nothing split says nothing about splitting.

**The four location rules read an opaque string, and there is no parent-folder rule.** Five of the host's twenty criteria were about *where* a file is, and they were written when PDM stored a filesystem path. It stores a reference (#229) - a file id for two of the four managers built here - and the one readable location there is, `displayPath`, is presentation only and has not reached PDM at all since 2.0. This package may name no manager either, so it cannot know one manager's grammar even for its own files. So the four that survive treat the string as opaque: how long it is, whether it contains something, whether it matches an expression. The parent-folder rule split it on separators, which is exactly the parsing the contract forbids, and it is gone rather than approximated. A page whose descriptions did not come back says so, because such a rule then matches nothing and a person would otherwise conclude the panel is broken.

The measurements come from the result and never from a description, although both carry a size and a width. The result's are what the scan recorded and are there for every image; a description is one call that may not have come back, and reading whichever is present would make the same rules answer differently on a page whose captions failed.

### 3c. Two things that happen around an action (#271)

**A batch says it is happening, for the whole of its run.** `BusyOverlay.vue` covers the frame - including this manager's own header, which used to stay clickable while a delete was in flight against the photographs on screen - is announced (`role="status"`, `aria-live="polite"`, `aria-busy="true"`), and takes focus so a keyboard does not go on tabbing through controls the pointer can no longer reach. It is **indeterminate on purpose**: a file operation performs one batch and answers once, at the end, with a count, so there is no per-item progress to draw and a bar would be a claim about the wire that is not true. It covers the frame and not the application, which is also a decision - one manager's batch should not freeze the host's chrome around it.

**Every destructive action is asked about the same way.** `useConfirmAction` is the one affordance; the four screens that each wrote their own `confirm.require` are moved onto it, and cancelling a running scan gains the question it never had. That last one matters more than it looks: a cancelled search is not `interrupted`, so the resume the progress panel offers is not drawn for it, and its reading is forgotten outright (#228) - there is nothing to go back to and not even a number left to look at. Reversible actions are drawn without the danger colour and irreversible ones with it, which is why moving to a recycle bin and deleting permanently do not read alike. It defaults to irreversible, so an action added without thinking about it is asked about in the louder way.

### 3d. Which of the three actions is offered, and who answers that (#294)

The buttons above the groups were gated on `files:recycle`, `files:delete` and `files:move` from `GET /v1/capabilities`. #277 - piece E of #254 - struck all three out of the core API's vocabulary along with the operations they authorised: the core stopped mediating resource access, so a capability there authorised nothing. This page went on asking for them.

**Nothing broke, which is why nobody noticed.** No error, no empty state, no failed request: three computeds that are permanently false render as three buttons that are not there, on a page that otherwise looks whole. For two releases the application whose purpose is removing duplicate photographs offered no way to remove one. It is the same signature as the in-process OAuth path #229 found - code that lost its caller and went on compiling.

**The question moved with the operation, and lands on the only program that can answer it.** The manager owns the file, performs the recycle, the delete and the move (section 3a), and is the one side that knows whether it is able to. So `POST /workspace/session` answers a `files` block beside the core's capabilities:

```json
{ "files": { "canRecycle": true, "canDelete": true, "canMove": true } }
```

Four things about it are deliberate.

**It is read off the manager's own manifest rather than restated.** What a manager claims about itself is claimed once, in the same document `auth.kind` on that answer has come from since #250. The manifest outlives the `/v1` route that serves it, which is a fossil of the withdrawn contract ([resource-manager-api.md](resource-manager-api.md)) and removable on its own.

**It varies per host, which is what makes it un-answerable from PDM's side.** `local` declares `canRecycle` only where there is a bin to send a file to, so on a host without one the button is not drawn at all. No list of grants PDM keeps could have said that, because it is a fact about the machine the manager is running on.

**The advert and the refusal are one reading of one document.** The three `/workspace/files/*` routes check the same manifest before anything is touched and answer 409 `unsupported_capability` for an operation this manager did not declare - a move before it resolves its destination, so that the answer is "does not move files" rather than "no such folder", the second being true and useless. A capability a session advertises and a route does not enforce is a promise nothing keeps: a page held open across a redeploy, or any caller that is not this frontend, walks straight past it into a source that was never asked to serve the operation.

**Hiding a group stayed on `groups:write`.** A group is PDM's own grouping over identifiers, no fact about a file decides it, and that scope still exists and still means what it says. The correction is about resource access, not about everything the page asks the core.

The guard against it coming back is a test rather than a note: `boundaries.test.ts` in `@pdm/manager-frontend` fails the build on any `files:*` literal in the package, precisely because the failure mode is a page that looks fine.

### 3e. A source that reads only one folder, and the screen that now says so (#212)

`canListRecursive` is a real capability and always has been. A manager declares it, the walk honours it, and a manager without it is documented to ignore the instruction and read one level.

It was rendered in exactly one place: a grant toggle in the admin panel, where it read "walk subfolders during a scan, not just one level". The person choosing a folder and pressing **Start** - the only one for whom it changes what they are about to get - was never told. So point such a manager at a library of eight thousand photographs across folders, and the scan reads the top of it, finds two hundred and reports success.

**There is nothing on the screen to notice.** No error, no empty state, no failed request - the same signature as 3d above, and as #229's dead OAuth path. It is worse here than a missing button, because the wrong answer is dressed as a right one: the progress panel counts what has been found, and a converging count of two hundred looks exactly like a small folder all the way through (section 4).

**The answer is the manager's, for the reason 3d's is.** It owns the walk since #276, and PDM has no concept of a folder at all (#163) - it is handed one batch of images at a time and could not tell a subtree from a single directory. So `POST /workspace/session` answers a `scan` block beside `files`, read off the same manifest:

```json
{ "scan": { "canListRecursive": true } }
```

Five things about it are deliberate.

**Declared, and nothing else.** It used to be declared *and* granted - `GrantListRecursive` on the registration, intersected with the manifest - and #278 dropped that column with the five beside it, because a grant is authority over a call PDM no longer makes. "Effective" and "declared" are one answer now, which is why this is read off the manifest rather than out of the core's capabilities.

**The control is not drawn rather than drawn and disabled.** A checkbox that cannot change what happens says the choice is somebody's when it is not, which is the rule 3d follows for the three file actions. What stands in its place is a sentence saying what the scan will cover.

**The wording is about the photographs, not about the capability.** Not "recursion unsupported" but "this source reads only the folder you choose; photographs in folders inside it will not be part of the scan". A person reading it is deciding whether to press a button, not diagnosing the software.

**The result says it too, in the past tense.** A result is opened long after the run that made it, and "why did it only find two hundred" is asked there rather than on the start page. It costs nothing to answer: a manager that cannot descend cannot have descended for any result it holds, so one field settles every one of them. It is **not** said for a scan that simply had the box unticked - that was somebody's own choice and they can make it again, while this is the case where there was no choice and nothing said so.

**Absent is not false.** A backend that answers no `scan` block has not said it cannot descend, and a screen that warned on silence would put a notice on every manager older than this package for a capability most of them have. Only an explicit `false` draws anything.

The other half is in the driver rather than on the screen, and the count is why - see section 3b of [manager-scan-driver.md](manager-scan-driver.md).

**It is latent today and that is the point of fixing it now.** All four built-ins declare the capability and each of the three cloud services lists a whole subtree in one query, so no deployment that exists can reach the bad case. It goes live at the first third-party manager that cannot descend - which is the premise the whole platform rests on, and the least affordable moment for a silent wrong answer, because the person watching will have no idea which side to blame.

## 4. Progress: pushed, and also readable

Three legs, and the two ends of them are the same object.

```text
scan  ->  SearchProgressSnapshots  ->  GET /core/v1/stream  ->  the manager  ->  GET /workspace/stream  ->  the page
                    |
                    +-------------->  GET /core/v1/searches/{id}  (the same reading, read)
```

**Server-sent events on both legs.** #217 drew the second one as a hub, written before the core's leg existed; #246 then chose server-sent events for that one, on the grounds that it needs no handshake, no negotiate step and no framing library. Both arguments hold here and one more does: a relay whose halves are the same shape has nothing to translate, so a reading reaches a screen as the object `GET /v1/searches/{searchId}` answers with. Three producers of one number stay one number, which is what #228 exists to keep.

**A connection is held only while somebody is watching.** The core's stream is opened when a frame subscribes and released when the last one goes. Publishing to nobody is what #246 refuses upstream, and holding a socket for a frame that is not open is the same waste one hop later. A scan with no screen in front of it keeps running and stays readable - which is why nothing is lost by not listening.

**Nothing is buffered for a frame that is not connected**, and **a slow frame is dropped from and told**. The queue in front of each frame is bounded; a write that does not fit marks that subscriber, and the next thing it successfully receives is a `resync`. Silently skipping would leave a page showing a number that stopped moving, which is indistinguishable from a scan that stalled. The answer to a resync is the same one it is upstream: read where things stand.

**The read is authoritative and the stream is an optimisation.** The page reads on mount, after starting a search, on regaining visibility and on any resync. That is safe precisely because a reading is absolute state rather than a delta: re-applying one cannot double-count. There is no timer - a page that polls a finished scan every two seconds keeps a laptop awake to learn nothing.

**What is done is shown in both branches** (#271). Until the walk closes nobody knows the total, so there is no denominator and the bar is indeterminate (#163) - but `processedItems` is in the same reading either way, and the panel used to withhold it for exactly as long as the walk was open, which on a real library is most of a scan. So the words are "N processed, M found so far" while it is open and "N of M" once it has closed. The denominator being unknown is not a reason to hide the numerator, and "found M" on its own is a number that climbs while reading like a total - which is how a correct screen got reported as a wrong one.

The one exception to "the session travels in a header" is this route: an `EventSource` cannot set one, so the handle goes in the query string. The alternative is a cookie, which is ambient authority inside a frame's browsing context and is what #164 rules out. It is the same trade the core API makes for a media token, bounded the same way - `WorkspaceGate` names the one path where it is accepted and checks the path rather than trusting it to stay on one route.

### 4a. Duplicates while the scan is still running (#295)

Duplicates used to appear on the results page **while the search was still running**. That was the live preview, and it shipped years ago. It stopped working when this screen moved to the manager's own origin, and nothing decided that it should: PDM went on computing a snapshot of the groups it had formed roughly once a second and went on broadcasting it as `ReceiveDuplicateGroups` on its own notification hub - which a page on this origin cannot subscribe to. So the work was done, the message was addressed to nobody, and a person watching a scan of a hundred thousand photographs saw an empty screen for the length of it. **Same shape as #294 and #271**: a consumer moved and a producer did not.

The four legs are the ones section 4 already draws, with a second store beside the first:

```text
grouping stage  ->  SearchPreviewSnapshots  ->  search_groups on GET /core/v1/stream  ->  the manager  ->  the frame
                            |
                            +-------------->  GET /core/v1/searches/{id}/groups  (the page, read)
```

**The push says when, and the read says what.** `search_groups` carries how many groups and how many images there now are and not the groups themselves. A snapshot of a large library is thousands of references, that connection drops for a subscriber that has fallen behind rather than waiting for it, and the screen renders one page - so putting the grouping on the wire would spend the whole drop budget on the one message whose content a page is going to read anyway. It is the same division `search_ended` and `GET /v1/searches/{searchId}` already have.

**Which makes the panel work with no stream at all.** It reads when it starts following a search, on regaining visibility, on a resync, and whenever a push says the count moved - and the count comparison is what makes the common second free, because the grouping stage reports every second and most of those seconds add nothing.

**A snapshot replaces and never accumulates.** Each one is the whole grouping as it stands, so re-reading cannot double-count and an arrival out of order costs nothing. Group ids are stable across the snapshots of one search, so this is a redraw rather than a remount and a tile does not re-fetch its photograph once a second.

**A live tile is read-only, and that is not a simplification.** PDM mints an image's identifier when the transaction that saves the result writes it, so a live group carries the manager's own reference and nothing else: there is nothing yet to select, delete, move or hide. `ImageTile` therefore takes a `readonly`, which removes the checkbox and the click, rather than being given handlers that do nothing - a checkbox that ticks nothing is worse than the absence of one. The captions are the same one call a finished page makes (`POST /workspace/images/describe`), asked again only when the page actually changes.

**The panel is shown only while a search is running**, and disappears when it ends. The finished result is a screen of its own where every photograph can be acted on; the same photographs twice, one copy of which does nothing when clicked, is worse than the panel going away. What makes that honest rather than a swap is that **the set streamed is the set the result holds**, plus whatever the last batch added: both are built from the same pairs, and the last snapshot is taken before the result is persisted rather than after.

**`ReceiveDuplicateGroups` is gone**, with `DuplicateGroupsPreviewResponse` and the dispatcher method behind it. #229's lesson is that nobody writes down when a producer loses its last consumer, so it was retired here rather than left addressed to a hub with no listener.

---

## 5. The account, which is the whole of what the three added

Everything here is between the page and the backend beside it. The links are rows in that manager's own database (#229), the OAuth flow is that manager's end to end, the tokens never leave the process, and the only thing PDM is ever told is the opaque `accountRef` a scan is tagged with.

### 5.1 Where the browser comes back, and what re-reads the status

Issue #217 raised this and left it to #250, because accounts first appear there. #148 had settled that **the browser comes back to PDM, not to the manager**: `auth/start` was handed a `returnUrl` on our own origin carrying a one-time Data-Protection state, and PDM re-read `auth/status` on return rather than believing the redirect. The property worth keeping was that second half - *a manager cannot assert a connection it did not make*.

**Decided: the OAuth flow is the manager's, end to end.** It is responsible for the external resource, so it owns connecting to it - the consent screen, the redirect, the callback and the token, all on its own origin. #254 makes that the only possible answer rather than a preference: the core knows nothing about a manager and cannot call it, so there is no `auth/status` for PDM to re-read on return.

**The #148 property dissolves rather than moves.** The core no longer holds a connection state to assert anything about, so there is nothing for a manager to lie to it about. Whether an account is linked is a fact the manager keeps and shows in its own frontend.

What replaces it, on the side where a claim could still be made, is stronger than a re-read: **the page believes nothing the returning window says either.** `GET /workspace/connect/done` carries no session, makes no claim of success, and does exactly two things - post a message saying the window is finished, and close. The list is then read again from the backend, which is the only side that knows. A page that asserted "connected" would be asserting something nothing could check.

Its one inline script is allowed by a **nonce minted for that response**, not by widening the served policy. `script-src 'self'` is what keeps an injected script out of the frontend on that origin, and a page that relaxed it for one convenience would relax it for everything there.

The consent screen itself is opened as a **window and never a frame**. An authorization server sets `X-Frame-Options` precisely so it cannot be framed, and a person being asked to hand over access to their photographs has a right to see the address bar it is being asked in.

**On the desktop that window is the system browser's, and this page never holds it.** wry subscribes to WebView2's new-window event unconditionally, so a shell with no handler answers it without creating anything and `window.open` returns null - indistinguishable here from a pop-up blocker, which is what connecting an account used to report, every time, with nothing a person could do about it. The facade opens `https` addresses externally and creates no webview (`src-tauri/src/external.rs`), which is also the only thing that can work: Google refuses OAuth in an embedded webview outright.

So a flow has **two shapes and the page does not try to tell them apart**. With a window of its own it watches that window, as above. Without one - the desktop always, a browser with a blocker - it publishes the address so a person can open it themselves (a link is a navigation, which no blocker refuses) and asks the backend every couple of seconds whether a link appeared, because that is the only side which can see the end of a flow this page is not holding a window for. Ten minutes of waiting, then `blocked`: long enough for a sign-in, a consent screen and a second factor on somebody else's service.

### 5.2 Choosing between several

The `accountRef` is contract 1.2's and is the manager's own opaque value - like `imageRef` and `folderRef`, and unlike `userRef`, which names a user. Three things about how a choice is made:

- **It rides the session, not every request.** It is a mode a person is in, and a page that had to thread it through every call would eventually forget one - which is a scan silently started against a different drive.
- **A ref belonging to nobody is refused, not replaced.** `POST /workspace/accounts/select` checks it against this user's own links and answers `forbidden` otherwise, and the refusal leaves the session working in nothing rather than in something else. That is contract 1.2's rule, enforced where a person actually chooses.
- **A choice whose account has gone is forgotten on read.** Somebody disconnecting a link in another tab would otherwise leave a session pointing at it, and the next scan would be refused three services down.

### 5.3 Which account a result belongs to

Core API 1.4 (#250) carries `accountRef` on `StartSearchRequest`, on `Search` and on `Result`. PDM keeps it on the job, copies it onto the result and hands it back to the manager; it never learns what an account is. See section 7 of [`core-api.md`](core-api.md).

That is what makes the result page able to say three different things where it used to have two, and they must not read alike:

| State | What it means | What a person can do |
| --- | --- | --- |
| `differentAccount` | the result was scanned under an account that is no longer linked | connect **that** account again; connecting a different one will not open these photographs |
| `disconnected` | there is no usable link here at all | connect an account |
| `availabilityChecked: false` | nobody could be asked just now | nothing; the page is showing what was true at the last scan |

Telling somebody to reconnect when the first one is true sends them to fix something that is already fine. That is the defect this field exists to make impossible.

---

## 6. Credentials a built-in can be handed

A manager's backend reaches the core API with the client credentials of #145 - which, until #248, only an administrator clicking a button in the admin panel could produce. On the desktop there is no administrator and no panel worth opening for it.

So `ResourceManagers:{id}:Credentials` is a seam: the seeder provisions what it names, PDM keeps the hash and never the value, and **a pair that already verifies is left alone** - writing it again would issue a fresh grant stamp on every restart, and a fresh stamp voids every token in flight, including the manager's own, seconds after it started.

The desktop facade mints a pair per boot for every manager whose bundle carries a frontend - all four since #250 - and hands the same value to both processes, exactly as it already does for the signing key. That makes three credentials with three directions:

| Credential | Direction | Where it lives |
| --- | --- | --- |
| The client secret (#145) | a manager authenticates **to PDM** | hashed on the registration; the value only in the manager |
| The signing key (#146) | PDM authenticates **to a manager** | `ResourceManagerClient:SigningKeys:{id}` and `{Section}:SigningKey` |
| A frame token (#249) | one user's browser authenticates **to a manager** | minted per session, never persisted |

A fourth thing is not a credential and belongs in the same paragraph anyway: **`frame-ancestors`**. A manager's document names the origins that may embed it (`ResourceManagerOptions.FrameAncestors`, `'none'` by default), and the app host's `frame-src` names the origins it will mount. A browser enforces both, so a blank panel is one of the two - see `app-host-frames.md` section 3.1.

Each registration is granted the core scopes its own frontend needs and no others. `export:read` and `import:write` are absent because none of these frontends offers either, and `events:publish` is absent because what a manager learns it tells its own frame rather than asking PDM to relay - a grant with no consumer is exactly what "a grant is always something somebody wrote down" is meant to prevent.

**Absent means there is no workspace, and that is not a failure.** A deployment that configures none of this runs the manager exactly as before - it serves the contract, PDM dials it - and `POST /workspace/session` answers 503 with the reason. That is the ordinary state of a manager that was only ever meant to be dialled.

---

## 7. How they are built and served

Four npm workspaces under `front-end/managers/`, named `@pdm/manager-local`, `@pdm/manager-googledrive`, `@pdm/manager-onedrive` and `@pdm/manager-dropbox`, each bundling `@pdm/frontend-shared` and `@pdm/manager-frontend` through `node_modules` like any other dependency.

`@pdm/manager-frontend` is consumed **as source**: its `exports` point at `src/index.ts`, each manager's vite build compiles its `.vue` files, and `vue-tsc` type-checks them as part of that bundle's program. There is no build step for it and nothing to rebuild before `npm run dev` in a manager.

`npm run build:managers` in `front-end/` writes each bundle straight into that service's `wwwroot`, which its csproj already copies to its publish output. Every packaging path therefore picks them up for free **provided they are built first**, which is ordered rather than noticed in all four places:

- `asp-back/cicd/server/publish.ps1` and `publish.sh`, before the publish loop;
- `asp-back/cicd/desktop/build-desktop.ps1`, before the publish loop rather than beside the installer, because `-BackendOnly` exits before the installer steps and still needs it;
- `.github/workflows/build-and-publish.yml`, before it publishes.

The four `cicd/docker/Dockerfile.resource-manager-*` build **no** frontend and say so: that would mean node in those images and the repository root as their build context, for bundles every other path already produces. A build with no bundle in the context is not an error - it produces a manager that serves its API and no pages, which is exactly what a deployment that only wants a resource manager asked for.

The bundles are gitignored. A committed one would be a second copy of the source that nobody rebuilds.

**Nothing new serves them.** Each manager's own service does, out of its own `wwwroot` on the port it already has - on a server and on the desktop alike, where the alternative would have been four more processes in a facade that already spawned nine at the time (eight since #318). That is #252, and the argument is section 2.1's header rather than the cost: a page served from anywhere other than its own backend has a second origin to name in `connect-src`, which is the one thing that policy exists to forbid. It also decided what happens to a bundle built without its pages - see `desktop-process-model.md` section 2a.

Since #265 the app host is a workspace member beside these four rather than the tree they sit in, so nothing here is a guest of anything - see `frontend-workspace-layout.md`.

Each served document carries a closed content security policy of the same shape as #164's, with the differences section 2.1 describes. `frame-ancestors` comes from `{Section}:FrameAncestors` and defaults to `'none'`: a build nobody has said should be embedded refuses to be.

### 7.1 Seeing one while you are working on it

Worth writing down because it spans five processes and is the one thing about this that is not obvious: **`npm run tauri:dev:front` mounts no frames at all, and that is not a fault.** That script sets `PDM_FACADE_SKIP_BACKEND=1`, which makes the facade skip its whole boot sequence - it starts no services and injects no configuration - so nothing announces a `UiOrigin`, `GET /resource-managers/frames` answers with an empty list, and the app host draws its own screens exactly as it did before any of this existed. The same is true of `npm run dev` against an API started from an IDE.

Three ways to actually look at one, cheapest first.

**The frontend on its own.** `npm run dev --workspace @pdm/manager-<id>` serves the page on its own port and proxies `/workspace` to that manager on its loopback port, so what the code does in development is what it does when it is served for real. It needs a session and there is no host to hand it one, so it takes one from the fragment: `http://localhost:531x/#token=<a frame token>`. Mint one at `POST /resource-managers/{id}/frame/session`.

**Mounted in the app host.** Most of it is already configured. `asp-back/PDM.API/appsettings.Development.json` carries the four `UiOrigin` values, the `UiOriginAllowed` grant and each manager's client credentials; each manager's `Properties/launchSettings.json` carries the matching `Core` settings and the shell's development origin as `FrameAncestors`. Both are files no build reads, which is why a development origin may be named in them and must not be in any `appsettings` file that ships - the same rule `shell-content-security-policy.md` states for the policy itself.

`FrameAncestors` is the half people forget, and its symptom is a panel that is blank with a console message and nothing in any log: the host's `frame-src` allows the frame and the manager's own document refuses to be one. Both documents have to agree - see `app-host-frames.md` section 3.1.

**The one thing you supply is the signing key**, because it is a secret and it is a pair. PDM signs with `ResourceManagerClient:SigningKeys:{id}` in its own gitignored `appsettings.Local.json`, and each manager verifies with the `SigningKey` of its own section in its own gitignored `appsettings.Development.json`. The two have to be the same value:

```json
{
  "GoogleDrive": {
    "SigningKey": "<the same value as PDM's ResourceManagerClient:SigningKeys:googledrive>"
  }
}
```

It is deliberately not in the launch profile, and that is worth knowing rather than rediscovering: an environment variable beats every file, so a fixture there silently overrides the key you set beside it and every call from PDM fails signature validation - a 401 on `POST /workspace/session`, four blank panels, and both sides individually correct. The manager says which check failed and names both settings, which is what that log line exists for.

**The whole desktop.** `npm run tauri dev` with `PDM_BACKEND_DIR` pointing at a published backend is the only path where none of the above is needed, because the facade mints the credentials, picks the ports and announces all four itself. Build that backend with `asp-back/cicd/desktop/build-desktop.ps1 -BackendOnly`, which runs `npm run build:managers` first.

A backend published before #248 landed carries no bundles at all, and since #252 those managers are **absent rather than blank**: the facade looks for the same `wwwroot/index.html` the manager itself looks for, announces nothing for a manager that has none, and the host draws its own screens for it. So a stale `PDM_BACKEND_DIR` now reads as "no frames" instead of four panels that load nothing, and the repair is the same either way - publish again.

---

## 8. What is still not done here

- **Removing the `Browse` grant.** That is #254's, and its first piece has already taken the compiled-in local provider - `PDM.ResourceManager.Local` is what serves a local source now, everywhere, and the core opens no photograph on any disk.
- **Reversing the data path.** #254 makes a manager upload its own bytes instead of answering `images:nextBatch`; #255 measured what that costs and found the socket under a millisecond a photograph. The scan half landed in #275 and #276 and changed nothing on these screens, as expected: what changed is what "start a search" does underneath them. **The read half did change them** - #277 moved the previews, the captions and the three file operations off the core, which is section 3a. #278 finished the reversal by retiring the gateway, the contract and the registry columns behind them, and changed nothing on these screens: nothing here ever called any of it.
- **Retiring what these leave unused.** That is #251, and it needed all four moved, which is now true.
- **Removing PDM's own result list.** #217 says the main page shows no results at all; that is the host's screen.
- **A selection over the whole result.** A run covers the page it is on (section 3b), which is the honest scope for something computed in a browser out of the page it holds. Doing it over a whole result means paging every group through this backend or asking PDM for an answer, and the second of those puts a selection policy in a core that has no opinion about one. Nobody has asked for it yet; a person cleaning up a two hundred page result would.
