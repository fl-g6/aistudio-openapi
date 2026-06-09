# Quickplay AI Studio Public API - Developer Guide

**API Version:** 6.3.1
**Base URL:** `https://verticalizer.videoai.wbd.com/genai-api`

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
| `media_profile` | MediaProfileTrack[] | Yes | Inline media profile (video/audio/cc tracks). See [Media Profile](#media-profile) below for track structure. |

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
| `AUTO_GENERATE_ALL` | Run all auto-generation steps |
| `AUTO_GENERATE_DEFAULT` | Run the default auto-generation steps |
| `AUTO_GENERATE_SOURCE_METADATA` | Auto-generate source metadata |
| `AUTO_DETECT_SCENES` | Auto-detect scenes in the asset |

The array must match **exactly one** of the following combinations:

1. `[AUTO_GENERATE_ALL]` by itself
2. `[AUTO_GENERATE_DEFAULT]` by itself
3. Any non-empty subset of `{AUTO_GENERATE_SOURCE_METADATA, AUTO_DETECT_SCENES}` (either one, or both)

`AUTO_GENERATE_ALL` and `AUTO_GENERATE_DEFAULT` are mutually exclusive with each other **and** with the granular flags. For example, `[AUTO_GENERATE_ALL, AUTO_DETECT_SCENES]` and `[AUTO_GENERATE_DEFAULT, AUTO_GENERATE_SOURCE_METADATA]` are both invalid.

**Valid examples:**

```json
["AUTO_GENERATE_ALL"]
["AUTO_GENERATE_DEFAULT"]
["AUTO_GENERATE_SOURCE_METADATA"]
["AUTO_DETECT_SCENES"]
["AUTO_GENERATE_SOURCE_METADATA", "AUTO_DETECT_SCENES"]
```

**Invalid examples:**

```json
["AUTO_GENERATE_ALL", "AUTO_GENERATE_DEFAULT"]
["AUTO_GENERATE_ALL", "AUTO_DETECT_SCENES"]
["AUTO_GENERATE_DEFAULT", "AUTO_GENERATE_SOURCE_METADATA"]
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

### POST /v2/smart-crop

Generate a video from a sequence of media assets and export to a destination based on a predefined export template. All media assets in the sequence must have been previously added via `/v3/upload`.

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_job_id` | string | Yes | External job identifier |
| `sequence` | array | Yes | Ordered list of media asset clips |
| `export_template_name` | string[] | No | Names of the export templates to apply |
| `priority` | integer | No | Processing priority, `0`-`1000` (default `0`). `0` is lowest, `1000` is highest. Jobs are processed based on this priority and request time -- earlier requests win within the same priority. |
| `email` | string | No | Email address of the requesting user. |

**Sequence entry fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ext_id` | string | Yes | External asset identifier. Must be the **master** group `ext_id` (not the proxy) previously registered via `/v3/upload`. |
| `start_timecode` | string | No | Start timecode for the clip |
| `end_timecode` | string | No | End timecode for the clip |

#### Example

```json
{
  "ext_job_id": "my-crop-job-001",
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

#### Responses

**202 Accepted** -- Job accepted.

```json
{
  "job_id": "1015825c-7c90-4042-8513-24585fb9c235"
}
```

---

### POST /v2/smart-crop/{ext_job_id}/publish

Publish (render and export) the result of an existing smart-crop job using one or more export templates. The video is rendered once per template in `export_template_name`.

The smart-crop job referenced by `ext_job_id` must have been created via `/v2/smart-crop`.

#### Path Parameters

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `ext_job_id` | string | Yes | External Job ID of the smart-crop job to publish. Must match the `ext_job_id` used when the job was created via `/v2/smart-crop`. |

#### Request Body

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `export_template_name` | string[] | Yes | Names of the export templates to render and export. Must contain at least one entry. |
| `priority` | integer | No | Processing priority, `0`-`1000` (default `0`). Same semantics as `/v2/smart-crop`. |
| `email` | string | No | Email address of the requesting user. |

#### Example: Publish using multiple export templates

```
POST /v2/smart-crop/my-crop-job-001/publish
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

## Typical Workflow

1. **Upload assets** -- `POST /v3/upload` to register the master (and optional proxy) asset groups
2. **Create a smart-crop job** -- `POST /v2/smart-crop` with the sequence of uploaded assets
3. **Poll for status** -- `POST /v2/status` with the `job_id` from step 2 until status is `Completed` or `Failed`

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
