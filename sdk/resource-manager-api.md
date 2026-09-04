# Resource Manager API v1 - withdrawn

> **This document described a contract that no longer exists.** It is a record of what was published and why it was taken back, kept because a document that was withdrawn is exactly the kind of thing somebody has to be able to look up. What a resource manager does today is [`writing-a-resource-manager.md`](writing-a-resource-manager.md).

`resource-manager-api.v1.yaml` described what a **resource manager served to PDM**: the manifest it declared itself with, its health, a cursor-paged walk of its images, the bytes of one image, a thumbnail, batch metadata, batch recycle / delete / move, connecting an account, and a push of the settings it declared. It was written for [#142](https://github.com/Iskandarus/photo-duplicates-manager/issues/142), reached 1.2, and was **withdrawn at 1.3 by [#278](https://github.com/Iskandarus/photo-duplicates-manager/issues/278)**, the last piece of [#254](https://github.com/Iskandarus/photo-duplicates-manager/issues/254).

## Why it was withdrawn

The reversal in #254 turned the whole integration around. The rule is one sentence - **the core only answers** - and everything the document described was PDM initiating: a request going out from the core to a program somebody else wrote, on an address that program had to expose. By #277 the last caller was gone; #278 deleted the gateway that made the calls, and a contract with no caller on one side is not a contract, it is a description of code nobody runs.

**No deprecation window**, which is the shape [#197](https://github.com/Iskandarus/photo-duplicates-manager/issues/197) used when it withdrew the UI protocol at 1.0. Nobody outside this repository has implemented it and nobody could have: it never left this private repository, and the owner has confirmed it was never handed to anybody privately (#255). The twelve-month overlap [`version-compatibility.md`](version-compatibility.md) section 3.2 asks for is not owed to an implementer who does not exist, and saying so here is the rule #197 set - a waiver is written down, never inferred.

## What went with it

| | Where it was |
| --- | --- |
| The document | `asp-back/contracts/resource-manager-api.v1.yaml` |
| The suite that checked a live manager against it | `asp-back/PDM.Conformance`, `PDM.Conformance.Tests` |
| A `dotnet new` template that served it | `asp-back/sdk/templates/pdm-resource-manager` |
| A dependency-free Node implementation of it | `asp-back/sdk/samples/node-resource-manager` |
| The two CI jobs that ran all three | `reference` and `sdk` in `.github/workflows/contracts.yml` |
| The client half - what PDM did when it dialled | `asp-back/PDM.ResourceManagers.Client` |

`docs/history.md` is the record of the client half, kept for the same reason as this one.

## What replaced it

One document and one direction: [`core-api.md`](core-api.md) and [`core-api.v1.yaml`](core-api.v1.yaml), which a resource manager **calls**. The mapping, operation by operation:

| It served | It calls, or serves to its own frontend |
| --- | --- |
| `GET /v1/manifest`, `GET /v1/health` | nothing. PDM asks a manager nothing about itself; what an administrator writes down on the registration is the whole of what PDM knows. |
| `POST /v1/images:nextBatch` | its own walk, declared with `POST /core/v1/searches/{searchId}/items` (#275, #276). |
| `GET /v1/images/{ref}/content` | `PUT /core/v1/searches/{searchId}/items/{imageRef}/content`, and only for the images PDM answered `needsContent` for. |
| `GET /v1/images/{ref}/thumbnail` | its own frontend, on its own origin (#277). PDM renders no tile. |
| `POST /v1/images:batchMetadata` | its own frontend, the same way. |
| `POST /v1/files/recycle`, `/delete`, `/move` | its own file operations, followed by `POST /core/v1/images/gone` so a stored result stops naming a file that is not there. |
| `GET /v1/auth/status`, `auth/start`, `auth/disconnect` | its own account handling, end to end, returning to its own origin (#250). |
| `PUT /v1/settings` | its own settings, in its own database (#229). |

## What a manager still serves, and what that is

Two things, and neither is this contract.

**`/workspace`** is the backend under a manager's own frontend (#248, #250). Its only correspondent is the page beside it, on the same origin, holding a session that manager minted. It is not a published contract and has no version: both halves ship together, in one build, from this repository.

**One credential still crosses from PDM.** `POST /workspace/session` accepts the short-lived frame token PDM mints when the app host mounts a manager's frontend (#249) - PDM signing, for a browser, a statement about which user is looking at the page. That is the one arrow #254 keeps, because a frame origin is where a browser loads a page and not where the core dials.

**The four built-in managers still expose `/v1`**, and nothing calls it. It is a fossil of this document: the surface compiles, is gated by a token class PDM no longer mints, and is unreachable by anything in this repository or outside it. Removing it is a deletion of its own - it reaches the SDK's schema, four services and three test projects - and it is deliberately not folded into #278, which is already the largest single change in the project. Until it goes, treat it as documented here and nowhere else: it implements a contract that has been withdrawn.
