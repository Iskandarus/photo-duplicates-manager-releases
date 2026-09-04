# The harness that plays the core

> [#194](https://github.com/Iskandarus/photo-duplicates-manager/issues/194). A program that answers the caller's half of `core-api.v1.yaml` on a socket, so a resource manager can be driven against it and told what it got wrong. `pdm-harness`, in `asp-back/sdk/harness/`.

## 1. Why the direction reversed

Before [#254](https://github.com/Iskandarus/photo-duplicates-manager/issues/254), a resource manager was a **server**: PDM dialled fourteen operations on it, and a conformance tool asked whether its answers matched a schema. That was `pdm-conformance`, and [#278](https://github.com/Iskandarus/photo-duplicates-manager/issues/278) retired it with the document it read.

A manager is a **client** now. It exchanges its credentials for an app token, reads what the deployment will accept, opens a search, walks its own resource, declares what it found, sends the bytes PDM asks for, honours the credits it is given, follows the answer path, and reports the files it has deleted. Nothing about it is served to PDM at all - so there is nothing left to validate a *response* against, and everything to validate a *sequence* against.

That is a harder thing to get right and a worse one to get wrong. **A manager that answers a schema badly serves one broken call; a manager that ignores credits floods a core that has one model and finite memory.**

## 2. What it is

One process, and the whole of it is an HTTP server:

```text
your manager  --- POST /v1/tokens/app ------------------->  pdm-harness
              --- GET  /v1/capabilities ----------------->
              --- POST /v1/searches -------------------->
              --- GET  /v1/stream (held open) ---------->
              --- POST /v1/searches/{id}/items --------->   cached | needsContent | rejected
              --- PUT  .../items/{ref}/content --------->   200, 202, 409, 429 ...
              --- POST /v1/searches/{id}/complete ------>
              --- POST /v1/images/gone ----------------->
```

Every refusal it makes is one PDM would make, with the same status and the same machine code from the contract's `ErrorCode` table. A manager that has learned to handle this has learned to handle the engine. What is added is beside the answer and never inside it: each call goes onto a transcript, and the checks it settles are decided in the endpoint that saw it.

**There is no pipeline behind it.** Nothing is decoded, embedded or grouped, and there is no result, group, export or import operation. Those are reads over a grouping this program does not compute, and a harness that invented one would be checking a manager against a second implementation of PDM rather than against the protocol.

## 3. Getting it, and running it

It ships with the SDK: the same release, the same version, one download beside the other.

```text
pdm-harness-<version>-win-x64.zip
pdm-harness-<version>-linux-x64.tar.gz
pdm-harness-<version>-osx-arm64.tar.gz
pdm-harness-<version>-osx-x64.tar.gz
```

**One self-contained file.** No .NET to install, no restore, nothing beside the executable - which is the same claim section 6 makes about the source, arriving at the reader. On macOS and Linux, unpack and `chmod +x pdm-harness`; neither archive format carries the executable bit through every producer, so it is said rather than assumed.

**The version is the SDK's**, which is the core API's plus a publication counter: `3.0.4` is a harness that serves core API 3.0, out of publication 4 of the documents that describe it. A number of its own would be a second number about the same contract - [`version-compatibility.md`](version-compatibility.md) section 2 - and the practical value is that a tool and a guide downloaded from one release cannot disagree about what they are. `pdm-harness version` answers with it; `SdkBundleTests` fails the build if the version it serves and the contract it is published beside come apart.

```text
pdm-harness [run] [options]     play the core and check a manager's calls
pdm-harness checks              what it checks, and why
pdm-harness version             the SDK version this build was published at
```

It prints what your manager needs and then listens:

```text
    core API root   http://127.0.0.1:8787
    version served  3.0
    registration    harness
    app id          harness-app
    app secret      6f1c...
    userRef         harness-user-0001

    credits         2 images in flight at once
    items per batch 3
    max image bytes 8388608
```

Point your manager's core API configuration at that root, with those credentials, and start a scan the way a person would. The run ends by itself, four ways, and each is reported rather than inferred: the walk said it was over, nothing was heard for the stall window, the time limit was reached, or you stopped it.

**The defaults are deliberately mean.** Two credits and three items to a batch is a smaller deployment than any real one, and that is the point: a manager that reads `GET /v1/capabilities` and honours the credit balance behaves identically at three items and at five hundred, while a manager that hard-coded either is wrong here in the first thirty seconds instead of in somebody's library in six months.

| Option | |
| --- | --- |
| `--url <address>` | where to listen. `http://127.0.0.1:0` binds a free port |
| `--app-id`, `--app-secret`, `--user-ref` | the credentials to issue. The secret is minted per run unless you name one |
| `--credits <n>` | images in flight at once |
| `--items-per-batch <n>` | `maxItemsPerBatch` |
| `--max-image-bytes <n>` | `maxImageBytes` |
| `--credit-hold-ms <n>` | how long a spent credit stays spent - see section 5 |
| `--wait`, `--stall`, `--settle` | the three deadlines, in seconds |
| `--no-resync-drill`, `--no-cache-drill` | turn off the two things the harness does on purpose |
| `--json <file>` | the report, for a build to read |
| `--ui` | serve the timeline on the same address |

Exit codes: **0** passed, **1** failed, **2** nothing was heard, **64** bad arguments.

## 4. The two halves, and why both

The issue kept both interfaces from the version of it written years earlier, and the reason for each is now written down rather than assumed.

**The command line is what fails a build** - ours and yours. It takes a time limit, writes a report something else can read, and sets an exit code.

**The interface is for somebody debugging their own manager**, and here it is not decoration: what is being checked *is a sequence*, so watching it unfold - the token, the allowance, each batch against its credits, the drop and the reconnect, the deletions - is the diagnosis. A log of a failed assertion tells you what broke; a timeline tells you where you diverged.

**It is a page, and the argument for that is the same one [#252](https://github.com/Iskandarus/photo-duplicates-manager/issues/252) made about hosting a microfrontend.** A native interface would be three builds, three toolchains, three installers and a signing story, for a program whose whole claim is that you download one file and run it - while this process is already an HTTP server, so a page costs one route and runs on every platform a browser does. The cheapest answer and the most portable one are the same one.

It is one listener rather than two. The manager calls `/v1` and nothing else exists for it; the page, its two assets and its feed are under `/` and `/harness`. A second port would be a second origin, a second thing to configure and a second thing to explain.

## 5. What it checks

`pdm-harness checks` prints the table with its reasons. In the order a manager meets it:

| | |
| --- | --- |
| `token.exchanged` | the credentials are exchanged before anything else is called |
| `token.credentials` | as HTTP Basic, with the pair the registration was issued |
| `token.userref` | naming the `userRef` this registration was handed |
| `token.presented` | and the minted bearer travels on every call afterwards |
| `limits.read` | `GET /v1/capabilities` is read **before** the first declaration |
| `limits.batch` | no declaration is over `maxItemsPerBatch` |
| `limits.imagebytes` | no upload is over `maxImageBytes` |
| `search.owned` | every ingest call names a search this caller opened |
| `search.completed` | the walk says it is over, whether it finished or failed |
| `search.closed` | and nothing is declared or sent after it did |
| `ingest.declared` | an image is declared before its bytes are sent |
| `ingest.decision` | bytes go only for an item answered `needsContent` |
| `ingest.content` | an upload states its length, and sends what was declared |
| `ingest.credits` | an upload is never started without a credit for it |
| `stream.held` | the answer path is held open while a walk is feeding a search |
| `stream.resync` | a subscriber that has to reconcile **reads**, rather than waiting to be told again |
| `gone.performed` | a reported deletion had happened before it was reported |
| `gone.shape` | a report carries references, in one call, within the batch size |

### 5a. Three states, and the third is not a pass

A check that never had the chance to fire is not a check that passed. A run against a manager that never opened the stream has said nothing about how that manager reconnects, and reporting it green is how a harness teaches somebody to trust a result it never produced - so the report has `ok`, `FAIL` and `-`, and the exit code has a third value for a run nobody called at all.

It is the same lesson [#331](https://github.com/Iskandarus/photo-duplicates-manager/issues/331) learned about a skipped benchmark arm, in the one other place in this repository where "it did not run" and "it was fine" look identical.

### 5b. Credits, and why a credit is held for a moment

The centre of it. A credit is one image, held from the first byte until the pipeline is done with it, so the balance bounds what is **in flight** rather than what is per second. The contract's own sentence is that *a caller that honours the number never has an upload refused* - so a `429 ingest_credit_exhausted` here is the number being ignored rather than a race, and it is failed as such, naming the upload, the balance and how many were in flight.

The harness has no embedding stage, so it holds each spent credit for `--credit-hold-ms` after the upload that spent it was answered. That stands in for the pipeline - an image occupies its credit until the model is done with the bytes, long after the socket - and it is what makes the check able to fail at all: a harness that returned the credit the instant the last byte landed would have a balance nothing could exceed, and would pass a manager that sends its whole library at once.

### 5c. The reconnect drill

The contract says nothing is buffered for a subscriber that is not connected, and that the `hello` frame's `resync` says whether what you already believe can be trusted. What a manager is supposed to do about it is **read** - `GET /v1/searches`, or the search itself - and carry on from what it says.

Waiting for that to happen by accident would check nothing, so the harness provokes it. Once, after an image has landed and while the walk still has images to send, it:

1. cuts the stream;
2. reports a balance of **zero** on every answer afterwards;
3. pushes no `ingest_credits`.

The true balance is on `GET /v1/searches/{id}` and nowhere else. A manager that reads it carries on and passes; a manager that waits to be told stalls until the run gives up, and the failure names the step the connection was cut on. It is the only thing the harness ever says that is not true, and the lie is corrected the moment somebody asks properly.

The drill arms only where a connection was actually open when the image landed. A manager that holds no stream at all is **reported rather than failed**: reading is authoritative and the push is the optimisation, so polling is correct and slower.

### 5d. The deletion check, and its one honest form

PDM dissolves a duplicate group for a file reported gone. A manager that reports first and then fails to delete has produced a photograph the user believes is gone and is not.

**The core cannot watch a filesystem**, so there is exactly one form of evidence available on this side of the socket - and it is real evidence: a manager's own walk can only declare a file that is there. So a reference reported gone and then **declared again** says plainly that it was not gone when it was reported.

That makes the drill a two-part thing you do rather than something the harness can force: delete some photographs through your own application, let your manager report them, and rescan the same folder. A run where nothing was reported leaves the check at `-`, and a run where something was reported and the walk that followed did not name it again is a pass.

### 5e. The cached drill

The harness answers the first item of every batch of several `cached`. It is entitled to - it is the core, and `cached` means "I already hold an embedding for this content version" - and it is the only way a single walk ever meets that answer at all.

`cached` and `rejected` are opposite answers and neither wants bytes: the first means the image is already in the search and finished, the second that it is not in the search at all. A manager that sends anyway has thrown the repeat-scan cache away, which in a library nothing has changed in is every transfer in the scan.

## 6. What it will not become

**No `ProjectReference` on any PDM assembly.** That was `pdm-conformance`'s rule under [#151](https://github.com/Iskandarus/photo-duplicates-manager/issues/151) and it is the most valuable thing carried across: a third party has to be able to check their manager without building PDM. It is also what decides the next argument rather than leaving it open - **if the harness turns out to need something the core API does not publish, the contract is wrong rather than the rule**, because a harness that reached inside for one fact would be checking managers against something no implementer can see.

**No package reference either**, which is the second half of "download it and run it": Kestrel, minimal APIs and `System.Text.Json` are the framework. A JSON schema validator, an argument parser or a logging package would each be a restore standing between somebody and the first run of a tool they are trying to debug their own program with. The published artifact keeps the promise the rule makes - a self-contained single file, so the framework is not a restore either.

**Not a revival of `pdm-conformance`.** The old suite read a document and evaluated responses against its schemas; this one plays a server and evaluates a conversation. Converting it would have carried the old shape into the new job.

## 7. Ours passes it too

The old suite ran against our own built-in for a reason that has not changed: nothing else stops the reference implementations drifting from what a third party is told to do.

Two arms, because there are two implementations and not four:

| | |
| --- | --- |
| `PDM.ResourceManagers.Cloud.Tests/HarnessConformanceTests.cs` | the three cloud managers, which share `ScanDriver` in the SDK |
| `PDM.ResourceManager.Local.Tests/HarnessConformanceTests.cs` | `PDM.ResourceManager.Local`, which has its own copy because it may reference no PDM assembly (#149) |

Both drive a real manager over a real socket against a real harness; only the **upstream** half is stubbed, because no build server can reach somebody's Google Drive. Both projects are in the solution, so `dotnet test` runs them per pull request with nothing added to a workflow.

The local arm is the one whose result means the most. That manager consumes no SDK of ours, so a run that passes there is a claim about the published contract - that somebody who has never cloned this repository can write a conforming caller from it - rather than a claim about code the other three share.

## 8. What is deliberately not checked

- **A response's shape against the document's schemas.** There are no responses to check: a manager serves PDM nothing. Where a *request* is malformed the harness refuses it the way PDM would, which is the same fact from the only side it still has.
- **Anything about a result.** No grouping is computed here, so there is nothing to page through and nothing a manager could get wrong about it that this program could see.
- **Rate limiting.** The contract has an allowance per caller and per user; a harness that enforced one would be failing managers for a number a deployment chooses, on a machine where the number is invented.
- **A manager's own frontend.** `/workspace` is the SDK's shape rather than the published contract, so a manager written in Go has no such surface and a check of it would only ever pass for ours.

## See also

- [writing-a-resource-manager.md](writing-a-resource-manager.md) - what to build, and the contract it calls
- [manager-scan-driver.md](manager-scan-driver.md) - the walk this harness watches, from the manager's side
- [core-api.md](core-api.md) section 9 - the ingest operations, and why a credit is one image
- [core-api.md](core-api.md) section 20 - the answer path, and what a `resync` is asking for
