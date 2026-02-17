## Backend API Specifications

### Technology

- **Language**: Go
- **Runtime**: Google Cloud Run
- **Network**: Behind Google Cloud Load Balancer + Cloud Armor

### Global API Rules

#### Allowed HTTP Methods

- THE SYSTEM SHALL only accept `GET` requests on all endpoints.
- WHEN a request uses any method other than `GET`, THE SYSTEM SHALL return HTTP `405 Method Not Allowed` with an `Allow: GET` header.

#### Error Response Strategy

- THE SYSTEM SHALL only return the following HTTP status codes to clients:
  - `200` — Success
  - `400` — Bad Request (invalid input)
  - `404` — Not Found (resource does not exist, or catch-all for masked errors)
  - `405` — Method Not Allowed
  - `429` — Too Many Requests (rate limit exceeded)
  - `503` — Service Unavailable (only for legitimate network/infrastructure failures)
- **THE SYSTEM SHALL NEVER return HTTP `500` to clients.**
  - IF an internal server error occurs, THE SYSTEM SHALL return `404` to the client.
  - THE SYSTEM SHALL log the actual error with full context to Google Cloud Logging.

#### Error Logging (Internal)

- WHEN an internal error occurs, THE SYSTEM SHALL log:
  - Timestamp (UTC)
  - Request method and path
  - Query parameters (sanitized)
  - Client IP address
  - Error message and stack trace
  - Request ID / correlation ID

#### Response Format

- All API responses SHALL use `Content-Type: application/json`.
- Article content responses SHALL include rendered content within the JSON payload.

---

### Endpoints

#### BE-API-001: `GET /`

**Description**: Returns the front page content.

**Request**: No parameters.

**Response** (`200`):

```json
{
  "content": "<markdown string of front page>"
}
```

**Behavior**:

- THE SYSTEM SHALL fetch the front page markdown content from the database.
- IF the content is not found, THE SYSTEM SHALL return `404`.

---

#### BE-API-002: `GET /technical`

**Description**: Returns a paginated list of technical blog articles.

**Query Parameters**:

| Parameter    | Type    | Required | Description                                    |
| ------------ | ------- | -------- | ---------------------------------------------- |
| `page`       | integer | No       | Page number (default: 1)                       |
| `q`          | string  | No       | Search query (max 300 characters)              |
| `category`   | string  | No       | Filter by category                             |
| `tags`       | string  | No       | Comma-separated list of tags to filter by      |
| `date_from`  | string  | No       | Start of date range (ISO 8601 date, e.g. `2025-01-01`) |
| `date_to`    | string  | No       | End of date range (ISO 8601 date)              |

**Validation**:

- IF `q` exceeds 300 characters, THE SYSTEM SHALL return `400`.
- IF `page` is not a positive integer, THE SYSTEM SHALL return `400`.
- IF `date_from` or `date_to` is not valid ISO 8601, THE SYSTEM SHALL return `400`.

**Response** (`200`):

```json
{
  "items": [
    {
      "title": "Article Title",
      "slug": "article-title-2025-01-15T10-30-00",
      "category": "DevOps",
      "abstract": "Brief summary of the article...",
      "tags": ["go", "cloud-run", "docker"],
      "date_created": "2025-01-15T10:30:00Z",
      "date_updated": "2025-06-20T14:00:00Z",
      "citation": "Monserrat, TJ. \"Article Title.\" tjmonserrat.com, 15 Jan 2025."
    }
  ],
  "pagination": {
    "current_page": 1,
    "total_pages": 5,
    "total_items": 42,
    "page_size": 10
  }
}
```

**Behavior**:

- THE SYSTEM SHALL return exactly 10 items per page (fixed, not configurable by the client).
- IF the search/filter yields no results, THE SYSTEM SHALL return `404`.

---

#### BE-API-003: `GET /technical/{slug}.md`

**Description**: Returns a single technical blog article.

**Path Parameters**:

| Parameter | Type   | Description                                    |
| --------- | ------ | ---------------------------------------------- |
| `slug`    | string | Article slug in format: `{title-slug}-{ISO8601-datetime}` |

**Validation**:

- IF the URL does not end with `.md`, THE SYSTEM SHALL return `404`.
- IF the slug does not match the prescribed pattern, THE SYSTEM SHALL return `404`.
  - Slug pattern: `^[a-z0-9]+(?:-[a-z0-9]+)*-\d{4}-\d{2}-\d{2}T\d{2}-\d{2}-\d{2}\.md$`
  - *See Clarification CLR-005 for exact pattern confirmation.*
- IF the article does not exist in the database, THE SYSTEM SHALL return `404`.

**Response** (`200`):

```json
{
  "title": "Article Title",
  "author": "TJ Monserrat",
  "date_created": "2025-01-15T10:30:00Z",
  "date_updated": "2025-06-20T14:00:00Z",
  "category": "DevOps",
  "tags": ["go", "cloud-run", "docker"],
  "changelog": [
    {
      "date": "2025-06-20T14:00:00Z",
      "description": "Updated code examples for Go 1.23"
    },
    {
      "date": "2025-01-15T10:30:00Z",
      "description": "Initial publication"
    }
  ],
  "content": "<markdown string of article body>",
  "pagination": {
    "current_page": 1,
    "total_pages": 3
  }
}
```

> **Note**: See Clarification CLR-003 regarding article pagination (multi-page articles).

---

#### BE-API-004: `GET /blog`

**Description**: Returns a paginated list of opinion/blog articles.

**Behavior**: Identical to BE-API-002 but queries the blog/opinions table instead of the technical table.

All query parameters, validation rules, response format, and error handling are the same as BE-API-002.

---

#### BE-API-005: `GET /blog/{slug}.md`

**Description**: Returns a single opinion/blog article.

**Behavior**: Identical to BE-API-003 but queries the blog/opinions table instead of the technical table.

All path parameters, validation rules, response format, and error handling are the same as BE-API-003.

---

#### BE-API-006: `GET /socials`

**Description**: Returns a list of social media links.

**Request**: No parameters.

**Response** (`200`):

```json
{
  "items": [
    {
      "platform": "GitHub",
      "url": "https://github.com/tjmonserrat",
      "display_name": "@tjmonserrat"
    },
    {
      "platform": "LinkedIn",
      "url": "https://linkedin.com/in/tjmonserrat",
      "display_name": "TJ Monserrat"
    }
  ]
}
```

---

#### BE-API-007: `GET /others`

**Description**: Returns a paginated list of external/notable content.

**Behavior**: Identical to BE-API-002 in terms of query parameters, pagination, validation, and error handling, but queries the "others" table.

**Response item differences**:

```json
{
  "items": [
    {
      "title": "External Article Title",
      "slug": "external-article-2025-03-10T08-00-00",
      "category": "Open Source",
      "abstract": "Description of the external content...",
      "tags": ["python", "colab"],
      "date_created": "2025-03-10T08:00:00Z",
      "date_updated": "2025-03-10T08:00:00Z",
      "external_url": "https://github.com/tjmonserrat/project"
    }
  ],
  "pagination": {
    "current_page": 1,
    "total_pages": 2,
    "total_items": 15,
    "page_size": 10
  }
}
```

> **Note**: Unlike technical/blog articles, items in `/others` link to external URLs rather than internal article pages.

---

#### BE-API-008: Tracking Endpoint

**Description**: Receives anonymous visitor tracking data.

> **Note**: See Clarification CLR-002. The initial plan specifies GET-only, but tracking requires sending data. Endpoint design depends on resolution of CLR-002.

**Proposed**: `GET /track` with query parameters (if GET-only is enforced):

| Parameter  | Type   | Description                     |
| ---------- | ------ | ------------------------------- |
| `page`     | string | Current page URL                |
| `ref`      | string | Referrer URL (optional)         |
| `action`   | string | User action identifier          |

- IP address and User-Agent are extracted from request headers server-side.

---

#### BE-API-009: Error Reporting Endpoint

**Description**: Receives frontend error/performance data.

> **Note**: See Clarification CLR-002. Same constraint as tracking.

**Proposed**: `GET /report-error` with query parameters (if GET-only is enforced):

| Parameter   | Type   | Description                        |
| ----------- | ------ | ---------------------------------- |
| `type`      | string | Error type (network, timeout, etc.)|
| `message`   | string | Error message (truncated)          |
| `page`      | string | Page URL where error occurred      |
| `speed`     | string | Connection speed (if detected)     |

---

### Rate Limiting

See [06-security-specifications.md](06-security-specifications.md) for full rate limiting specifications.
