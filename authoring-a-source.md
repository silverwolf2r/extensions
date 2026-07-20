# Authoring a book source

This repo hosts `booksources.json` — the book-source config the Fic Reader app
installs from a URL. This guide explains **how to build a source from scratch**,
test it, and add it (either to this shared list or as your own private config).

A "source" is pure **data**, never code. It points one of the app's four
built-in parser *strategies* at some hosts. The app ships the strategies (generic
web-scraping logic that names no site); your config supplies the domains, paths,
and download wiring.

---

## 0. Before you start: does the site even qualify?

Two hard requirements. Fail either and **no config can fix it** — the limit is
in the site.

### Requirement 1 — fetchable with a plain HTTP GET (no JavaScript)

The app fetches with native HTTP (a normal browser User-Agent, from the device's
residential IP) and **runs no JavaScript**. The site must return usable HTML to a
plain GET.

**Fails** this: anything behind Cloudflare's "Just a moment…", Turnstile, or any
"checking your browser / enable JavaScript" managed challenge; and single-page
apps that render results client-side (you'd get an empty shell). A **JSON API**
the page calls is fine — point the config at that.

> The `web-cards` strategy solves one *self-contained* SHA-1 proof-of-work gate.
> It does **not** solve a Cloudflare managed challenge — nothing in the app does,
> because that needs a JS engine.

Ten-second check:

```bash
curl -s -A 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_5 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Mobile/15E148 Safari/604.1' \
  'https://SITE/SEARCH-PATH' | head -c 800
```

Real result markup → good. `Just a moment…`, `challenge-platform`, `Enable
JavaScript`, `Attention Required`, or an empty `<div id="root">` → it fails.
(A datacenter IP is challenged harder than a phone, but a *JS* challenge blocks
the app's no-JS fetch on a phone too — a residential IP adds reputation, not a JS
engine.)

### Requirement 2 — a real, downloadable EPUB

Every download is validated by **magic bytes**: an EPUB is a ZIP, so the file
must start with `PK`. A site that only offers an *online reader* (streams into a
JS flip-book, no file) has nothing to download. There must be a reachable actual
`.epub` (or a link/redirect ending at one).

---

## 1. Pick a strategy

| `parser` | Results page looks like… | Download model |
| --- | --- | --- |
| `html-table` | a classic HTML `<table>`, one `<tr>` per book, header row naming columns | `downloadMirrors`: mirror page → `GET` link → stream → validate |
| `flex-rows` | not a table; each result a block keyed by an `/md5/<32-hex>` link | shared-hash mirrors first, then the site's own `/md5` page |
| `web-cards` | custom card elements (attributes hold metadata) **and** a self-contained SHA-1 "checking your browser" gate | per-book page → `/dl/{token}` button → anonymous download |
| `web-search` | you search a **public search engine** for `site:HOST` and resolve hit pages | follow the hit page to a direct `.epub`; validate bytes |

- Site has its own GETtable HTML results page → `html-table` (table) or
  `flex-rows` (not a table).
- Site gates behind a self-contained JS proof-of-work and uses card elements →
  `web-cards`.
- Scrape-hostile for listing, or you'd rather find books via a search engine →
  `web-search` (also powers "paste a book link to import").

If the markup matches none of these, the site needs a **new strategy in the app**
(code + review), not just a config.

---

## 2. The fields (placeholders substituted at request time)

| Token | Becomes |
| --- | --- |
| `{query}` | search text, URL-encoded |
| `{page}` | 1-based page number |
| `{md5}` | book content hash, lower-cased (download templates) |
| `{hash}` | per-book hash from a `web-cards` result (`bookPath`) |

- **`searchBases`** — one or more `https://` origins, tried in order; list mirrors
  so a dead one is skipped.
- **`searchPaths`** — path templates. Run a real search on the site, copy the URL,
  replace the query with `{query}` and page number with `{page}`.
- **`downloadMirrors`** (`html-table` / `flex-rows`) — e.g.
  `https://host/ads.php?md5={md5}`; a bare host gets `/main/{md5}` appended.
- **`bookPath`** (`web-cards`) — per-book page template, e.g. `/book/{hash}`.
- **`hosts`** (`web-search`, required) — content hosts to keep + resolve.
  **`fileHosts`** (optional) — direct-file CDN hostnames.
- **`userAgent`** (optional) — override the default desktop UA.

---

## 3. Confirm the markup matches

```bash
curl -s -A 'Mozilla/5.0 …' 'https://SITE/your-search-path?with=a+real+query' > out.html
```

- `html-table`: a `<table>` whose header cells are labelled Author / Title /
  Publisher / Year / Language / Size / Ext (columns are mapped **by label**), and
  each data row carries an `md5=<32-hex>` link.
- `flex-rows`: each result carries a `/md5/<32-hex>` link, plus a `·`-separated
  meta line like `English [en] · EPUB · 9.8MB · 2022`.
- `web-cards`: results are custom elements with `extension`/`filesize`/
  `href="/book/<hash>"` attributes and slotted title/author; a first GET returns a
  503 "Checking your browser" page whose script is a self-contained SHA-1 loop.
- `web-search`: the search engine returns anchors to `site:HOST` pages, and a book
  page on that host has a reachable `.epub` link.

---

## 4. Write the JSON

```jsonc
// html-table
{ "id": "src", "name": "Source", "parser": "html-table",
  "searchBases": ["https://host.invalid"],
  "searchPaths": ["/index.php?req={query}&res=100&page={page}"],
  "downloadMirrors": ["https://host.invalid/ads.php?md5={md5}", "https://mirror.invalid"] }

// flex-rows
{ "id": "src", "name": "Source", "parser": "flex-rows",
  "searchBases": ["https://host.invalid"],
  "searchPaths": ["/search?q={query}&ext=epub&page={page}"],
  "downloadMirrors": ["https://mirror.invalid/ads.php?md5={md5}"] }

// web-cards
{ "id": "src", "name": "Source", "parser": "web-cards",
  "searchBases": ["https://host.invalid"],
  "searchPaths": ["/s/{query}?page={page}&extensions[]=EPUB"],
  "bookPath": "/book/{hash}" }

// web-search
{ "id": "src", "name": "Source", "parser": "web-search",
  "searchBases": ["https://search-engine.invalid"],
  "searchPaths": ["/html/?q={query}%20epub%20site%3Ahost.invalid"],
  "hosts": ["host.invalid"], "fileHosts": ["cdn.invalid"] }
```

Envelope:

```jsonc
{ "version": 1, "name": "My sources", "providers": [ /* one or more of the above */ ] }
```

The app rejects a file that isn't a JSON object, lists no providers, names an
unknown `parser`, has a provider with no `https?://` `searchBase` or no
`searchPath`, or is a `web-search` provider with no `hosts`. A bad file shows a
specific error at install and nothing is stored.

---

## 5. Add it

### To this shared source list (maintainer)

1. Edit `booksources.json` in this repo and append your provider object to the
   `providers` array (keep it valid JSON).
2. Commit.
3. The install URL is this file's **raw** URL:
   `https://raw.githubusercontent.com/<owner>/<repo>/main/booksources.json`.

> **Refresh semantics:** the app fetches + validates the config **once, at
> install**, then persists it on the device. Editing this file changes what *new*
> installs receive; a device that already installed the URL keeps its stored copy
> until the user **re-adds the same URL** (install overwrites). No background
> auto-refresh today.

### Your own private source (no repo needed)

1. Write your `providers` JSON (section 4).
2. Host it at any `https://` URL that returns the raw JSON to an anonymous GET —
   a GitHub **raw** file, a **gist** (Raw button), or any static host. (A private
   repo's raw URL needs a token and won't work — it must be public.)
3. In the app: **Browse → Books** → paste the `https://…json` link → **Add**. The
   app fetches, validates, and persists it; providers appear as chips; **Remove**
   clears it. Native app only.

### Bonus: paste-a-link import

Install a `web-search` provider and pasting a book/page URL on one of its `hosts`
into the **Import** sheet routes it through that provider's resolver — a single
link imports without searching.

---

## 6. Worked example — epub.pub (does **not** qualify)

The best teacher is a site that fails the requirements. Take
`https://www.epub.pub/`.

```bash
curl -s -A 'Mozilla/5.0 …' 'https://www.epub.pub/?s=hail%20mary' | head -c 300
# → <!DOCTYPE html><html><head><title>Just a moment...</title> … challenge-platform …
```

That's Cloudflare's managed JS challenge — **fails Requirement 1**. The app's
no-JS fetch can't clear it, so `html-table`, `flex-rows`, `web-cards`, and the
*download* step of `web-search` all receive the challenge page, not content.

`web-search` sidesteps it only for *search* (it queries a search engine, not
epub.pub). But the *download* step must fetch the epub.pub book page for its
`.epub` link — Cloudflare-gated → fails. It would need a direct `.epub` on a
non-Cloudflare host (epub.pub has none), and epub.pub is largely online-reader
content anyway (**Requirement 2**).

The config you'd write (valid *shape*, so it installs) is
[`examples/epub-pub.json`](./examples/epub-pub.json):

```json
{
  "version": 1,
  "name": "epub.pub (demo — will not download)",
  "providers": [
    { "id": "epubpub", "name": "epub.pub", "parser": "web-search",
      "searchBases": ["https://html.duckduckgo.com"],
      "searchPaths": ["/html/?q={query}%20epub%20site%3Aepub.pub"],
      "hosts": ["epub.pub"] }
  ]
}
```

**Verdict:** installs as a valid `web-search` config, but **won't retrieve
books** — Cloudflare's JS challenge blocks the fetch, and epub.pub serves an
online reader rather than downloadable EPUBs. A good `web-search` candidate
returns plain HTML to a GET *and* has book pages with a real `.epub` link (or a
direct `.epub` on a plain CDN you can list in `fileHosts`).
