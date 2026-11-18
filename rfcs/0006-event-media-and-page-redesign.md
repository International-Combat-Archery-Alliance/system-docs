# RFC-0006: Event Media & Event Page Redesign

> Status: **Draft** · Date: 2026-11-17
> Depends on: **None** — additive to `event-registration` and `icaa.world`; uses existing `assets-api`
> endpoints (`GET /assets/v1`, `GET /assets/v1/by-path`, folder/upload admin routes)
> Related: RFC-0002 (Schedule/Standings cards gain room in the new layout), RFC-0005 (admin
> asset-picker patterns), [../docs/frontend.md](../docs/frontend.md),
> [../docs/services.md](../docs/services.md)

## 1. Background & Problem

`Event` in `event-registration` carries only two flat media fields: `imageName` (the event logo, a URL
string) and `rulesDocLink` (the rules document, a URL string). Both are rendered by hand on the event
page and nothing else can be attached to an event. Organizers therefore have **no structured way** to
publish post-event or live event assets:

- photo folders and individual files that already live in `assets-api` (the site's only asset store),
- YouTube/Twitch VODs and streams,
- arbitrary external resource links (brackets, stat sheets, sponsor pages).

Today those links are either pasted into article content, sent out on social media, or not published
at all. There is no ordering, no titles/descriptions, and no way to render a photo gallery or a VOD
playlist from event data.

On the frontend, `icaa.world/src/components/EventPageTemplate.tsx` uses a rigid
`lg:grid-cols-3` CSS Grid with **hardcoded row indices** (`lg:row-start-1` … `lg:row-start-4`) and a
hardcoded right column (`lg:col-start-3`). Two concrete problems follow:

1. **Schedule is cramped.** It is pinned to `lg:col-span-1 lg:col-start-3 lg:row-span-4` — one third
   of a `max-w-screen-xl` container — while Standings takes `lg:col-span-2`. A 5-column game table in
   ~400 px forces wrapping and horizontal squeeze on ordinary desktop windows.
2. **Variable media is impossible.** Every card must claim a fixed grid slot. Adding a photo
   carousel, a gallery, a VOD playlist, or "only when populated" resource links would require
   reassigning `row-start` indices per event — the layout cannot grow with the data.

This RFC adds ordered, typed media to the `Event` aggregate and replaces the fixed grid with a
breathing, section-based event page.

## 2. Goals / Non-goals

### Goals

- Add an **optional, ordered `media` list** (`EventMediaItem[]`) to `Event` in `event-registration`.
- Support linking both **folders** (`asset_folder`) and **individual files** (`asset_file`) from
  `assets-api`, as well as **YouTube/Twitch VODs & streams** (`video`) and **external links**
  (`external_link`).
- Upgrade `AssetPickerModal` and `AdminEventForm` so admins can select folders, create new folders
  inline, batch-upload multiple files, and configure/reorder event media items.
- Redesign `EventPageTemplate.tsx` so **Schedule and Standings have generous horizontal space** on
  desktop, and media sections render dynamically whenever populated using **Smart Section Grouping**
  (§4).
- Keep the default ICAA Twitch/YouTube livestream card as a **fallback** when no `video` items are
  configured on an event.

### Non-goals (defer)

- **Changes to `assets-api`.** Its public `GET /assets/v1` and `GET /assets/v1/by-path` reads plus the
  admin folder and presigned-upload endpoints already support every operation this RFC needs. No new
  asset endpoints, no new S3 layout.
- **Non-admin / public user uploads.** Posting media stays admin-only; visitors can only consume it.
- Replacing `imageName` (logo) or `rulesDocLink` (rules). Both stay exactly as they are; `media` is
  additive (decision D33).
- Changing where Schedule/Standings data comes from — that is RFC-0002. This RFC only reflows their
  containers and removes the hardcoded grid rows.
- Per-media view analytics, comments, reactions, or scheduled publish windows.

## 3. API & Data Model (`event-registration`)

### 3.1 OpenAPI additions (`event-registration/spec/api.yaml`)

`Event` is the request body for both `POST /events/v1` and `PUT /events/v1/{id}`, so adding one
optional property covers create, update, and read.

```yaml
EventMediaType:
  type: string
  enum:
    - asset_folder    # path in assets-api, e.g. /images/events/boston-2026
    - asset_file      # direct CDN URL of one file
    - video           # YouTube / Twitch URL (VOD, live stream, playlist)
    - external_link   # any other resource URL

EventMediaDisplayMode:
  type: string
  enum:
    - auto          # resolve from type + folder contents (default)
    - carousel      # asset_folder: Embla carousel
    - gallery       # asset_folder: thumbnail grid + lightbox
    - file_list     # asset_folder: downloadable file rows
    - embed         # asset_file/video: inline player or viewer
    - link          # asset_file/external_link/video: CTA / resource row

EventMediaItem:
  type: object
  required:
    - type
    - source
  properties:
    type:
      $ref: '#/components/schemas/EventMediaType'
    source:
      type: string
      maxLength: 1000
      description: >-
        An assets-api folder path (starts with "/") for asset_folder, otherwise a valid
        http(s) URL.
      example: /images/events/boston-2026
    title:
      type: string
      maxLength: 255
      example: 2026 Boston Championship Photos
    description:
      type: string
      maxLength: 1000
      example: Finals, podium, and behind-the-scenes shots.
    displayMode:
      $ref: '#/components/schemas/EventMediaDisplayMode'
```

`Event.media` (optional, no `minItems`):

```yaml
media:
  type: array
  maxItems: 50
  items:
    $ref: '#/components/schemas/EventMediaItem'
```

Example update payload:

```json
{
  "name": "2026 Boston Championship",
  "media": [
    { "type": "asset_folder", "source": "/images/events/boston-2026", "title": "Event Photos", "displayMode": "carousel" },
    { "type": "video", "source": "https://www.youtube.com/watch?v=abc123", "title": "Championship Final", "displayMode": "embed" },
    { "type": "video", "source": "https://www.twitch.tv/videos/987654321", "title": "Full Day 1 VOD", "displayMode": "link" },
    { "type": "external_link", "source": "https://example.com/bracket.pdf", "title": "Bracket" }
  ]
}
```

### 3.2 Domain & DynamoDB

`events/events.go`:

```go
type EventMediaType string

const (
    MediaTypeAssetFolder  EventMediaType = "asset_folder"
    MediaTypeAssetFile    EventMediaType = "asset_file"
    MediaTypeVideo        EventMediaType = "video"
    MediaTypeExternalLink EventMediaType = "external_link"
)

type EventMediaDisplayMode string // auto | carousel | gallery | file_list | embed | link

type EventMediaItem struct {
    Type        EventMediaType
    Source      string
    Title       *string
    Description *string
    DisplayMode *EventMediaDisplayMode // nil == auto
}
```

`Event` gains `Media []EventMediaItem`. `eventDynamo` in `dynamo/events.go` gains the matching
`Media []eventMediaItemDynamo` with `dynamo:"media,omitempty"` and is mapped in both directions
(`toDynamo`/`fromDynamo`).

- **Stored inline** on the `EVENT#<id>` item — no new item type, no GSI attributes touched (the
  only-EVENT/TEAM GSI rule from RFC-0001/0002 still holds).
- **Backward compatible:** items written before this RFC have no `media` attribute and deserialize
  as `nil`/empty. Reads must not error; writes must not emit an empty attribute.
- `CreateEvent` and `UpdateEvent` copy `Media`; `UpdateEvent` treats it like any other mutable field
  (absent body field = clear, per the existing full-replace semantics of `PUT`).
- Order is the admin-defined slice order and is preserved as written. **No server-side sort.**

### 3.3 Validation (`api/events.go`)

Validation runs on create and update, before the repository call. Failures return the service's
standard `400` with the existing error shape.

| `type` | `source` rule |
|---|---|
| `asset_folder` | Non-empty, must start with `/`, no `..` segments, no trailing slash (except root `/`), ≤ 1000 chars. Does **not** require the folder to exist yet — assets may be uploaded after the event is saved. |
| `asset_file` | Non-empty, valid absolute `http(s)` URL, ≤ 1000 chars. Typically an assets CDN URL. |
| `video` | Non-empty, valid absolute `http(s)` URL, ≤ 1000 chars. Embeddability is a frontend concern (§5); unknown hosts degrade to a link. |
| `external_link` | Non-empty, valid absolute `http(s)` URL, ≤ 1000 chars. |

Also enforced: `len(media) ≤ 50`; known `type`/`displayMode` enum values only (OpenAPI does this, the
handler re-checks for defense in depth); `title ≤ 255`, `description ≤ 1000`. Validation lives in one
helper (`validateMedia`) so create and update share it and table-driven tests cover every branch.

## 4. Event Page Redesign & Smart Media Grouping (`icaa.world`)

### 4.1 Page layout redesign (`EventPageTemplate.tsx`)

Replace the `lg:grid-cols-3` / `lg:row-start-*` grid with a vertical, section-based container:

```tsx
<div className="mx-auto max-w-screen-xl space-y-6">
  <EventHeroCard event={event} />          {/* full width */}
  <StandingsCard eventId={event.id} />     {/* full width */}
  <ScheduleCard eventId={event.id} />      {/* full width */}
  <EventMediaSections event={event} />     {/* smart grouping, §4.2 */}
  <RulesCard rulesDocLink={event.rulesDocLink} />
</div>
```

1. **Event Overview / Hero Card** — full-width header card combining the event logo (`imageName`),
   title, date/time (timezone-aware), location/address, calendar/share buttons, and registration
   CTAs. This absorbs the current "Details" card; the registration-closed notice and pricing buttons
   move here unchanged.
2. **Competition Section (Standings & Schedule)** — full-width cards, stacked with consistent
   spacing (`space-y-6`). Schedule is no longer a right sidebar: its 5–6 column tables get the whole
   content width. On very wide screens the two cards may sit side by side (`lg:grid-cols-2`) only when
   the table is narrow (≤ 5 columns); the default is stacked so tables never wrap. The
   `isBoston*` hardcoded data stays until RFC-0002's backfill lands, then the cards render entirely
   from the API — this RFC changes only their containers.
3. **Media & Livestream Sections (Smart Section Grouping)** — see §4.2.
4. **Rules Section** — the existing embedded Google Drive/PDF viewer card, unchanged, rendered only
   when `rulesDocLink` is set.

No `row-start`/`col-start` classes remain anywhere in the page.

### 4.2 Smart Section Grouping

Media items are rendered into **type-based sections**, not in one flat list. Section order is fixed
(videos → photos → featured files → resources) so pages stay predictable; admin order is preserved
**within** each section. A section renders only when it has at least one item, so an event with no
media shows exactly the old page (hero + competition + fallback livestream + rules).

| `type` | `displayMode` | Rendering |
|---|---|---|
| `video` | `embed` (default `auto`) | Entry in the **Videos & Livestreams** card: featured 16:9 player + VOD playlist selector |
| `video` | `link` | CTA button in the same card's footer (opens the source in a new tab) |
| `asset_folder` | `carousel`, or `auto` when the folder contains images | Own **Photo card**: Embla carousel (`CarouselImages`) + "View all" → Lightbox |
| `asset_folder` | `gallery` | Own **Photo card**: responsive thumbnail grid + Lightbox with prev/next |
| `asset_folder` | `file_list`, or `auto` when the folder has no images | Own **Photo/asset card**: downloadable file list (PDFs, stat sheets) |
| `asset_file` | `embed`, or `auto` for images/PDFs | **Featured Files** card: inline image or embedded PDF viewer |
| `asset_file` | `link` | Row in the **Resources & Links** card (download action) |
| `external_link` | any (`auto`/`link` expected) | Row in the **Resources & Links** card (external action) |

`auto` resolution is a pure frontend helper (`resolveMediaRender(item, assets)`) so it is testable and
identical for every call site. Incompatible combinations (e.g. `external_link` + `embed`) degrade to
`link` instead of erroring — the schema stays open and the page never breaks.

#### Videos & Livestreams card (featured player + VOD selector)

- Collects all `video` items in admin order. Default active item is the first one.
- **Featured 16:9 Player:** a single `aspect-video` iframe for the active item only. Switching the
  selection swaps the `src`/`key`; **multiple iframes are never mounted at once** (no background
  players, no autoplaying audio).
- **VOD playlist selector:** a clickable list of every video (title, optional description, active
  highlight via `aria-pressed`) below the player on mobile and beside it on desktop
  (`lg:grid-cols-[minmax(0,1fr)_18rem]`). The list is the only way to switch players; it also gives
  link-mode items a home.
- **Link-mode videos** (`displayMode: link`) render as CTA buttons in the card footer instead of in
  the selector.
- **Fallback:** if the event has **zero** `video` items, the card renders the existing default ICAA
  Twitch & YouTube buttons (`twitch.tv/icaa_world`, `youtube.com/@Icaa-world/streams`). If the event
  has any `video` item, the fallback is not shown — admins opted into their own list.

#### Photo / asset folder cards (`asset_folder`)

- Each folder item is **its own Card**, in admin order, titled `item.title || "Event Photos"` with the
  optional `description` as supporting text.
- Data comes from `useGetAssets(item.source, 100)`; each card fetches independently, so one slow or
  broken folder never blocks the rest of the page. TanStack Query de-dupes the identical query used
  by `CarouselImages`.
- `carousel` / `auto` (image folder): `CarouselImages` (Embla) with a "View all photos" button.
  Clicking the active slide or the button opens the Lightbox at that index. `CarouselImages` gains an
  optional `onImageClick?: (index: number) => void` prop (backwards compatible) so the parent can open
  the Lightbox at the right image.
- `gallery`: `grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4`, square `object-cover` thumbnails;
  clicking one opens the Lightbox at that index.
- `file_list`: rows of name, type icon, size, and a download action (`<a download>`); alphabetical.
- **Lightbox** (new `MediaLightbox` component, shadcn `Dialog`): full-screen image, prev/next
  controls, keyboard arrows, `3 / 24` counter, closes on Escape/backdrop. Index state lives in the
  card.
- Empty folders collapse on the public page (no dead "no photos" cards); a loading skeleton shows
  while fetching. Folders are capped at the first 100 assets in v1 (cursor paging exists in
  `useGetAssets` if a larger limit is ever needed).

#### Featured inline files (`asset_file` with `embed`)

- Images render inline (`w-full rounded-lg`) with optional title/description caption.
- PDFs render in an embedded viewer iframe (reuse the Rules card's
  `drive.google.com/viewerng/viewer?embedded=true&url=…` wrapper) at `h-[70vh]`.
- Other file types fall back to a resource row rather than an empty frame.

#### Resources & Links card

- Automatically groups **all** `external_link` items and **all** `asset_file` items in `link` mode
  (plus `embed` fallbacks for non-renderable types) into a single card titled "Resources & Links".
- Rows show icon (globe vs file), `item.title` (or URL host / file name), optional description, and
  the appropriate action: external link (`target="_blank"` + `rel="noreferrer noopener"`) or
  download.
- The card is omitted when the group is empty.

### 4.3 Component shape

| Component | Responsibility |
|---|---|
| `EventPageTemplate` | Layout shell + grouping; no fetching beyond the event |
| `EventMediaSections` | Resolves `Event.media` into groups, renders section order, hides empties |
| `EventVideosCard` | Featured player, VOD selector, link CTAs, channel fallback |
| `EventFolderCard` | Per-folder card; carousel / gallery / file list + Lightbox trigger |
| `EventFeaturedFileCard` | Inline image or PDF viewer |
| `EventResourcesCard` | External links + linked files |
| `MediaLightbox` | Shared image viewer with prev/next + keyboard nav |

`CarouselImages` is reused as-is apart from the optional `onImageClick` prop. No existing carousel,
asset-browser, or upload behavior is removed.

### 4.4 Edge cases & fallbacks

- Event with no `media` at all → old behavior + default livestream buttons; nothing renders empty.
- Legacy events (no `media` attribute) → `nil` slice → same as above.
- Folder deleted in `assets-api` after the event was saved → the card hides (public) after the query
  errors; other cards still render.
- Folder with mixed content → `auto` picks carousel when any image exists, otherwise file list.
- Video URL that cannot be embedded (unknown host, invalid ID) → render a link row/CTA, never a blank
  iframe.
- Fewer images than a carousel needs (1–2) → gallery-style single image; controls omitted.
- Duplicate `(type, source)` pairs are allowed by the schema (same folder may appear twice) but the
  admin form surfaces a non-blocking warning.
- 50-item cap keeps the payload and the DynamoDB item well bounded (items are small strings; item
  size is dominated by `media` metadata only).

## 5. Embed Utilities (`src/lib/embedUtils.ts`)

`buildEmbedData` already resolves YouTube/Editor.js patterns, PDFs, and a generic fallback. Extend it
with **Twitch**, which Editor.js cannot embed correctly on its own because Twitch requires a `parent`
parameter and rejects generic iframes without one.

Patterns to support (both `twitch.tv` and `www.twitch.tv`, optional trailing slash, ignore query):

| URL shape | Embed target |
|---|---|
| `twitch.tv/videos/<id>` | `player.twitch.tv/?video=<id>&parent=<host>&autoplay=false` |
| `twitch.tv/<channel>/clip/<slug>` | `clips.twitch.tv/embed?clip=<slug>&parent=<host>&autoplay=false` |
| `clips.twitch.tv/<slug>` | `clips.twitch.tv/embed?clip=<slug>&parent=<host>&autoplay=false` |
| `twitch.tv/<channel>` | `player.twitch.tv/?channel=<channel>&parent=<host>&autoplay=false` |

Implementation notes:

- Resolve the parent from `window.location.hostname` (`icaa.world` in prod, `localhost` in dev) and
  guard `typeof window !== 'undefined'` even though the SPA is client-only.
- A pure `parseTwitchUrl(url)` returns a discriminated union
  (`{kind: 'video' | 'clip' | 'channel', id}`), unit-testable with URL fixtures; `buildEmbedData`
  returns `service: 'twitch'` with a 16:9 `width`/`height`.
- Twitch detection runs **before** the generic Editor.js pattern loop (its twitch pattern does not add
  `parent`), and before the PDF/generic fallback.
- YouTube videos, live streams (`/live/<id>`), and playlists continue to route through the existing
  Editor.js detection; no behavior change for them.
- The player iframe is sandboxed with `allow-scripts allow-same-origin allow-presentation
  allow-popups` — Twitch refuses to load without same-origin.

## 6. Admin Experience (`AssetPickerModal.tsx` & `AdminEventForm.tsx`)

### 6.1 `AssetPickerModal`

Extend `AssetPickerMode` from `'image' | 'allFiles'` to `'image' | 'allFiles' | 'folder'`:

- **`mode="folder"`** turns the browse tab into a folder tree walker: folders are selectable, files
  are not. The action button becomes **"Select Current Folder"** and emits the current path (starting
  with `/`) through `onSelect`. Root `/` is a valid selection but shows a hint that it is the site
  root.
- **Inline "New Folder"** button in the browse tab opens a small name input, calls
  `useCreateFolder()` with `{ path: currentPath, name }`, then invalidates the `['get', '/assets/v1']`
  query so the new folder appears immediately and is auto-selected. Validation mirrors `assets-api`
  (non-empty, no `/`, no reserved names) and the existing `ParentFolderNotFound` message path stays
  useful.
- **Multi-file upload** in `UploadTab`: `<input type="file" multiple>`; upload files sequentially
  through the existing presign → S3 POST → confirm flow (not in parallel — the current flow already
  cleans up on failure), with progress `current / total` and per-file error reporting. After the
  batch, refresh the browse listing and call `onSelect` for the last uploaded file (single-select
  callers keep working); batch callers can consume the new `onBatchUploaded?: (urls: string[]) => void`
  callback to add several `asset_file` items at once.
- Existing behavior (`image`, `allFiles`, delete, replace, error handling) is unchanged.

### 6.2 `AdminEventForm`

Add an **"Event Media & Post-Event Assets"** card below the existing logo/rules fields:

- Zod schema gains `media: z.array(mediaItemSchema).max(50, 'At most 50 media items')`, default `[]`;
  edit mode seeds from `editData.media ?? []`; the submit body includes `media`.
- `useFieldArray({ control, name: 'media' })` renders one row per item with:
  - a `type` select (`asset_folder | asset_file | video | external_link`),
  - a `displayMode` select limited to modes that make sense for the chosen type (plus `auto`),
  - `source` input (read-only text field filled by the picker for asset types; plain URL input for
    `video`/`external_link`),
  - optional `title` / `description` inputs,
  - **Move up / Move down** buttons (`swap`) and **Remove** (`remove`),
- Picker triggers are type-aware: **"Browse Assets"** opens `AssetPickerModal` in `allFiles` mode for
  `asset_file`; **"Select Folder"** opens it in `folder` mode for `asset_folder` and writes the
  returned path into `source`. A **"Add Media Item"** button appends an empty row (default
  `displayMode: auto`).
- Ordering in the form is the stored order; no drag-and-drop in v1 (move buttons only).
- Invalid rows block submit with inline messages; unknown/invalid URLs are caught client-side and
  again server-side (§3.3).

## 7. Implementation Progress Checklist

**Backend (`event-registration`)**

- [ ] `spec/api.yaml`: `EventMediaType`, `EventMediaDisplayMode`, `EventMediaItem`, `Event.media`
      (`maxItems: 50`) + `go generate ./...`
- [ ] Domain: `events.EventMediaItem` / enums on `Event.Media`
- [ ] DynamoDB: inline `media` on `EVENT#<id>` (omitempty), map in/out, `CreateEvent` +
      `UpdateEvent`; legacy items read as empty
- [ ] `api/events.go`: `validateMedia` (folder path vs URL, lengths, enums, ≤ 50) wired into create
      and update; table-driven tests
- [ ] Handler mapping (`toAPIEvent`) returns `media` in list + detail responses

**Frontend — embeds & codegen**

- [ ] `bun run codegen` for the events spec (generated types must not be hand-edited)
- [ ] `embedUtils.ts`: Twitch parser (`videos`, `clip`, `clips.twitch.tv`, channel) with
      `parent`/`autoplay=false`; detection before generic Editor.js patterns
- [ ] URL fixtures for YouTube video/live/playlist + Twitch + PDF + unknown host

**Frontend — `AssetPickerModal` + `AdminEventForm`**

- [ ] `mode="folder"` + "Select Current Folder" (path emission)
- [ ] Inline "New Folder" via `useCreateFolder` + listing invalidation + auto-select
- [ ] `UploadTab` multi-file batch upload with sequential progress + `onBatchUploaded`
- [ ] `AdminEventForm`: Zod `media` array, `useFieldArray` rows, type-aware picker triggers,
      move up/down/remove, submit mapping

**Frontend — `EventPageTemplate` redesign & media components**

- [ ] Replace `lg:grid-cols-3` / `row-start` grid with `max-w-screen-xl mx-auto space-y-6` sections
- [ ] Full-width Hero card (logo, title, date/time, location, calendar/share, registration CTAs)
- [ ] Full-width Standings and Schedule cards (no right sidebar; tables no longer wrap)
- [ ] `MediaLightbox` (prev/next, keyboard, counter) + `CarouselImages` `onImageClick`
- [ ] `EventVideosCard`: featured 16:9 player + VOD selector, link CTAs, ICAA channel fallback
- [ ] `EventFolderCard`: carousel / gallery / file_list via `useGetAssets(source, 100)`
- [ ] `EventFeaturedFileCard` (inline image / PDF viewer) + `EventResourcesCard`
- [ ] `EventMediaSections`: grouping order, `auto` resolution, hide empty sections
- [ ] Verify legacy events (no `media`) render exactly as before and the livestream fallback appears

**Docs & verification**

- [ ] Update `docs/data.md` (`EVENT` item gains `media`) and `docs/frontend.md` (event page media
      sections, admin media form)
- [ ] `bun run lint`, `bun run test:ts`, `bun run format`, `bun run build`; Go tests + `go vet`

## Open decisions

Product choices specific to event media. Keep global `D#`s; the
[index](../OPEN-DECISIONS.md) maps every decision to its owning RFC.

| # | Decision | Recommended default | Status |
|---|---|---|---|
| D32 | **Media storage shape** — inline `media` on the `EVENT#<id>` item vs a separate `MEDIA#<eventId>#<n>` item type. Inline keeps reads to one `GetItem` and updates atomic with the event, at the cost of a 50-item/400 KB item bound. | Inline on EVENT (`maxItems: 50`) | [ ] |
| D33 | **Legacy fields** — migrate `imageName` (logo) and `rulesDocLink` (rules) into `media` items, or leave them as dedicated fields? | Leave as-is; `media` is additive (hero keeps `imageName`, Rules card keeps `rulesDocLink`) | [ ] |
