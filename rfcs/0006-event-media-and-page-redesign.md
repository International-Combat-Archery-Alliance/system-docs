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

`Event` is the request body for `POST /events/v1` and `PATCH /events/v1/{id}`. Adding one optional
property covers create, update, and read. The existing update operation is `PATCH`, not `PUT`.

```yaml
EventMediaType:
  type: string
  enum:
    - asset_folder    # path in assets-api, e.g. /images/events/boston-2026
    - asset_file      # direct CDN URL of one file
    - video           # YouTube video/live/playlist or Twitch VOD/clip/channel URL
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

EventMediaIcon:
  type: string
  enum:
    - youtube
    - twitch
    - google_drive
    - instagram
    - pdf
    - photo_gallery
    - file
    - video
    - external_link

EventMediaItem:
  type: object
  required:
    - id
    - type
    - source
  properties:
    id:
      type: string
      format: uuid
      description: Stable client-generated identity, preserved across edits and reordering.
    type:
      $ref: '#/components/schemas/EventMediaType'
    source:
      type: string
      maxLength: 1000
      description: >-
        A canonical assets-api folder path (starts with "/") for asset_folder, otherwise an
        absolute https URL. HTTP is permitted only for loopback development URLs in local mode.
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
    icon:
      $ref: '#/components/schemas/EventMediaIcon'
      description: Optional preset override; when omitted, the frontend detects an icon automatically.
```

Use one `source` property for every media type; `type` determines whether it is an assets folder
path or a URL. Do not add separate per-type path/URL fields.

`Event.media` (optional, no `minItems`):

```yaml
media:
  type: array
  maxItems: 50
  items:
    $ref: '#/components/schemas/EventMediaItem'
```

Example update payload (IDs are stable across edits and reordering):

```json
{
  "name": "2026 Boston Championship",
  "location": {
    "name": "Boston Community Center",
    "address": {
      "street": "123 Main St",
      "city": "Boston",
      "state": "MA",
      "postalCode": "02108",
      "country": "USA"
    }
  },
  "startTime": "2026-08-19T18:00:00.000Z",
  "endTime": "2026-08-19T22:00:00.000Z",
  "registrationCloseTime": "2026-08-17T23:59:59.000Z",
  "registrationOptions": [
    { "registrationType": "ByIndividual", "price": { "currency": "USD", "amount": 0 } }
  ],
  "allowedTeamSizeRange": { "min": 1, "max": 5 },
  "media": [
    { "id": "8a7bf642-8a30-4c38-a73c-bb8da871e81f", "type": "asset_folder", "source": "/images/events/boston-2026", "title": "Event Photos", "displayMode": "carousel" },
    { "id": "42a1b918-85f3-498f-a9aa-df7bc3658188", "type": "video", "source": "https://www.youtube.com/watch?v=abc123", "title": "Championship Final", "displayMode": "embed" },
    { "id": "66aa0176-bc02-4782-9820-84331a626331", "type": "video", "source": "https://www.twitch.tv/videos/987654321", "title": "Full Day 1 VOD", "displayMode": "link" },
    { "id": "6ca48ec4-1f28-4ab1-a482-a5b18c5f8939", "type": "external_link", "source": "https://example.com/bracket.pdf", "title": "Bracket", "icon": "pdf" }
  ]
}
```

### 3.2 Domain & DynamoDB

`events/events.go`:

```go
type EventMediaType string

// EventMediaItem.ID is a UUID stable across edits and reordering.
// The admin client generates it when an item is added.
type EventMediaItem struct {
    ID          uuid.UUID
    Type        EventMediaType
    Source      string
    Title       *string
    Description *string
    DisplayMode *EventMediaDisplayMode // nil == auto
    Icon        *EventMediaIcon        // nil == automatic detection
}

const (
    MediaTypeAssetFolder  EventMediaType = "asset_folder"
    MediaTypeAssetFile    EventMediaType = "asset_file"
    MediaTypeVideo        EventMediaType = "video"
    MediaTypeExternalLink EventMediaType = "external_link"
)

type EventMediaDisplayMode string

const (
    MediaDisplayModeAuto     EventMediaDisplayMode = "auto"
    MediaDisplayModeCarousel EventMediaDisplayMode = "carousel"
    MediaDisplayModeGallery  EventMediaDisplayMode = "gallery"
    MediaDisplayModeFileList EventMediaDisplayMode = "file_list"
    MediaDisplayModeEmbed    EventMediaDisplayMode = "embed"
    MediaDisplayModeLink     EventMediaDisplayMode = "link"
)

type EventMediaIcon string

const (
    MediaIconYouTube      EventMediaIcon = "youtube"
    MediaIconTwitch       EventMediaIcon = "twitch"
    MediaIconGoogleDrive  EventMediaIcon = "google_drive"
    MediaIconInstagram    EventMediaIcon = "instagram"
    MediaIconPDF          EventMediaIcon = "pdf"
    MediaIconPhotoGallery EventMediaIcon = "photo_gallery"
    MediaIconFile         EventMediaIcon = "file"
    MediaIconVideo        EventMediaIcon = "video"
    MediaIconExternalLink EventMediaIcon = "external_link"
)
```

`Event` gains `Media []EventMediaItem`. The API schema makes `id`, `type`, and `source` required;
`oapi-codegen` generates the UUID-backed Go ID and, with the current config, an optional
`Media *[]EventMediaItem` field. That pointer distinguishes an omitted property from an explicit
empty array. `openapi-typescript` generates `media?: EventMediaItem[]`. Generated types are not
runtime validators; the existing OpenAPI request-validation middleware enforces schema constraints,
and handlers enforce the type-dependent rules below.

`eventDynamo` in `dynamo/events.go` gains the matching `Media []eventMediaItemDynamo`, using the
AWS SDK tag `dynamodbav:"Media,omitempty"` (the existing event attributes use Go field names in
DynamoDB). Store each nested item ID as a string in DynamoDB and parse it back to the domain UUID.
Use `dynamodbav:"...,omitempty"` on optional nested fields, including `Icon`, so unset presentation
values are not stored as DynamoDB `NULL`s. Map items in both directions (`toDynamo`/`fromDynamo`);
validate item IDs are unique within one event. Duplicate `(type, source)` pairs remain allowed.
`Icon` is presentation-only and does not affect URL validation, link behavior, or embed eligibility.

- **Stored inline** on the `EVENT#<id>` item — no new item type, no GSI attributes touched (the
  only-EVENT/TEAM GSI rule from RFC-0001/0002 still holds).
- **Backward compatible:** items written before this RFC have no `Media` attribute and deserialize
  as `nil`/empty. Reads must not error; writes must not emit an empty attribute.
- `CreateEvent` treats an omitted or empty list as no media. On the existing `PATCH` update, an
  omitted `media` property means **preserve the current list**; an explicit `"media": []` clears it.
  `null` is invalid. This lets older clients update other event fields without erasing new media,
  while the updated admin form always sends the list (including `[]` when clearing all items).
  Preserve the distinction between absent and empty through request mapping and domain update logic.
- Order is the admin-defined slice order and is preserved as written. **No server-side sort.**

### 3.3 Validation (`api/events.go`)

Validation runs on create and update, before the repository call. Failures return the service's
standard `400` with the existing error shape.

| `type` | `source` rule |
|---|---|
| `asset_folder` | Non-empty canonical virtual path, starts with `/`, no trailing slash except root `/`, no empty, `.` or `..` segments, backslashes, query, or fragment; reject encoded separators and encoded dot-segments after one decode; ≤ 1000 characters. Do not normalize traversal segments with `path.Clean`. Does **not** require the folder to exist yet. This is an assets-api metadata path, not a filesystem or S3 object path. |
| `asset_file` | Non-empty absolute `https` URL, ≤ 1000 characters; local mode may allow loopback HTTP only. The picker uses the assets API's CDN URL. Inline rendering is limited to the configured assets CDN origin. |
| `video` | Non-empty absolute `https` URL, ≤ 1000 characters; local mode may allow loopback HTTP only. Only recognized provider hosts are embeddable; other valid URLs render as links. |
| `external_link` | Non-empty absolute `https` URL, ≤ 1000 characters; local mode may allow loopback HTTP only. |

Reject URL credentials and non-HTTP(S) schemes; do not treat `javascript:`, `data:`, protocol-relative,
or malformed URLs as valid absolute URLs. Also enforce unique item IDs, `len(media) ≤ 50`, known
`type`/`displayMode`/`icon` enum values, and `title ≤ 255` / `description ≤ 1000`. Schema bounds and enums
are checked by the request-validation middleware; handler validation repeats them for defense in
depth and checks semantic URL/path rules the schema cannot express. Keep the validator in one helper
(`validateMedia`) so create and update share it and table-driven tests cover every branch.

The item-count cap alone does not guarantee the DynamoDB 400 KB item limit: OpenAPI string lengths
count characters, but DynamoDB stores UTF-8 bytes and the event has other attributes. Cap the
serialized media list at 200 KiB in request validation and return the standard `400` before the
repository call when it exceeds that bound. The DynamoDB adapter also checks the marshaled full item
size before `PutItem` and fails closed as an internal storage error if the 400 KB limit would be
exceeded. Test worst-case multibyte values; keep this storage guard in the adapter rather than
exposing DynamoDB details through the domain. Keep the 50-item cap as a separate API/product bound.

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
   from the API — this RFC changes only their containers. The current page has no live polling yet;
   RFC-0002 defines that future behavior.
3. **Media & Livestream Sections (Smart Section Grouping)** — see §4.2.
4. **Rules Section** — the existing embedded Google Drive/PDF viewer when `rulesDocLink` is set.
   The current page shows a “Tournament Format coming soon” placeholder when it is unset; hiding
   that empty card is an intentional presentation change, not unchanged behavior.

No `row-start`/`col-start` classes remain anywhere in the page.

### 4.2 Smart Section Grouping

Media items are rendered into **type-based sections**, not in one flat list. Section order is fixed
(videos → photos → resources) so pages stay predictable; admin order is preserved
**within** each section. Empty media adds no media cards. The page layout changes for every event;
legacy event content, the livestream fallback, and any configured rules document remain.

| `type` | `displayMode` | Rendering |
|---|---|---|
| `video` | `embed` (default `auto`) | Recognized YouTube/Twitch URL in the **Videos & Livestreams** card: featured player + playlist selector |
| `video` | `link` | CTA button in the same card's footer (opens the source in a new tab) |
| `asset_folder` | `carousel`, or `auto` when the folder contains images | Own **Photo card**: Embla carousel (`CarouselImages`) + "View all" → Lightbox; mixed folders also show non-image files |
| `asset_folder` | `gallery` | Own **Photo card**: responsive thumbnail grid + Lightbox with prev/next |
| `asset_folder` | `file_list`, or `auto` when the folder has no images | Own **Photo/asset card**: downloadable file list (PDFs, stat sheets) |
| `asset_file` | `embed`, or `auto` for known images/PDFs | Featured item in the **Resources & Files** card: inline image or embedded PDF viewer |
| `asset_file` | `link` | Row in the **Resources & Files** card (open/download action) |
| `external_link` | any (`auto`/`link` expected) | Row in the **Resources & Files** card (external action) |

`auto` resolution is a pure frontend helper (`resolveMediaRender(item, assets)`) so it is testable and
identical for every call site. Incompatible combinations (e.g. `external_link` + `embed`) degrade to
safe defaults instead of erroring: folders use their `auto` folder renderer and URL-based items
become links. A folder path is never placed in an anchor or iframe. Unknown enum values are rejected
by schema validation.

#### Videos & Livestreams card (featured player + VOD selector)

- Collects all `video` items in admin order. The player selector contains only recognized embeddable
  items; link-mode and unsupported-host items are never treated as player selections. Default active
  item is the first embeddable item.
- **Featured Player:** a single `aspect-video` iframe for the active item only. Switching the
  selection swaps the `src`/`key`; **multiple iframes are never mounted at once** (no background
  players, no autoplaying audio).
- **VOD playlist selector:** a clickable list of every embeddable video (title, optional description,
  active highlight via `aria-pressed`) below the player on mobile and beside it on desktop
  (`lg:grid-cols-[minmax(0,1fr)_18rem]`). Only embedded items appear in the selector; link-mode
  items render in the footer.
- **Link-mode videos** (`displayMode: link`) render as CTA buttons in the card footer instead of in
  the selector.
- If the list contains only link-mode or unsupported videos, render its links without an iframe or
  empty player. The configured video items still suppress the default-channel fallback.
- **Fallback:** if the event has **zero** `video` items, the card renders the existing default ICAA
  Twitch & YouTube buttons (`twitch.tv/icaa_world`, `youtube.com/@Icaa-world/streams`). If the event
  has any `video` item, the fallback is not shown — admins opted into their own list.
- Twitch requires an embedded viewport of at least 400 × 300 pixels. On narrow layouts, meet that
  minimum without causing page-wide horizontal overflow or render a link instead.

#### Photo / asset folder cards (`asset_folder`)

- Each folder item is **its own Card**, in admin order, titled `item.title || "Event Photos"` with the
  optional `description` as supporting text.
- Data comes from `useGetAssets(item.source, 100)`; each card fetches independently, so one slow or
  broken folder never blocks the rest of the page. The hook is infinite-paginated, but a card that
  does not call `fetchNextPage` sees only the first 100 assets. TanStack Query de-dupes the identical
  query used by `CarouselImages`.
- `carousel` / `auto` (image folder): for 3+ images, use `CarouselImages` (Embla) with a "View all
  photos" button. Clicking the active slide or the button opens the Lightbox at that index.
  `CarouselImages` gains an optional `onImageClick?: (index: number) => void` prop (backwards
  compatible) so the parent can open the Lightbox at the right image, and an `autoplay?: boolean`
  prop that defaults to the current behavior. Event pages set `autoplay={false}`. For 1–2 images,
  render a gallery-style thumbnail/lightbox without carousel controls. In mixed folders, show
  non-image files as rows below the image view rather than silently dropping them.
- `gallery`: `grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4`, square `object-cover` thumbnails;
  clicking one opens the Lightbox at that index.
- `file_list`: rows of name, type icon, size, and an open/download action; alphabetical. The asset
  CDN is cross-origin from `icaa.world`, so `<a download>` alone does not guarantee a download.
  Verify `Content-Disposition`/CORS behavior before promising a forced download; otherwise label the
  action as opening the file.
- **Lightbox** (new `MediaLightbox` component, shadcn `Dialog`): full-screen image, prev/next
  controls, keyboard arrows, `3 / 24` counter, closes on Escape/backdrop. Index state lives in the
  card. Retain the dialog's focus trap and return focus to the image/button that opened it.
- Empty folders collapse on the public page (no dead "no photos" cards); a loading skeleton shows
  while fetching. Folders are capped at the first 100 assets in v1 (cursor paging exists in
  `useGetAssets` if a larger limit is ever needed).

#### Resources & Files card

- Images render inline (`w-full rounded-lg`) with optional title/description caption.
- PDFs render in an embedded viewer iframe at `h-[70vh]`. Encode the source URL as a query parameter
  with `URLSearchParams`; do not copy the Rules card's current unencoded string interpolation.
- Classify files conservatively. The asset URL contains a filename extension, but it is not an
  authoritative MIME type; unknown or mismatched types fall back to a resource row.
- Other file types fall back to a resource row rather than an empty frame.
- Automatically groups **all** `external_link` items and **all** `asset_file` items (featured
  embeds, link mode, and embed fallbacks for non-renderable types) into one "Resources & Files" card.
- Rows show icon (globe vs file), `item.title` (or URL host / file name), optional description, and
  the appropriate action: external link (`target="_blank"` + `rel="noreferrer noopener"`) or
  open/download. For cross-origin asset URLs, do not rely on `<a download>` alone to force a
  download.
- The card is omitted when the group is empty.
- Build every external `href` and iframe `src` from a validated URL. Unknown video hosts and
  incompatible URL types must never be assigned directly to an iframe.

#### Platform and media icons

Render an icon alongside link buttons (including the default ICAA channel buttons), video selector
rows, resource links, and file/gallery actions.
`resolveMediaIcon(item)` uses `item.icon` when present; otherwise it detects a preset from the media
type and parsed URL:

- YouTube hosts (`youtube.com`, `www.youtube.com`, `youtu.be`) → YouTube; Twitch hosts
  (`twitch.tv`, `www.twitch.tv`, `clips.twitch.tv`) → Twitch.
- `drive.google.com` / `docs.google.com` → Google Drive; `instagram.com` / `www.instagram.com` →
  Instagram. Match exact hostnames, not arbitrary substring matches.
- `.pdf` in the URL pathname (not query text) → PDF; `asset_folder` → photo gallery; other
  `asset_file`, `video`, and `external_link` items → file, video, and external-link icons.
- `EventMediaItem.icon` is an optional enum override for these presets. In the admin form,
  “Automatic” is the default and stores no `icon`; explicit selection overrides detection without
  changing URL validation or embed support.

Use local icon components or bundled SVG assets; do not fetch brand icons from remote URLs. Icons
that repeat visible button text are decorative (`aria-hidden`); button/link text remains the
accessible name. Keep detection and icon overrides strictly presentational.

### 4.3 Component shape

| Component | Responsibility |
|---|---|
| `EventPageTemplate` | Layout shell + grouping; no fetching beyond the event |
| `EventMediaSections` | Resolves `Event.media` into groups, renders section order, hides empties |
| `EventVideosCard` | Featured player, VOD selector, link CTAs, channel fallback |
| `EventFolderCard` | Per-folder card; carousel / gallery / file list + Lightbox trigger |
| `EventResourcesCard` | Featured inline files, external links, and linked files |
| `MediaLightbox` | Shared image viewer with prev/next + keyboard nav |

`CarouselImages` keeps existing behavior by default; event cards opt out of autoplay. No existing
asset-browser or upload behavior is removed.

### 4.4 Edge cases & fallbacks

- Event with no `media` at all → no media sections; the new page layout still applies, and the
  default livestream buttons remain.
- Legacy events (no `media` attribute) → `nil` slice → same as above.
- Folder deleted in `assets-api` after the event was saved → the card hides (public) after the query
  errors; other cards still render.
- Folder with mixed content → `auto` picks a carousel when any image exists and also lists
  non-image files; folders without images show only the file list.
- Video URL that cannot be embedded (unknown host, invalid ID) → render a safe link row/CTA, never a
  blank or arbitrary-origin iframe.
- Fewer images than a carousel needs (1–2) → gallery-style thumbnails/lightbox; carousel controls
  and autoplay are omitted. Event carousels are manually advanced, with visible focus states.
- Duplicate `(type, source)` pairs are allowed by the schema (same folder may appear twice) but the
  admin form surfaces a non-blocking warning.
- The 50-item cap bounds list length, but the aggregate UTF-8 serialized `EVENT` item must also stay
  below DynamoDB's 400 KB limit (§3.3).

## 5. Embed Utilities (`src/lib/embedUtils.ts`)

`buildEmbedData` is also used by `EmbedUrlDialog` and currently returns a generic iframe for any
parseable URL. Tighten that fallback to accept only absolute `http:`/`https:` URLs, rejecting
`javascript:`, `data:`, and other schemes, while retaining generic HTTP(S) embeds for existing editor
content. Do not use the generic fallback for event media. Add a strict event-media resolver (or an
explicit strict mode) that embeds only recognized YouTube/Twitch URLs and returns a link result for
unsupported hosts. Parse URLs with the platform `URL` API, require exact allowed hostnames, reject
credentials and unexpected ports, and construct iframe URLs from validated IDs rather than accepting
arbitrary iframe URLs.

This hardens the URL utility but does not by itself make the existing generic `EditorJsRenderer`
iframe safe: that call site currently combines `allow-scripts` with `allow-same-origin`. Keep event
media on the strict provider resolver and track the legacy renderer's source/sandbox policy as a
separate security follow-up.

The existing Editor.js matcher handles common YouTube `watch`, `shorts`, and `youtu.be` links, but
does not cover `/live/<id>` or a standalone `/playlist?list=...` route. Add fixtures and explicit
resolvers for these forms; do not claim they are already handled. Support YouTube videos, live
streams, and playlists. Twitch v1 support is limited to VODs, clips, and channels; Twitch collections
remain links unless collection parsing and rendering are added.

Patterns to support (exact hostnames only; case-insensitive host matching). Recognize only
`twitch.tv`, `www.twitch.tv`, `clips.twitch.tv`, `youtube.com`, `www.youtube.com`, and `youtu.be`
for embeds. The parser may accept `http` and `https` fixture URLs, but always emits an HTTPS embed
URL; event API writes reject non-HTTPS public URLs.

| URL shape | Embed target |
|---|---|
| `twitch.tv/videos/<id>` | `player.twitch.tv/?video=v<id>&parent=<host>&autoplay=false` |
| `twitch.tv/<channel>/clip/<slug>` | `clips.twitch.tv/embed?clip=<slug>&parent=<host>&autoplay=false` |
| `clips.twitch.tv/<slug>` | `clips.twitch.tv/embed?clip=<slug>&parent=<host>&autoplay=false` |
| `twitch.tv/<channel>` | `player.twitch.tv/?channel=<channel>&parent=<host>&autoplay=false` |
| `youtube.com/watch?v=<id>` | `www.youtube.com/embed/<id>` |
| `youtu.be/<id>` | `www.youtube.com/embed/<id>` |
| `youtube.com/shorts/<id>` | `www.youtube.com/embed/<id>` |
| `youtube.com/live/<id>` | `www.youtube.com/embed/<id>` |
| `youtube.com/playlist?list=<id>` | `www.youtube.com/embed/videoseries?list=<id>` |
| `youtube.com/watch?v=<id>&list=<id>` | `www.youtube.com/embed/<id>?list=<id>` |

Implementation notes:

- Resolve Twitch's required `parent` from `window.location.hostname` and guard
  `typeof window !== 'undefined'`. Construct the query with `URLSearchParams`; discard any
  user-supplied `parent` parameter so it cannot override the actual embedding host.
- A pure `parseTwitchUrl(url)` returns a discriminated union
  (`{kind: 'video' | 'clip' | 'channel', id}`), unit-testable with URL fixtures; the strict event
  media resolver returns `service: 'twitch'` with the provider-specific dimensions. Twitch VOD IDs
  must be prefixed with `v` in the player query. Reject reserved/non-channel paths such as
  `directory`, `videos` without an ID, and `clip`; validate channel names, numeric VOD IDs, and clip
  slugs.
- Build YouTube and Twitch embed URLs on their known provider origins. Unsupported or malformed URLs
  return a link result, not an iframe to the submitted URL.
- Give every iframe an accessible title and only the permissions needed by its provider. Do not
  grant `allow-same-origin` to an arbitrary source while also allowing scripts; sandbox behavior
  must be verified against the fixed provider origins rather than assuming Twitch requires this
  token.
- The repository has no checked-in CSP policy. Confirm deployed Cloudflare Pages headers and set a
  narrow `frame-src` allowlist for `https://player.twitch.tv`, `https://clips.twitch.tv`,
  `https://www.youtube.com`, and `https://drive.google.com`; do not use a wildcard.

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
  request length bounds and adds UI guards against empty names, `/`, `.` and `..`. The current
  assets-api spec does not define these additional name restrictions, and its handler does not
  enforce them; treat picker validation as UX only, not a server-side security boundary. The
  existing `ParentFolderNotFound` response path stays useful.
- **Multi-file upload** in `UploadTab`: `<input type="file" multiple>`; upload files sequentially
  through the existing presign → S3 POST → confirm flow, with progress `current / total` and
  per-file error reporting. A batch is not transactional: continue through the selected files,
  report each result, and pass only confirmed URLs to `onBatchUploaded`. Keep single-select callers
  working by calling `onSelect` for the last confirmed file. Existing cleanup is best-effort for an
  S3 POST failure; a confirm failure after upload can leave a pending asset until its normal expiry,
  so do not promise rollback of a failed batch.
- Existing behavior (`image`, `allFiles`, delete, replace, error handling) is unchanged.

### 6.2 `AdminEventForm`

Add an **"Event Media & Post-Event Assets"** card below the existing logo/rules fields:

- Zod schema gains `media: z.array(mediaItemSchema).max(50, 'At most 50 media items')`, default `[]`;
  edit mode seeds from `editData.media ?? []`; the submit body always includes `media` so an empty
  list intentionally clears all items. Each appended item gets a stable `crypto.randomUUID()` ID.
- `useFieldArray({ control, name: 'media' })` renders one row per item with:
  - a `type` select (`asset_folder | asset_file | video | external_link`),
  - a `displayMode` select limited to modes that make sense for the chosen type (plus `auto`),
  - an icon select with “Automatic” as the default and the curated preset options,
  - `source` input (read-only text field filled by the picker for asset types; plain URL input for
    `video`/`external_link`),
  - optional `title` / `description` inputs,
  - **Move up / Move down** buttons (`swap`) and **Remove** (`remove`),
- Picker triggers are type-aware: **"Browse Assets"** opens `AssetPickerModal` in `allFiles` mode for
  `asset_file`; **"Select Folder"** opens it in `folder` mode for `asset_folder` and writes the
  returned path into `source`. A **"Add Media Item"** button appends an empty row (default
  `displayMode: auto`).
- Removing a media item only removes its event reference; it never deletes the selected asset.
- Ordering in the form is the stored order; no drag-and-drop in v1 (move buttons only).
- Invalid rows block submit with inline messages; unknown/invalid URLs are caught client-side and
  again server-side (§3.3).
- Because `EventMediaItem.id` is persisted, configure `useFieldArray` with a separate key name (or
  another distinct React key) so its generated row key does not overwrite the media item's stable ID.

## 7. Implementation Progress Checklist

**Backend (`event-registration`)**

- [ ] `spec/api.yaml`: `EventMediaType`, `EventMediaDisplayMode`, `EventMediaIcon`, `EventMediaItem`
      with stable UUID `id`, `Event.media` (`maxItems: 50`) + `go generate ./...`
- [ ] Domain: `events.EventMediaItem` / enums on `Event.Media`
- [ ] DynamoDB: inline `Media` on `EVENT#<id>` (`dynamodbav:"Media,omitempty"`), map in/out,
      `CreateEvent` + `UpdateEvent`; legacy items read as empty
- [ ] `api/events.go`: `validateMedia` (canonical folder path vs HTTPS URL, lengths, type/display/
      icon enums, unique IDs, ≤ 50, aggregate byte budget) wired into create and update; tests
- [ ] API rejects media collections over 200 KiB; Dynamo adapter tests and guards the full 400 KB
      marshaled item limit without leaking storage details
- [ ] `PATCH` mapping distinguishes omitted `media` (preserve) from `media: []` (clear)
- [ ] Create response includes `media`; `eventToApiEvent` maps it in update, list, and detail responses

**Frontend — embeds & codegen**

- [ ] Regenerate events types from the updated spec. Current `bun run codegen` fetches
      `https://api.icaa.world/events/openapi.json`, so use the local/branch spec or deploy the updated
      spec before running it; generated types must not be hand-edited
- [ ] `embedUtils.ts`: strict event-media resolver for Twitch VOD/clip/channel and YouTube
      video/live/playlist; Twitch `video=v<id>`, correct `parent`, `autoplay=false`; never use the
      generic iframe fallback used by `EmbedUrlDialog`
- [ ] URL fixtures for YouTube watch/shorts/live/playlist, Twitch VOD/clip/channel/reserved paths,
      malformed URLs, overridden `parent`, unsupported hosts, PDFs, and narrow viewport behavior
- [ ] `resolveMediaIcon` tests for exact platform hosts, PDF pathname detection, type fallbacks, and
      explicit icon overrides; overrides never make unsupported URLs embeddable

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
- [ ] `MediaLightbox` (prev/next, keyboard, counter, focus management) + `CarouselImages`
      `onImageClick` and optional autoplay control (disabled on event pages)
- [ ] `EventVideosCard`: featured 16:9 player + VOD selector, link CTAs, ICAA channel fallback
- [ ] `EventFolderCard`: carousel / gallery / file_list via `useGetAssets(source, 100)`
- [ ] `EventResourcesCard` (inline image / PDF viewer, links, and linked files)
- [ ] Automatic platform/file icon detection + optional `EventMediaItem.icon` preset override;
      verify it cannot change URL validation or embed eligibility
- [ ] `EventMediaSections`: grouping order, `auto` resolution, hide empty sections
- [ ] Verify legacy events (no `media`) gain no media cards and retain the livestream fallback;
      document intentional layout and empty-rules-state changes

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
