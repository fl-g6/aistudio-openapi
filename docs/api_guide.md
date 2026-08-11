# Quickplay AI Studio Public API - Developer Guide

**API Version:** 6.3.3
**Base URL:** `https://demo.cloud.quickplay.com/genai-api`

**See also:**

- [VOD Clipping Guide](ai_studio_vod_guide.md) — end-to-end flow for pre-recorded (VOD) assets (uses `/v3/upload`).
- [Live Clipping Guide](ai_studio_live_guide.md) — end-to-end flow for live / scheduled HLS events (uses `/v3/event`).

## Authentication

All endpoints require an API key passed in the `x-api-key` header.

```
x-api-key: YOUR_API_KEY
```

## Endpoints

### POST /v3/upload

Add or update a media asset group with optional metadata. `master` and `proxy` are **objects** that each wrap a `media_assets` array, so a single request can register multiple master and proxy assets together (e.g. multi-track or multi-rendition deliveries). If a group with the same `ext_id` already exists, it is updated and the existing `media_id` is returned.

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `master` | MediaAssetV3Group | Yes | Master asset group containing a `media_assets` array. |
| `proxy` | MediaAssetV3Group | No | Proxy asset group containing a `media_assets` array. |
| `priority` | integer | No | Processing priority, `0`-`1000` (default `0`). `0` is lowest, `1000` is highest. Jobs are processed based on this priority and request time -- earlier requests win within the same priority. |
| `email` | string | No | Email address of the requesting user. |
| `ai_flags` | string[] | No | AI processing flags. See [AI flags](#ai-flags) below for allowed values and combination rules. When provided, the request is processed **asynchronously** and returns HTTP 202 with a `job_id`. |
| `metadata` | array | No | Localized metadata entries. |

**MediaAssetV3Group fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_id` | string | Yes | Identifier for this asset group in the external MAM / DAM system. |
| `media_assets` | MediaAssetV3Item[] | Yes | One or more media assets in the group. At least one entry. |

**MediaAssetV3Item fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_asset_id` | string | Yes | Identifier for the asset within this upload group |
| `uri` | string | Yes | Location of the asset. Supported schemes: `https`, `s3`, `gcs` |
| `duration` | number | No | Duration of the media asset in seconds. |
| `size` | integer | No | Size of the media asset in bytes. |
| `media_profile` | MediaProfileTrack[] | No | Inline media profile (video/audio/cc tracks). See [Media Profile](#media-profile) below for track structure. Optional — when omitted, the profile is inferred from the asset. |

Note: only inline `media_profile` is accepted; a pre-configured profile name is not supported.

**Metadata entry fields:**

| Field | Type | Description |
|-------|------|-------------|
| `language` | string | Language code (e.g. `en_CA`) |
| `title` | string | Asset title |
| `description` | string | Asset description |

#### AI flags

Supported values for `ai_flags`:

| Value | Description |
|-------|-------------|
| `GENERATE_METADATA` | Auto-generate source metadata |
| `DETECT_SCENES` | Auto-detect scenes in the asset |
| `REFRAME_9_16` | Smart Crop to 9:16 aspect ratio |
| `REFRAME_1_1` | Smart Crop to 1:1 aspect ratio |
| `REFRAME_3_4` | Smart Crop to 3:4 aspect ratio |

Any non-empty subset of these values is accepted; values must be unique. Multiple reframe aspect ratios can be requested in a single call, and reframing can be combined with metadata generation and/or scene detection.

**Valid examples:**

```json
["GENERATE_METADATA"]
["DETECT_SCENES"]
["REFRAME_9_16"]
["REFRAME_9_16", "REFRAME_1_1", "REFRAME_3_4"]
["GENERATE_METADATA", "DETECT_SCENES", "REFRAME_9_16"]
```

**Invalid examples:**

```json
[]                                                        // empty array not allowed
["REFRAME_9_16", "REFRAME_9_16"]                // duplicate values
```

When `ai_flags` is provided the upload is processed **asynchronously** and the API returns **202 Accepted** with only a `job_id` and `status` of `Queued`. Poll `/v2/status` with the returned `job_id` to track progress.

#### Media Profile

When providing an inline `media_profile`, each entry is a track with a `group_type` discriminator:

**Video track** (`group_type: video`)

| Field | Type | Description |
|-------|------|-------------|
| `track_info.track_id` | string | Opaque caller-supplied track identifier (e.g. an asset ID from an upstream MAM system). Optional. |
| `track_info.codec` | string | Master: `mpeg-2-video`, `dnxhd-sq`, `avc-intra`, `prores422-hq`, `jpeg2000`, `prores422`, `dnxhr-sq`, `xavc-i`, `dnxhr-hq`, `prores4444`, `prores422-lt`, `dnxhd-hq`, `dnxhd`, `dnxhd-hqx`, `dnxhr-444`, `dnxhr-hqx`, `dnxhd-lb`, `dnxhd-tr`. Proxy: only `avc`. |
| `track_info.bitRate` | integer | Bit rate in bits per second |
| `track_info.frameRate` | object | `{ numerator, denominator }` (e.g. 30000/1001 for 29.97fps) |
| `track_info.frameSize` | object | `{ width, height }` in pixels |
| `track_info.offset_sec` | number | Offset of the track relative to the primary timeline, in seconds. |

**Audio track** (`group_type: audio`)

| Field | Type | Description |
|-------|------|-------------|
| `audio_kind` | string | Audio content type: `PRM`, `SAP`, `HI`, `DV`, `DX`, `MX`, `FX`, `FFX`, `ME`, `OP`, `MESP`, `DME`, `NDME`, `PNAR`, `ONAR`, `VO`, `VI`, `CM`, `LCM`, `MOS` |
| `audio_group` | string | Channel configuration: `ST`, `DM`, `LtRt`, `DNS`, `3.0`, `4.0`, `5.0`, `5.1`, `5.1EX`, `6.0`, `6.1`, `7.0DS`, `7.1DS`, `7.1SDS`, `HA`, `VA` |
| `language` | string | Language code (e.g. `en-US`) |
| `track_info.track_id` | string | Opaque caller-supplied track identifier (e.g. an asset ID from an upstream MAM system). Optional. |
| `track_info.codec` | string | `aac`, `pcm` |
| `track_info.bitRate` | integer | Bit rate in bits per second |
| `track_info.sampleRate` | integer | Sample rate in Hz |
| `track_info.offset_sec` | number | Offset of the track relative to the primary timeline, in seconds. Used to align sidecar audio files. |
| `channel_layouts` | array | Channel mapping entries with `channel_locator` (required) and `audio_channel` |

Supported `audio_channel` values: `L`, `R`, `C`, `LFE`, `Ls`, `Rs`, `Lss`, `Rss`, `Lrs`, `Rrs`, `Lc`, `Rc`, `Cs`, `HI`, `VIN`, `Lt`, `Rt`, `Lst`, `Rst`, `S`, `M1`, `M2`

**Closed caption track** (`group_type: cc`)

| Field | Type | Description |
|-------|------|-------------|
| `track_id` | string | Opaque caller-supplied track identifier (e.g. an asset ID from an upstream MAM system). Optional. |
| `format` | string | `CEA-608`, `EIA-608`, `CEA-708`, `EIA-708`, `SRT`, `TTML`, `WebVTT`, `PAC`, `SCC`, `IMSC` |
| `language` | string | Language code (e.g. `en-US`) |
| `offset_sec` | number | Offset relative to the primary timeline, in seconds; used to align sidecar caption files. Optional. |

#### Example: Master only (no proxy)

```json
{
  "master": {
    "ext_id": "asset-group-003",
    "media_assets": [
      {
        "ext_asset_id": "master-video-003",
        "uri": "s3://bucket/media/video.mxf",
        "duration": 120.5,
        "size": 7516192768,
        "media_profile": [
          {
            "group_type": "video",
            "track_info": {
              "codec": "prores422",
              "bitRate": 50000000,
              "frameRate": { "numerator": 30000, "denominator": 1001 },
              "frameSize": { "width": 1920, "height": 1080 },
              "offset_sec": 0
            }
          }
        ]
      }
    ]
  },
  "priority": 500,
  "email": "producer@example.com",
  "metadata": [
    {
      "language": "en-US",
      "title": "Sample Video",
      "description": "A master-only upload"
    }
  ]
}
```

#### Example: Multiple master assets with corresponding proxies

```json
{
  "master": {
    "ext_id": "asset-group-001",
    "media_assets": [
      {
        "ext_asset_id": "master-video-001",
        "uri": "s3://bucket/media/video.mxf",
        "duration": 120.5,
        "size": 7516192768,
        "media_profile": [
          {
            "group_type": "video",
            "track_info": {
              "codec": "prores422",
              "bitRate": 50000000,
              "frameRate": { "numerator": 30000, "denominator": 1001 },
              "frameSize": { "width": 1920, "height": 1080 },
              "offset_sec": 0
            }
          }
        ]
      },
      {
        "ext_asset_id": "master-audio-001",
        "uri": "s3://bucket/media/audio.wav",
        "duration": 120.5,
        "size": 34603008,
        "media_profile": [
          {
            "group_type": "audio",
            "audio_kind": "PRM",
            "audio_group": "ST",
            "language": "en-US",
            "track_info": {
              "codec": "pcm",
              "bitRate": 2304000,
              "sampleRate": 48000,
              "offset_sec": 0
            },
            "channel_layouts": [
              { "channel_locator": "1", "audio_channel": "L" },
              { "channel_locator": "2", "audio_channel": "R" }
            ]
          }
        ]
      },
      {
        "ext_asset_id": "master-cc-001",
        "uri": "s3://bucket/media/captions.scc",
        "duration": 120.5,
        "size": 1362,
        "media_profile": [
          {
            "group_type": "cc",
            "format": "SCC",
            "language": "en",
            "offset_sec": -1.18
          }
        ]
      }
    ]
  },
  "proxy": {
    "ext_id": "asset-group-001-proxy",
    "media_assets": [
      {
        "ext_asset_id": "proxy-video-001",
        "uri": "s3://bucket/media/video_proxy.mp4",
        "duration": 120.5,
        "size": 75161927,
        "media_profile": [
          {
            "group_type": "video",
            "track_info": {
              "codec": "avc",
              "bitRate": 5000000,
              "frameRate": { "numerator": 30000, "denominator": 1001 },
              "frameSize": { "width": 1280, "height": 720 },
              "offset_sec": 0
            }
          }
        ]
      }
    ]
  },
  "priority": 500,
  "email": "producer@example.com",
  "metadata": [
    {
      "language": "en-US",
      "title": "Sample Video",
      "description": "A sample multi-asset upload"
    }
  ]
}
```

#### Responses

- **200 OK** -- asset group already exists
- **201 Created** -- asset group created
- **202 Accepted** -- returned when `ai_flags` is provided; body is `{ "job_id": "...", "status": "Queued" }`
- **400 / 401 / 403** -- standard error responses

The 200/201 response body matches the upload response fields: `job_id`, `ext_id`, `uri`, `media_id`, `duration`, `mimeType`, `status`.

---

### POST /v3/event

Register a live or scheduled event for ingestion and AI processing. The event is identified by a source URL plus a **single** schedule (start/end). Each POST registers **one** event occurrence — **recurrence is not supported**. `output_format` selects whether the output is a VOD asset (default) or a reframed live stream. Processing is **always asynchronous**; the API returns **202 Accepted** with a `job_id` that can be polled via `/v2/status`.

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_job_id` | string | Yes | External job identifier for this event registration. |
| `output_format` | string | No | `VOD` (default) or `Live`. Selects the output type — see [output_format](#output_format) below. |
| `media_assets` | EventMediaAsset | Yes | Single event media asset. Accepted URL scheme depends on `output_format` — see below. |
| `schedule` | EventSchedule | **Yes** | Single schedule entry describing when the event occurs. |
| `email` | string | **Yes** | Email address of the requesting user. |
| `priority` | integer | No | Processing priority, `0`-`1000` (default `0`). Same semantics as `/v3/upload`. |
| `ai_flags` | string[] | No | AI processing flags. Same combination rules as `/v3/upload` — see [AI flags](#ai-flags) under `/v3/upload`. `/v3/event` is already asynchronous (returns 202); `ai_flags` selects **what** AI processing to run, not whether the request is async. When `output_format = Live`, the `REFRAME_*` flags drive the live reframe. |

#### output_format

`output_format` controls the shape of the delivered output and constrains which input formats are accepted on `media_assets.uri`:

| Value | Output | Accepted `media_assets.uri` |
|-------|--------|-----------------------------|
| `VOD` *(default)* | Video-on-demand asset | **HLS only** (`https://…/master.m3u8`) |
| `Live` | Live stream, reframed per the `REFRAME_*` flags supplied in `ai_flags` | **HLS TS** (`https://…/master.m3u8` with MPEG-TS segments) **or SRT** (`srt://…`) |

**EventMediaAsset fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_asset_id` | string | Yes | Identifier for the event media asset within this event. |
| `uri` | string | Yes | Source URL for the live or scheduled event stream. Accepted schemes depend on `output_format` (see above). |

**EventSchedule fields** (no additional properties beyond these):

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_event_id` | string | Yes | External identifier for this scheduled occurrence. |
| `start` | EventDateTime | Yes | When the event starts. |
| `end` | EventDateTime | Yes | When the event ends. |
| `metadata` | array | No | Localized metadata entries (`language`, `title`, `description`) for this schedule. |

**EventDateTime fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `date_time` | string (date-time) | Yes | RFC 3339 timestamp with UTC `Z` suffix or an explicit offset (e.g. `2026-06-15T19:00:00Z` or `2026-06-15T15:00:00-04:00`). |

#### Example: Single live event

```json
{
  "ext_job_id": "event-group-001",
  "media_assets": {
    "ext_asset_id": "live-event-001",
    "uri": "https://example.com/live/event-001/master.m3u8"
  },
  "schedule": {
    "ext_event_id": "event-001-occurrence",
    "start": { "date_time": "2026-06-15T19:00:00Z" },
    "end":   { "date_time": "2026-06-15T21:00:00Z" },
    "metadata": [
      { "language": "en-US", "title": "Live Sports Event", "description": "A single live broadcast" }
    ]
  },
  "priority": 500,
  "email": "producer@example.com"
}
```

#### Example: Single live event with AI processing flags (VOD output)

```json
{
  "ext_job_id": "event-group-002",
  "output_format": "VOD",
  "media_assets": {
    "ext_asset_id": "live-event-002",
    "uri": "https://example.com/live/event-002/master.m3u8"
  },
  "schedule": {
    "ext_event_id": "event-002-occurrence",
    "start": { "date_time": "2026-06-15T19:00:00Z" },
    "end":   { "date_time": "2026-06-15T21:00:00Z" },
    "metadata": [
      { "language": "en-US", "title": "Weekly Show" }
    ]
  },
  "priority": 500,
  "email": "producer@example.com",
  "ai_flags": ["GENERATE_METADATA", "DETECT_SCENES"]
}
```

#### Example: Live output with reframing (SRT input)

```json
{
  "ext_job_id": "event-group-003",
  "output_format": "Live",
  "media_assets": {
    "ext_asset_id": "live-event-003",
    "uri": "srt://ingest.example.com:9000?streamid=event-003"
  },
  "schedule": {
    "ext_event_id": "event-003-occurrence",
    "start": { "date_time": "2026-06-15T19:00:00Z" },
    "end":   { "date_time": "2026-06-15T21:00:00Z" },
    "metadata": [
      { "language": "en-US", "title": "Live Reframed Broadcast" }
    ]
  },
  "priority": 500,
  "email": "producer@example.com",
  "ai_flags": ["REFRAME_9_16", "REFRAME_1_1"]
}
```

#### Example: Live output with reframing (HLS TS input)

```json
{
  "ext_job_id": "event-group-004",
  "output_format": "Live",
  "media_assets": {
    "ext_asset_id": "live-event-004",
    "uri": "https://example.com/live/event-004/master.m3u8"
  },
  "schedule": {
    "ext_event_id": "event-004-occurrence",
    "start": { "date_time": "2026-06-15T19:00:00Z" },
    "end":   { "date_time": "2026-06-15T21:00:00Z" }
  },
  "priority": 500,
  "email": "producer@example.com",
  "ai_flags": ["REFRAME_9_16"]
}
```

#### Responses

- **202 Accepted** -- event registration queued; body is `{ "job_id": "...", "status": "Queued" }`. Poll `/v2/status` with the `job_id`.
- **400 / 401 / 403** -- standard error responses.

---

### PUT /v3/event/{ext_job_id}

Update an existing event registration. Only `schedule`, `priority`, `email`, and `ai_flags` may be edited. The identity fields (`ext_job_id`, `media_assets`) are **immutable** — to change them, delete and re-register the event. `email` is **required** on every update, and at least one of `schedule`, `priority`, or `ai_flags` must also be supplied. Any field omitted from the request is left unchanged. Processing is asynchronous (same semantics as `POST /v3/event`).

#### Path Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_job_id` | string | Yes | External job identifier previously supplied on `POST /v3/event`. |

#### Request Body

`email` is required. At least one of `schedule`, `priority`, or `ai_flags` must also be supplied.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | **Yes** | Email address of the requesting user. Sent on every update for audit / notification. |
| `schedule` | EventSchedule | Conditional | Replacement schedule. **Full replacement** — supplying this replaces the existing schedule. Field shape matches `POST /v3/event`. |
| `priority` | integer | Conditional | Processing priority, `0`-`1000`. |
| `ai_flags` | string[] | Conditional | AI processing flags. Same enum as `POST /v3/event` — see [AI flags](#ai-flags). |

#### Example: Reschedule an event

```json
{
  "email": "producer@example.com",
  "schedule": {
    "ext_event_id": "event-001-occurrence-rescheduled",
    "start": { "date_time": "2026-06-22T19:00:00Z" },
    "end":   { "date_time": "2026-06-22T21:00:00Z" },
    "metadata": [
      { "language": "en-US", "title": "Live Sports Event (Rescheduled)" }
    ]
  }
}
```

#### Example: Adjust priority only

```json
{
  "email": "producer@example.com",
  "priority": 800
}
```

#### Example: Change AI processing flags

```json
{
  "email": "producer@example.com",
  "ai_flags": ["GENERATE_METADATA", "REFRAME_9_16", "REFRAME_1_1"]
}
```

#### Responses

- **202 Accepted** -- update queued; body is `{ "job_id": "...", "status": "Queued" }`. Poll `/v2/status` with the `job_id`.
- **400** -- `email` is missing, no other editable field supplied, or the API key is invalid/expired.
- **401 / 403** -- standard error responses.
- **404** -- no event exists for the supplied `ext_job_id`.

---

### POST /v3/smart-crop

Generate a video from a sequence of media assets and export to a destination based on a predefined export template. Media assets referenced in `sequence[].ext_id` must be registered — **either** previously via `/v3/upload`, **or** inline in this request via the optional `media_assets` field (equivalent to invoking `/v3/upload` first).

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_job_id` | string | Yes | External job identifier |
| `email` | string | **Yes** | Email address of the requesting user. |
| `sequence` | array | Yes | Ordered list of media asset clips |
| `media_assets` | MediaAssetV3Item[] | No | Optional inline asset registration. Each item has the same shape as a `/v3/upload` `master.media_assets[]` entry — see [POST /v3/upload](#post-v3upload). Omit when the referenced assets have already been registered via `/v3/upload`. |
| `export_template_name` | string[] | No | Names of the export templates to apply |
| `priority` | integer | No | Processing priority, `0`-`1000` (default `0`). `0` is lowest, `1000` is highest. Jobs are processed based on this priority and request time -- earlier requests win within the same priority. |

**Sequence entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_id` | string | Yes | External asset identifier — previously registered via `/v3/upload`. |
| `start_timecode` | string | No | Start timecode for the clip |
| `end_timecode` | string | No | End timecode for the clip |

#### Example: Pre-registered assets

```json
{
  "ext_job_id": "my-crop-job-001",
  "email": "producer@example.com",
  "sequence": [
    {
      "ext_id": "asset-001",
      "start_timecode": "00:00:10:00",
      "end_timecode": "00:01:30:00"
    },
    {
      "ext_id": "asset-002"
    }
  ],
  "export_template_name": ["HD-1080p-Stereo", "Proxy-720p"]
}
```

#### Example: Inline asset registration + smart-crop

Registers the media assets inline as part of this call, then clips them — equivalent to calling `/v3/upload` followed by `/v3/smart-crop`. Note that `media_profile` is optional; when omitted, the profile is inferred from the asset.

```json
{
  "ext_job_id": "ext-job-002",
  "email": "producer@example.com",
  "media_assets": [
    {
      "ext_asset_id": "master-video-005",
      "uri": "http://<HLS URL>",
      "duration": 120.5,
      "size": 7516192768
    }
  ],
  "sequence": [
    {
      "ext_id": "asset-group-005",
      "start_timecode": "00:00:05:00",
      "end_timecode": "00:00:20:00"
    }
  ],
  "export_template_name": ["HD-1080p-Stereo"],
  "priority": 500
}
```

#### Responses

**202 Accepted** -- Job accepted.

```json
{
  "job_id": "1015825c-7c90-4042-8513-24585fb9c235"
}
```

---

### POST /v3/smart-crop/{ext_job_id}/publish

Publish (render and export) the result of an existing smart-crop job using one or more export templates. The video is rendered once per template in `export_template_name`.

The smart-crop job referenced by `ext_job_id` must have been created via `/v3/smart-crop`.

#### Path Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `ext_job_id` | string | Yes | External Job ID of the smart-crop job to publish. Must match the `ext_job_id` used when the job was created via `/v3/smart-crop`. |

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `export_template_name` | string[] | Yes | Names of the export templates to render and export. Must contain at least one entry. |
| `priority` | integer | No | Processing priority, `0`-`1000` (default `0`). Same semantics as `/v3/smart-crop`. |
| `email` | string | No | Email address of the requesting user. |

#### Example: Publish using multiple export templates

```
POST /v3/smart-crop/my-crop-job-001/publish
```

```json
{
  "export_template_name": ["HD-1080p-Stereo", "Proxy-720p"],
  "priority": 0,
  "email": "producer@example.com"
}
```

#### Responses

**202 Accepted** -- Publish job accepted.

```json
{
  "job_id": "1015825c-7c90-4042-8513-24585fb9c235"
}
```

**404 Not Found** -- No smart-crop job with the given `ext_job_id`.

**400 / 401 / 403** -- Standard error responses.

---

### DELETE /v3/job/{ext_job_id}

Delete an existing job. Works for jobs created via **either** `POST /v3/event` (event registration) **or** `POST /v3/smart-crop` (smart-crop job). Removes the job and cancels any **pending** processing associated with the given `ext_job_id`. Any downstream artifacts already produced (e.g. published renders) are **unaffected**. `email` is **required** in the request body (identifies the requesting user for audit purposes).

#### Path Parameters

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_job_id` | string | Yes | External job identifier previously supplied on `POST /v3/event` or `POST /v3/smart-crop`. |

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string | **Yes** | Email address of the requesting user. |

```json
{
  "email": "producer@example.com"
}
```

#### Responses

- **204 No Content** -- job deleted successfully. No response body.
- **400** -- `email` is missing, or the API key is invalid/expired.
- **401 / 403** -- standard error responses.
- **404** -- no job exists for the supplied `ext_job_id`.

---

### POST /v2/status

Get the status of one or more jobs. Query by **either** AI Studio job IDs or external job IDs (not both).

#### Request Body

Provide **exactly one** of:

| Field | Type | Description |
|-------|------|-------------|
| `job_id` | string[] | AI Studio job IDs |
| `ext_job_id` | string[] | External job IDs |

#### Example: Query by job ID

```json
{
  "job_id": [
    "1015825c-7c90-4042-8513-24585fb9c235",
    "2a3b4c5d-1234-5678-abcd-ef0123456789"
  ]
}
```

#### Example: Query by external job ID

```json
{
  "ext_job_id": ["ext-job-001", "ext-job-002"]
}
```

#### Response

Returns an array of job status objects. The shape varies by status:

**Common fields (all statuses):**

| Field | Type | Description |
|-------|------|-------------|
| `job_id` | string | AI Studio job ID |
| `ext_job_id` | string | External job ID |
| `status` | string | `Queued`, `Processing`, `Completed`, or `Failed` |
| `progress_percent` | integer | Completion percentage (0-100) |

**Additional fields when `status: Completed`:**

| Field | Type | Description |
|-------|------|-------------|
| `media_assets` | array | Output assets produced by the job |
| `media_assets[].uri` | string | Location of the output asset |
| `media_assets[].asset_type` | string | Type of the output asset (e.g. `video`, `proxy`) |

**Additional fields when `status: Failed`:**

| Field | Type | Description |
|-------|------|-------------|
| `error.code` | integer | Error code |
| `error.message` | string | Error message |

#### Response examples

**Queued:**
```json
[
  {
    "job_id": "1015825c-7c90-4042-8513-24585fb9c235",
    "ext_job_id": "ext-job-001",
    "status": "Queued",
    "progress_percent": 0
  }
]
```

**Completed:**
```json
[
  {
    "job_id": "3c4d5e6f-2345-6789-bcde-f01234567890",
    "ext_job_id": "ext-job-003",
    "status": "Completed",
    "progress_percent": 100,
    "media_assets": [
      { "uri": "s3://bucket/output/file.mp4", "asset_type": "video" },
      { "uri": "s3://bucket/output/file_proxy.mp4", "asset_type": "proxy" }
    ]
  }
]
```

**Failed:**
```json
[
  {
    "job_id": "4d5e6f7a-3456-789a-cdef-012345678901",
    "ext_job_id": "ext-job-004",
    "status": "Failed",
    "progress_percent": 30,
    "error": {
      "code": 400342,
      "message": "Invalid Video"
    }
  }
]
```

---

### POST /moments/search

Search across previously uploaded media for moments matching one or more natural-language prompts. Returns AI-curated clips (start/end timestamps, preview thumbnail, title, hashtags, confidence score, and a human-readable relevance reason) ranked by relevance.

The search can be scoped:

- **All media** (no `media_id` and no `external_id`) -- search across every asset the API key can see.
- **By internal IDs** -- pass `media_id` (UUIDs returned by `/v3/upload`).
- **By external IDs** -- pass `external_id` (the `ext_id` values you supplied when uploading).

`media_id` and `external_id` are **mutually exclusive** -- include at most one.

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompts` | string[] | Yes | One or more natural-language prompts describing the desired content (e.g. `"Highlight moments from this video"`). |
| `media_id` | string[] (uuid) | No | Restrict the search to these internal media IDs. Mutually exclusive with `external_id`. |
| `external_id` | string[] | No | Restrict the search to these external IDs. Mutually exclusive with `media_id`. |

#### Example: Search by external ID

```json
{
  "prompts": ["Highlight moments from this video"],
  "external_id": ["26714368-a156-4d7a-b667-45ac6c08e79a"]
}
```

#### Example: Search across all media

```json
{
  "prompts": ["Verstappen winning the Japanese GP"]
}
```

#### Response

Returns an array of moment objects, sorted by `rank` (1 = most relevant).

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `start` | integer | Yes | Start time of the moment, in seconds, within the source media. |
| `end` | integer | Yes | End time of the moment, in seconds. |
| `media_id` | string (uuid) | Yes | Internal identifier of the source media asset. |
| `media_external_id` | string | No | External identifier supplied at upload time. |
| `confidence_score` | number | Yes | Similarity score against the prompt (0.0 – 1.0). |
| `rank` | integer | Yes | Relevance ranking; `1` is the strongest match. |
| `preview_uri` | string | No | URL of a thumbnail/preview frame for this moment. |
| `relevance_reason` | string | No | Plain-language explanation of why the moment matched. |
| `title` | string | No | Generated title for the moment. |
| `hashtags` | string | No | Comma-separated tags attached to the moment. |

#### Example response

```json
[
  {
    "start": 387,
    "end": 457,
    "media_id": "db708bc2-ae70-453e-9839-bc07e84ce160",
    "media_external_id": "db708bc2-ae70-453e-9839-bc07e84ce160",
    "confidence_score": 0.5826060175895691,
    "rank": 1,
    "preview_uri": "https://sandbox2.dev.quickplay.com/media-preview/.../db708bc2-ae70-453e-9839-bc07e84ce160-0000000064.jpeg",
    "title": "Max Verstappen's Dominant Japanese GP Victory",
    "hashtags": "#F1,#JapaneseGP,#MaxVerstappen,#RedBullRacing,#Winner,#Suzuka",
    "relevance_reason": "This entry directly matches the query by describing Max Verstappen's dominant victory at the Japanese Grand Prix, mentioning his win, the location (Suzuka), and the key players (Verstappen, Norris, Piastri)."
  },
  {
    "start": 458,
    "end": 482,
    "media_id": "db708bc2-ae70-453e-9839-bc07e84ce160",
    "confidence_score": 0.5418363809585571,
    "rank": 3,
    "title": "Verstappen's Japanese GP Victory Celebration",
    "hashtags": "#F1,#JapaneseGP,#Verstappen,#RedBullRacing,#Podium"
  }
]
```

#### Responses

- **200 OK** -- array of matching moments, sorted by `rank` (empty array if no matches).
- **400 / 401 / 403** -- standard error responses.

---

## Typical Workflow

1. **Upload assets** -- `POST /v3/upload` to register the master (and optional proxy) asset groups
2. **Create a smart-crop job** -- `POST /v3/smart-crop` with the sequence of uploaded assets
3. **Poll for status** -- `POST /v2/status` with the `job_id` from step 2 until status is `Completed` or `Failed`
4. **(Optional) Discover moments** -- once an asset's AI processing has finished, `POST /moments/search` with one or more prompts to retrieve ranked, AI-curated clips for use in subsequent crop or publish jobs.

---

## Error Responses

All endpoints return the same error format for standard HTTP errors:

**400 Bad Request** (invalid or expired API key):
```json
{
  "error": {
    "code": 400,
    "message": "API key not valid. Please pass a valid API key.",
    "status": "INVALID_ARGUMENT",
    "details": [
      {
        "@type": "type.googleapis.com/google.rpc.ErrorInfo",
        "reason": "API_KEY_INVALID",
        "domain": "googleapis.com"
      }
    ]
  }
}
```

**401 Unauthorized** (missing API key):
```json
{
  "message": "UNAUTHENTICATED: Method doesn't allow unregistered callers",
  "code": 401
}
```

**403 Forbidden** (permission denied):
```json
{
  "message": "PERMISSION_DENIED: The API targeted by this request is invalid for the given API key.",
  "code": 403
}
```
