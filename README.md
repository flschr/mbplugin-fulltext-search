# Advanced Micro.blog search page

Fork of Manton Reece's <a href="microdotblog/plugin-search-page">original Micro.blog search plugin</a> — thanks to Manton for sharing the foundation.

This version differs from the upstream release by:
- Localising the entire search page for German, including a shortcode intro that surfaces the total years and posts on the site.
- Replacing the original static results list with a live search interface that highlights keywords, shows contextual snippets, and lists tag/category metadata alongside keyboard-friendly controls.
- Sanitising the archive JSON feed before indexing, stripping `<script>`, `<style>`, `<noscript>`, and other markup noise, and augmenting it with taxonomy fields for richer filtering.
- **Instant search with localStorage caching** - Search field is immediately available, no waiting for archive to load.
- **Two-stage progressive loading** - Loads lightweight title index first for instant search, then enriches with full-text content in background.
- **Smart cache invalidation** - Automatically detects new content and refreshes cache when site is updated.

The fork is built for my private Micro.blog setup and can be tested at https://fischr.org/search.

## Features

### Instant Search Ready
- Search input is available immediately on page load
- No "Loading..." messages blocking the user
- Progressive enhancement: titles first, then full-text

### Intelligent Caching
- Uses localStorage to cache archive data
- Automatic cache invalidation based on `.Site.LastChange` timestamp
- Significantly faster subsequent page loads
- Old cache versions are automatically cleaned up

### Two-Stage Loading (Optional)
For optimal performance, you can configure a lightweight title-only archive:

1. Add to your site's `config.toml`:
```toml
[outputs]
  home = ["HTML", "JSON", "archivetitles"]

[outputFormats.archivetitles]
  name = "archivetitles"
  baseName = "index-titles"
  mediaType = "application/json"
  path = "archive"
  isPlainText = true
```

2. The search will then:
   - Load `/archive/index-titles.json` first (fast, ~10-20% of full size)
   - Enable title/tag/category search immediately
   - Load `/archive/index.json` in background for full-text search

If you don't configure the title-only archive, the plugin automatically falls back to loading the full archive directly (still with caching).

## How It Works

1. **On first visit**: Loads archive data from server, caches in localStorage
2. **On subsequent visits**: Loads from cache instantly (if version matches)
3. **When site updates**: Detects new `cache_version`, invalidates old cache, fetches fresh data
4. **Progressive enhancement**: If title-only archive exists, loads it first for instant search, then enriches with full content

## Cache Management

- **Storage**: Uses localStorage with keys like `archive_cache_<version>_titles` and `archive_cache_<version>_full`
- **Version tracking**: Based on Hugo's `.Site.LastChange.Unix` timestamp
- **Automatic cleanup**: Old cache versions are removed when new version is detected
- **Fallback**: If localStorage is unavailable or full, search still works (just without caching)
