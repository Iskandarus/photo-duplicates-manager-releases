# Photo Duplicates Manager - resource manager SDK 3.0.5

Everything you need to write a resource manager: a program that owns a collection of photographs and
wants Photo Duplicates Manager to find the duplicates in it.

**Start with [`writing-a-resource-manager.md`](writing-a-resource-manager.md).** You implement no API and serve nothing to
PDM. Your manager holds client credentials, walks its own resource, and feeds PDM what it finds.

## What is in here

| File | Where it comes from |
| --- | --- |
| `writing-a-resource-manager.md` | `docs/writing-a-resource-manager.md` |
| `core-api.v1.yaml` | `asp-back/contracts/core-api.v1.yaml` |
| `CHANGELOG.md` | `asp-back/contracts/CHANGELOG.md` |
| `core-api.md` | `docs/core-api.md` |
| `manager-scan-driver.md` | `docs/manager-scan-driver.md` |
| `manager-harness.md` | `docs/manager-harness.md` |
| `resource-manager-frontend.md` | `docs/resource-manager-frontend.md` |
| `version-compatibility.md` | `docs/version-compatibility.md` |
| `resource-manager-api.md` | `docs/resource-manager-api.md` |

`core-api.v1.yaml` is the contract. Everything else is prose about it, and the two are meant to be
read together: generate your client from the document, and read the guide for the half a schema
cannot say.

## And the tool, beside this zip

`pdm-harness-3.0.5-<platform>` is on the same release. It **plays the core** on a loopback port,
so you can drive your manager against something that answers and be told which of eighteen rules it
broke and at which call. One self-contained file: nothing to install, no runtime to fetch. On macOS
and Linux, unpack the `.tar.gz` and `chmod +x pdm-harness`.

`manager-harness.md` is what it checks and why.

## Versions

This SDK is **3.0.5**: core API **3.0**, revision **5**. The
first two numbers are the contract's own version, which is what your client negotiates with the
`PDM-Core-Version` header on every request. The third counts re-publications of the same contract -
a clearer guide, a corrected example, a fix in the harness - and means nothing to a client.

The zip and the harness carry that number together, so a tool and a guide that came out of the same
release always describe the same contract.

`CHANGELOG.md` is the contract's history. Its second column is written for an implementer who does
nothing, which is usually the honest answer: a minor is additive.

## Where this came from

Iskandarus/photo-duplicates-manager-releases, tag `sdk-v3.0.5`. PDM's own source is not
public; these documents are, because a contract nobody can read is not a contract.

Sentences in these documents sometimes name a file in PDM's repository. Those links are flattened to
plain text here rather than left pointing at nothing: the name is still there if you have reason to
ask about it, and it is not something you need in order to write a manager.