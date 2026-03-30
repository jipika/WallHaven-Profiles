# WallHaven-Profiles

Community-maintained data source profiles for the [WallHaven](https://github.com/jipika/WallHaven) macOS app.

This repository holds JSON profile definitions that tell WallHaven how to fetch wallpapers and media from different sources. Instead of hardcoding site-specific logic, WallHaven loads profiles at runtime, so anyone can contribute a new source or adjust an existing one without modifying the app code.

---

## Overview

A **profile** describes two separate data sources:

| Section | Purpose |
|---------|---------|
| `wallpaper` | WallHaven API configuration (API key, image URL template, etc.) |
| `media` | HTML scraping configuration for a dynamic wallpaper site (routes, headers, XPath parsing rules) |

The app ships with a built-in default profile. This repository lets you:

- Override the default with a locally tuned configuration
- Add entirely new media sources (e.g., a different video wallpaper site)
- Share your configurations with the community

---

## Quick Start

### Option A: Use an existing community profile

1. Find a profile that matches your desired wallpaper source.
2. Open the WallHaven app, go to **Settings** > **Data Source Profile**.
3. Click **Import JSON** and paste the profile JSON (or load the file).
4. Select the newly imported profile and confirm.

### Option B: Create your own profile

See the **Creating a Custom Profile** section below.

---

## Profile Structure

Each profile is a JSON object with the following top-level fields:

```json
{
  "id": "unique-profile-id",
  "name": "Human-readable name",
  "description": "What this profile is for",
  "wallpaper": { ... },
  "media": { ... }
}
```

### `wallpaper` section

| Field | Required | Description |
|-------|----------|-------------|
| `provider` | Yes | Provider identifier (e.g., `"wallhaven_api"`) |
| `displayName` | Yes | Shown in the UI |
| `apiBaseURL` | Yes | Base URL for the API (e.g., `"https://wallhaven.cc/api/v1"`) |
| `searchPath` | Yes | Path segment for search requests (e.g., `"/search"`) |
| `wallpaperPath` | Yes | URL path template for individual wallpapers (e.g., `"/w/{id}"`) |
| `imageURLTemplate` | Yes | Full image URL pattern. Supports placeholders: `{prefix}` (first 2 chars of ID), `{id}`, `{ext}` |
| `authHeaderName` | No | HTTP header name for API key authentication (e.g., `"X-API-Key"`) |

### `media` section

| Field | Required | Description |
|-------|----------|-------------|
| `provider` | Yes | Provider identifier (e.g., `"motionbgs_html"`) |
| `displayName` | Yes | Shown in the UI |
| `baseURL` | Yes | Base URL for the HTML media site |
| `headers` | No | Extra HTTP headers sent with every request (e.g., User-Agent) |
| `routes` | Yes | URL path templates for each page type |
| `parsing` | Yes | XPath rules for extracting data from HTML |

#### `routes`

| Field | Template Placeholder | Example |
|-------|---------------------|---------|
| `home` | - | `"/"` |
| `mobile` | - | `"/mobile/"` |
| `tag` | `{slug}` | `"/tag:{slug}/"` |
| `search` | `{query}` | `"/search?q={query}"` |
| `detail` | `{slug}` | `"/{slug}"` |

#### `parsing` (XPath-based)

> **Note:** WallHaven uses XPath (not regex) to parse HTML responses. XPath is more readable and less error-prone for structured HTML.

| Field | Required | Description |
|-------|----------|-------------|
| `searchList` | Yes | XPath selecting all result item containers in the search results page |
| `searchName` | Yes | XPath (relative to each list item) extracting the item name/title |
| `searchResult` | Yes | XPath (relative to each list item) extracting the detail page link |
| `searchCover` | No | XPath (relative to each list item) extracting the cover image URL |
| `nextPage` | No | XPath for the "next page" link |
| `detailList` | No | XPath for download/link items on the detail page |
| `detailName` | No | XPath for the name on the detail page |
| `detailLink` | No | XPath for the download link on the detail page |
| `tagList` | No | XPath for tag links on a tag page |
| `tagName` | No | XPath for tag names |

**XPath tips:**

- `//div[@class='item']` - Selects any `div` with `class="item"` anywhere in the document
- `.//img/@src` - Relative to the current node, selects the `src` attribute of any `img`
- `@href` - Selects the `href` attribute of the current node
- `//a[contains(text(), 'Next')]/@href` - Selects `href` of an anchor containing the text "Next"
- XPath expressions can be combined with `|` (union): `//a/@href | //button/@hx-get`

---

## Creating a Custom Profile

### Step 1: Inspect the target website

1. Open your target wallpaper/media site in a browser.
2. Right-click on a search result card and choose **Inspect Element**.
3. Note the HTML structure (tag names, classes, attributes).

### Step 2: Write the XPath rules

Based on the HTML structure you observed, write XPath expressions:

```json
"parsing": {
  "searchList": "//div[@class='results']/article",
  "searchName": ".//h3/text()",
  "searchResult": ".//a/@href",
  "searchCover": ".//img/@src",
  "nextPage": "//a[contains(@class, 'next')]/@href"
}
```

### Step 3: Test locally

1. Copy `DataSourceProfile.sample.json` from this repository.
2. Fill in all required fields for your target site.
3. In WallHaven, go to **Settings** > **Data Source Profile** > **Import JSON**.
4. Select your profile and verify that search results appear.

### Step 4: Share with the community

Submit a pull request adding your profile to `DataSourceProfile.sample.json`.

---

## Profile Validation

WallHaven validates profiles before importing:

- `id` and `name` must be non-empty strings
- `wallpaper.apiBaseURL` and `media.baseURL` must be valid URLs
- Required XPath fields (`searchList`, `searchName`, `searchResult`) must be non-empty
- XPath expressions must start with `//`, `./`, `@`, `(`, or be exactly `.`

If validation fails, WallHaven will show a descriptive error and refuse to import.

---

## Catalog Format

Instead of submitting one profile at a time, you can submit a **catalog** that wraps multiple profiles:

```json
{
  "schemaVersion": "1.0.0",
  "profiles": [
    { "...profile 1..." },
    { "...profile 2..." }
  ]
}
```

The app will import all profiles in the catalog at once.

---

## Troubleshooting

**Import fails with "Invalid XPath pattern"**

Make sure all XPath expressions start with `//`, `./`, `@`, `(`, or are exactly `.`:

```json
"searchList": "//div[@class='item']"  // valid
"searchName": ".//h3/text()"           // valid
"searchResult": "@href"                 // valid
"searchCover": "."                     // valid
```

**Search results are empty but the site loads fine**

The site's HTML structure may have changed. Update the XPath expressions to match the current structure, then re-import the profile.

**API returns 403 or 429**

- For WallHaven: ensure you have a valid API key and it is set in the app settings.
- For media sites: try adding a `User-Agent` header in the `headers` field.

---

## Contributing

Contributions are welcome. Please:

1. Test your profile locally before submitting.
2. Ensure all required fields are present.
3. Use clear, descriptive `id`, `name`, and `description` values.
4. Keep XPath expressions as simple and readable as possible.

---

## License

This repository is MIT licensed. See [LICENSE](LICENSE) for details.
