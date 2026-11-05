# Media: Design

> **In short:** media holds the images a site's documents use. A group owns each image, and uploading asks Authorisation. Every image is made web-ready on upload: resized, compressed as tightly as it will go, and stripped of location and other metadata. A group may also keep a cleaned full-resolution original, which is private. Alt text is required, and who uploaded it is always recorded. The records live in media's own PostgreSQL schema, and the files in the storage capability. Each group has a storage allowance, set by an entitlement. Agreed with the owner on 25 October 2026; nothing is built yet.

## What it is for

content-editor's design ([content-editor](https://github.com/nuxt4-layers/content-editor/blob/4bdcbe93120cb448ef8705d3b5fc46dca6597d64/docs/design.md)) needs images in documents. Publisher refers to an image as `asset:<id>` and asks its asset port for the address ([Publisher contract](https://github.com/nuxt4-layers/publisher/blob/4410cab7b84c1b0395de2bdf657e3bcc3d476b87/docs/contracts.md) §6.2, §11). media answers that port and offers content-editor a picker. A later gallery or photo of the week draws on the same images: media stores them, and the gallery (most simply a Publisher document type listing `asset:` ids) chooses, orders and captions them.

## Decisions

| Decision | Means |
|---|---|
| Records here, files in a bucket | Each item's record (owning group, uploader, alt text, credit, kind, sizes, status) is in media's own schema ([ADR-0002](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0002-composition-supplied-persistence-and-capability-owned-schemas.md)). The files are kept through storage's contract (its `CONTRACT.md`), which media composes as its storage port; media keeps each file's storage key in its record. Storage makes the keys, under the tenant's and group's identifiers, never under names people chose, and owns buckets, providers and backups of the files |
| Accepted uploads | JPEG, PNG, WebP, AVIF and HEIC (an iPhone's default), up to 10 MB each, or the group's larger limit (below). The file's content is checked, not its name. Images over 50 megapixels are refused before decoding, so a small file that expands enormously cannot exhaust the server. Anything else is `validation-failed` |
| Uploads go straight to storage | Requests through the site are size-limited, so after authorising the upload media asks storage for a signed upload address, the browser uploads to it, and media claims and processes the file. Storage removes an upload that is never claimed within a day |
| Web-ready on upload | Each image is decoded and re-encoded once, as WebP at 480, 960 and 1920 pixels wide (never wider than the original), compressed as tightly as it will go without visible loss. Re-encoding also discards anything hidden in the file. Processing has time and memory limits; one that fails or runs over leaves nothing behind and says so |
| Metadata removed | Location, camera and every other metadata is always removed, from web versions and originals alike |
| Originals off by default | A group whose entitlement allows it keeps a full-resolution original as well. It is cleaned too: a JPEG's metadata is removed without re-encoding, so the picture is untouched; other formats are re-encoded at full size and the highest quality. Originals are never public: they are served through the site to holders of `media.originals:download` |
| Alt text required | An image cannot be inserted until it has alt text, or is marked decorative on purpose (alt left empty deliberately) |
| Credit and rights | The uploader confirms they have the right to use the image. A credit line is optional for their own photos and required for any image they did not take; it is shown wherever the image is |
| Photos of children | The uploader confirms that a parent or guardian consented for anyone identifiable under 18; without that confirmation the upload is refused. The confirmation is recorded with the item |
| Public and private images | Images in public documents have storage's public addresses. Images in members-only content (such as a gallery's members' albums) are served through the site, with an access check on each request, never from a public address |
| A group owns each image | As with content, an image belongs to a group. Uploading needs `media.items:upload` in that group; using an image in a document needs the right to edit that group's content. All through the shared `decide` shape ([ADR-0012](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0012-trivial-composition.md) §2) |
| The uploader is always recorded | Each item keeps its uploader's identity identifier, and screens show the uploader's name through Profile. Events and logs carry identifiers only |
| Allowances by entitlement | media counts each group's storage. Its allowance (total size, keeping originals, and the larger 25 MB upload limit that comes with them) comes from an entitlement port, defaulting to a free allowance of 100 MB per group. media enforces it from its own records and reconciles them with storage's usage figures. A paid subscription or an exception, such as always free for islanders, raises it. Until those capabilities exist, the platform's administrators set allowances by hand. Media never handles payment |
| Over the allowance, refuse | Uploads beyond the allowance are refused with the use and the limit shown. Nothing is ever deleted for being over: a lowered allowance, such as a lapsed subscription, stops new uploads only |
| No deletion while in use | Deleting is refused while a document links to the image, and the refusal lists those documents, as with Publisher's `acknowledgedLinks`. Archiving hides an image from the picker without breaking documents that use it |
| Unused images tidied | An image no document has used within 90 days of upload is listed to its group's administrators for deletion; nothing is removed without them |
| Erasure leaves a blank | When an image must go (the uploader's or a pictured person's erasure, or a takedown), its files and original are deleted and it answers a blank image, at a new address so that cached copies are not served, with its alt text set to a neutral "Image removed". The `asset:` id still resolves, so documents never break. On the uploader's erasure, their identifier is removed from the record |
| Reporting an image | Every public image can be reported. A report goes to the group's administrators and the platform's operators, who keep the image or take it down; reports of the most serious kinds (the platform's list) hide it at once while they decide. Reports and decisions are events for the audit capability. This comes in version 1, as the first part of the Online Safety Act duties; slice 3 adds the rest |
| Backups in step | Storage backs up the files, ageing copies out within 30 days; media backs up its records on the same schedule, so a restore never leaves records without files or the reverse. Each erasure is recorded, so a restore applies it again |

## Permissions

| Permission | Risk | Allows |
|---|---|---|
| `media.items:view` | low | Seeing a group's images in the picker and library |
| `media.items:upload` | low | Uploading to the group, and editing alt text and credit |
| `media.items:archive` | medium | Archiving and restoring |
| `media.items:delete` | medium | Deleting an unused image |
| `media.originals:download` | medium | Downloading a kept original |
| `media.reports:decide` | medium | Keeping or taking down a reported image |
| `media.allowances:manage` | high | Setting a group's allowance by hand (platform operators only) |

## Screens

A presentation part ([ADR-0009](https://github.com/nuxt4-layers/platform-architecture/blob/249cd48af72e0a3fc232d40c8aa05e8da3b32069/docs/decisions/ADR-0009-headless-capability-layers.md)), for people who edit: a picker component that content-editor uses, a library page per group at `/media/:groupId` showing its use against its allowance, and a reports page at `/media/:groupId/reports`. The report button on public images is a small component the host places. All are styled only with Theme Manager's vocabulary, worded from an en-GB catalogue, accessible in light and dark mode, and can be switched off with `media: { presentation: false }`.

## Where the files go for Westray-Papa-Westray

The provider, region and limits belong to storage and the site. Storage's limits on the bucket are reported by storage; media's allowances are media's. **The site pays for all storage,** free groups included, so the free allowance is the owner's own cost per group.

## Not in version 1

Video, documents such as PDFs, AVIF output alongside WebP, blurred placeholders, captions and long descriptions, cropping and focal points, albums, and download statistics.

## Order of work

1. storage's contract is available.
2. media's core: contract, composing storage, direct uploads, processing, records, allowances, private serving, Publisher's asset resolver, fakes and tests.
3. Its presentation part: the picker, the library and reports.
4. `examples/minimal-host`, `CONTRACT.md` and a release; then content-editor builds on it.
