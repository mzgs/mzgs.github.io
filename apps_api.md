# API Documentation

Base URL: `https://apps.mzgs.net`

This document covers the key-authenticated Apps API and Sensor Tower keyword
search API.

## Authentication

Set the shared key in **Admin > Settings > API Access Key**, or set the
`API_ACCESS_KEY` environment variable. Send the key as a Bearer token:

```http
Authorization: Bearer YOUR_API_KEY
```

API responses use `application/json; charset=utf-8`.

## Apps API

The Apps API reads and writes the same non-backup records shown on the Apps
page. It supports get, insert, update, and file upload. There is no
delete endpoint.

### Endpoints

| Method | Path | Description | Success |
| --- | --- | --- | --- |
| `GET` | `/api/apps` | Get all non-backup apps | `200` |
| `GET` | `/api/apps/{id}` | Get one non-backup app | `200` |
| `POST` | `/api/apps` | Insert an app | `201` |
| `PUT` | `/api/apps/{id}` | Update only the supplied fields | `200` |
| `POST` | `/api/apps/{id}/uploads` | Upload an iOS or Android asset | `201` |

`{id}` must be a positive integer. The list endpoint has no pagination or
filters. It sorts by `show_in_dashboard` descending, `app_order` ascending, and
`id` descending.

### Get all apps

```bash
curl "https://apps.mzgs.net/api/apps" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Response:

```json
{
  "data": [
    {
      "id": 18,
      "name": "Example App",
      "platform": "apple",
      "package_name": "com.example.app",
      "is_active": 1
    }
  ],
  "count": 1
}
```

The objects contain every database field for the app. `data`, `ios_locales`,
and `android_locales` are decoded into JSON values in responses. Integer and
boolean-like fields are returned as integers; nullable fields can be `null`.

### Get one app

```bash
curl "https://apps.mzgs.net/api/apps/18" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Response:

```json
{
  "data": {
    "id": 18,
    "name": "Example App",
    "platform": "apple",
    "package_name": "com.example.app"
  }
}
```

### Insert an app

Send a JSON object with `Content-Type: application/json` and at least one
writable field. No individual field is required. Fields not supplied use
database/API defaults.

```bash
curl -X POST "https://apps.mzgs.net/api/apps" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Example App",
    "platform": "apple",
    "package_name": "com.example.app",
    "is_active": true
  }'
```

Defaults applied when the field is omitted:

| Field | Default |
| --- | --- |
| `platform` | `"apple"` |
| `app_order` | `0` |
| `is_active` | `1` |
| `is_backup` | `0` (internal, not writable) |

The response has the same `{"data": {...}}` shape as the single-app endpoint,
uses status `201`, and includes `Location: /api/apps/{id}`.

### Update an app

Updates are partial: only supplied fields change. At least one writable field
must be supplied.

```bash
curl -X PUT "https://apps.mzgs.net/api/apps/18" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated App",
    "app_order": 2,
    "show_in_dashboard": true,
    "is_active": true
  }'
```

The response contains the complete updated record in
`{"data": {...}}`.

### Upload files

Create the app first, then use its returned ID to upload assets. Send a
`multipart/form-data` request with a `type` field and file data in `files[]`.

Upload an app icon:

```bash
curl -X POST "https://apps.mzgs.net/api/apps/18/uploads" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "type=icon" \
  -F "files[]=@app-icon.png"
```

Upload multiple iPhone screenshots:

```bash
curl -X POST "https://apps.mzgs.net/api/apps/18/uploads" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "type=apple_iphone" \
  -F "files[]=@iphone-01.png" \
  -F "files[]=@iphone-02.png"
```

Upload an Android App Bundle:

```bash
curl -X POST "https://apps.mzgs.net/api/apps/18/uploads" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "type=android_bundle" \
  -F "files[]=@release.aab"
```

Supported upload types:

| Type | Updated field | Accepted file | Behavior | API limit |
| --- | --- | --- | --- | --- |
| `icon` | `icon` | One PNG/JPEG | Replace field | 20 MB |
| `apple_purchase_review` | `apple_purchase_review_img` | One PNG/JPEG | Replace field | 20 MB |
| `apple_iphone` | `apple_iphone_ss` | 1–10 PNG/JPEG images | Append | 20 MB each |
| `apple_ipad` | `apple_ipad_ss` | 1–10 PNG/JPEG images | Append | 20 MB each |
| `android_feature_graphic` | `android_app_feature_graphic` | One PNG/JPEG | Replace field | 20 MB |
| `android_phone` | `android_app_phone_screenshots` | 1–10 PNG/JPEG images | Append | 20 MB each |
| `android_tablet` | `android_app_tablet_screenshots` | 1–10 PNG/JPEG images | Append | 20 MB each |
| `android_bundle` | `android_app_bundle` | One valid AAB archive | Replace field | 500 MB |
| `android_localized_screenshots` | `android_screenshots` | One valid ZIP archive | Replace field | 250 MB |

Replacement uploads update the database field but do not delete the previously
stored file. Screenshot uploads append filenames to the existing comma-separated
list. Server `upload_max_filesize` and `post_max_size` settings can impose lower
limits than the API limits above.

Successful uploads use status `201` and return the selected type, updated field,
stored file details, and the complete updated app:

```json
{
  "type": "icon",
  "field": "icon",
  "uploaded": [
    {
      "filename": "app-18-icon-app-icon.png",
      "url": "/app/public/uploads/app-18-icon-app-icon.png",
      "size": 248120,
      "mime": "image/png"
    }
  ],
  "data": {
    "id": 18,
    "icon": "app-18-icon-app-icon.png"
  }
}
```

### Create/update JSON request rules

- The `POST /api/apps` and `PUT /api/apps/{id}` body must be a JSON object, not
  a JSON array or scalar.
- Unknown or read-only fields cause a `422` response; they are not ignored.
- Except for the three JSON fields, writable values must be scalar or `null`.
- String fields should be sent as JSON strings or `null`.
- `platform` accepts only `"apple"` or `"android"` (case-insensitive input).
- `account_id` and `app_order` accept JSON integers, integer strings, or `null`.
- `show_in_dashboard` and `is_active` accept JSON booleans, `0`, `1`, common
  boolean strings such as `"true"`/`"false"`, or `null`.
- `data`, `ios_locales`, and `android_locales` accept a JSON object, a JSON
  array, a string containing a valid JSON object/array, or `null`.
- File and image fields accept an existing stored filename/path as a string.
  These JSON endpoints do not upload files or accept multipart bodies.

### All writable request fields

No fields other than those in the tables below are accepted by `POST` or
`PUT`.

#### Common fields

| Field | Accepted value | Notes |
| --- | --- | --- |
| `name` | string or `null` | App display name; database limit 255 characters. |
| `platform` | `"apple"` or `"android"` | Defaults to `"apple"` on insert. |
| `package_name` | string or `null` | Bundle ID/package name; database limit 255 characters. |
| `icon` | string or `null` | Existing icon filename/path; database limit 255 characters. |
| `account_id` | integer, integer string, or `null` | Related App Store account ID. |
| `apple_app_id` | string or `null` | Apple numeric app identifier stored as text; database limit 255 characters. |
| `admob_app_id` | string or `null` | AdMob app identifier; database limit 255 characters. |
| `app_order` | integer, integer string, or `null` | Display order; defaults to `0` on insert. |
| `show_in_dashboard` | boolean, `0`, `1`, boolean string, or `null` | Whether the app appears on the dashboard. |
| `is_active` | boolean, `0`, `1`, boolean string, or `null` | Active state; defaults to `1` on insert. |
| `campaign_prefix` | string or `null` | Ads campaign prefix; database limit 255 characters. |
| `subscription_prefix` | string or `null` | In-app subscription prefix; database limit 255 characters. |
| `notes` | string or `null` | Free-form notes. |
| `data` | JSON object, JSON array, valid JSON string, or `null` | General app data, including store status. |

#### Apple metadata and Apple Ads fields

| Field | Accepted value | Notes |
| --- | --- | --- |
| `asa_keywords` | string or `null` | Apple Search Ads keywords, commonly newline-separated. |
| `asa_countries` | string or `null` | Apple Search Ads countries, commonly newline-separated country codes. |
| `custom_asa_name` | string or `null` | Custom Apple Search Ads name; database limit 255 characters. |
| `ios_locales` | JSON object, JSON array, valid JSON string, or `null` | Localized iOS metadata. |
| `apple_app_title` | string or `null` | App Store title; the Apps form recommends at most 30 characters. |
| `apple_app_subtitle` | string or `null` | App Store subtitle; the Apps form recommends at most 30 characters. |
| `apple_app_description` | string or `null` | App Store description; the Apps form recommends at most 4,000 characters. |
| `apple_app_keywords` | string or `null` | Comma-separated App Store keywords; the Apps form recommends at most 100 characters. |
| `apple_purchase_review_img` | string or `null` | Existing purchase-review image filename/path. |
| `apple_iphone_ss` | string or `null` | Existing iPhone screenshot filenames, stored as a comma-separated string. |
| `apple_ipad_ss` | string or `null` | Existing iPad screenshot filenames, stored as a comma-separated string. |

#### Google metadata and Google Ads fields

| Field | Accepted value | Notes |
| --- | --- | --- |
| `google_ads_campaigns_to_create` | string or `null` | Campaign configuration, commonly one `COUNTRY BID` pair per line. |
| `google_ads_campaign_titles` | string or `null` | Campaign titles, commonly newline-separated. |
| `google_ads_campaign_descriptions` | string or `null` | Campaign descriptions, commonly newline-separated. |
| `android_locales` | JSON object, JSON array, valid JSON string, or `null` | Localized Google Play metadata. |
| `android_app_title` | string or `null` | Google Play title; database limit 255 characters. |
| `android_app_short_description` | string or `null` | Google Play short description; database limit 255 characters. |
| `android_app_full_description` | string or `null` | Google Play full description. |
| `android_app_googleplayconsole_url` | string or `null` | Google Play Console URL; database limit 255 characters. |
| `android_app_privacy_url` | string or `null` | Privacy-policy URL; database limit 255 characters. |
| `android_app_feature_graphic` | string or `null` | Existing feature-graphic filename/path; database limit 255 characters. |
| `android_app_phone_screenshots` | string or `null` | Existing phone screenshot filenames, stored as a comma-separated string. |
| `android_app_tablet_screenshots` | string or `null` | Existing tablet screenshot filenames, stored as a comma-separated string. |
| `android_app_bundle` | string or `null` | Existing Android App Bundle filename/path; database limit 255 characters. |
| `android_screenshots` | string or `null` | Existing localized screenshots ZIP filename/path; database limit 255 characters. |

### Read-only response fields

The following fields can appear in GET and write responses but cannot be sent
in a request body:

| Field | Meaning |
| --- | --- |
| `id` | Auto-generated app ID. |
| `is_backup` | Internal backup marker. Apps API results exclude backup records. |
| `backup_date` | Internal backup creation date. |

### Complete request example

This example shows every writable key. Use only the fields needed for the
request.

```json
{
  "name": "Example App",
  "platform": "apple",
  "package_name": "com.example.app",
  "icon": "com.example.app.png",
  "account_id": 1,
  "apple_app_id": "1234567890",
  "admob_app_id": "ca-app-pub-0000000000000000~0000000000",
  "app_order": 10,
  "show_in_dashboard": true,
  "is_active": true,
  "campaign_prefix": "EXAMPLE",
  "subscription_prefix": "example",
  "google_ads_campaigns_to_create": "US 6\nGB 2",
  "google_ads_campaign_titles": "Example title one\nExample title two",
  "google_ads_campaign_descriptions": "Example description one\nExample description two",
  "asa_keywords": "example app\nexample tool",
  "asa_countries": "US\nGB",
  "custom_asa_name": "Example ASA",
  "ios_locales": {
    "en-US": {
      "title": "Example App",
      "subtitle": "Example subtitle"
    }
  },
  "apple_app_title": "Example App",
  "apple_app_subtitle": "Example subtitle",
  "apple_app_description": "Long App Store description",
  "apple_app_keywords": "example,tool,utility",
  "apple_purchase_review_img": "purchase-review.png",
  "apple_iphone_ss": "iphone-01.png,iphone-02.png",
  "apple_ipad_ss": "ipad-01.png,ipad-02.png",
  "android_app_title": "Example App",
  "android_app_short_description": "Example short description",
  "android_app_full_description": "Long Google Play description",
  "android_app_feature_graphic": "feature-graphic.png",
  "android_app_phone_screenshots": "phone-01.png,phone-02.png",
  "android_app_tablet_screenshots": "tablet-01.png,tablet-02.png",
  "android_app_googleplayconsole_url": "https://play.google.com/console/",
  "android_app_privacy_url": "https://example.com/privacy",
  "data": {
    "appstore_status": {
      "state": "READY_FOR_SALE",
      "version": "1.0"
    }
  },
  "android_app_bundle": "example.aab",
  "android_locales": {
    "en-US": {
      "title": "Example App",
      "short_description": "Example short description"
    }
  },
  "notes": "Internal app notes",
  "android_screenshots": "localized-screenshots.zip"
}
```

### Apps API errors

| Status | Meaning |
| --- | --- |
| `400` | The request body or uploaded file is malformed. |
| `401` | The API key is missing or incorrect. |
| `404` | The requested app does not exist or is a backup record. |
| `413` | An uploaded file exceeds its API or server size limit. |
| `422` | The ID, field name, field type, enum, JSON value, or update body is invalid. |
| `500` | The database insert or update failed. |
| `503` | No API access key is configured on the server. |

Error response example:

```json
{
  "error": "Request contains fields that cannot be written",
  "fields": ["id"]
}
```

## Sensor Tower Keyword Search API

### Endpoint

`GET /api/sensortower/keyword-search`

This endpoint uses query-string parameters and does not accept a request body.

| Query parameter | Required | Accepted value | Default |
| --- | --- | --- | --- |
| `term` | Yes | Non-empty search term | None |
| `country` | No | Two-letter country code, such as `US` or `TR` | `US` |
| `platform` | No | `ios` or `android` | `ios` |
| `page` | No | Positive integer | `1` |

```bash
curl -G "https://apps.mzgs.net/api/sensortower/keyword-search" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  --data-urlencode "term=photo editor" \
  --data-urlencode "country=US" \
  --data-urlencode "platform=ios" \
  --data-urlencode "page=1"
```

A successful request returns the raw Sensor Tower JSON response. Errors use
the shared `{"error": "message"}` format. Invalid parameters return `422`, an
upstream Sensor Tower failure returns `502`, and missing server configuration
returns `503`.
