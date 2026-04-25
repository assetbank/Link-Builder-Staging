# Link Builder — Technical Documentation

**Audience:** Technical support team  
**Last updated:** 2026-04-23  
**Tool URL:** https://jaxxon.github.io/Link-Builder/ *(update with actual URL)*

---

## Overview

The Link Builder is a browser-based tool that generates Bynder DAM search URLs. Users enter a portal domain, metaproperties, tags, or keywords, optionally select status filters, and the tool assembles a ready-to-use deep-link URL. No data is sent to any server — all processing happens in the browser.

---

## File Structure

```
Link-Builder/                    ← Production repository
├── index.html                   ← Main application page
├── style.css                    ← All styles
├── script.js                    ← All JavaScript logic
├── bynder-logo-white.png        ← Header logo
└── terms/
    └── index.html               ← Terms of Use & Privacy Policy page

Link-Builder-Staging-NEW/        ← Staging repository (mirrors production)
├── index.html
├── style.css
├── script.js
├── bynder-logo-white.png
└── terms/
    └── index.html
```

**Key difference between staging and production:**

| | Staging | Production |
|---|---|---|
| CSS path | `style.css?v=2` (relative) | `/style.css?v=2` (root-relative) |
| Google Analytics | Not present | Included (`G-Q0S8FCFFCK`) |

The `?v=2` query string is a cache-busting parameter — increment it after any CSS change to force browsers to fetch the updated file.

---

## Architecture

Plain HTML/CSS/JavaScript — no framework, no build step, no backend. Hosted on GitHub Pages. The tool is entirely client-side.

---

## Component Breakdown

### Header

Fixed navigation bar at the top of every page. Auto-hides when scrolling down and reappears when scrolling up, implemented via a `scroll` event listener that adds/removes the `.hidden` CSS class.

```
Logo (links to support.bynder.com) | Link Builder | Terms & Privacy
```

### Tab Bar

Two tabs switch between the two main workflows:

- **Metaproperties & Tags** (Panel 1) — builds filter URLs using DAM metaproperties, option values, and tags
- **Keywords** (Panel 2) — builds search URLs using free-text keyword search

Tab state is managed in JavaScript. Clicking a tab removes `.active` from the current panel and tab button, then adds `.active` to the new ones. A CSS `fadeIn` animation plays on the newly visible panel.

### Portal URL Input (`#baseURL`)

Shared between both tabs. The user enters their Bynder portal domain here (e.g. `dam.bynder.com`).

**Sanitization** (runs on every keystroke):
1. Strips leading `https://` or `http://`
2. Strips everything from the first `/` onwards
3. Strips query strings or fragments (`?`, `#`)

**Validation** (runs before generating any link):
- Must match: `^[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
- Must contain at least one dot and a valid TLD
- If invalid, a red error message appears below the input and no link is generated

---

## Panel 1: Metaproperties & Tags

### Metaproperty Rows (`#metaproperty-rows`)

Each row has two inputs:
- **Metaproperty** — the database name of the metaproperty (e.g. `Brand`)
- **Option** — the database name of the option value (e.g. `Bynder`)

Metaproperty and option values are sanitized: any character that is not a letter, number, or hyphen is replaced with an underscore. This matches Bynder's database naming rules.

The generated parameter format is:
```
field=metaproperty_<MetapropertyName>&value=<OptionValue>
```

Multiple metaproperty rows are joined together with `&`.

The `+` button below this section appends a new blank row dynamically.

### Tag Rows (`#tag-rows`)

Each row contains two tag inputs. Tags are not sanitized (capitalization and special characters are preserved). The parameter format is:
```
field=tags&value=<TagValue>
```

The `+` button appends a new row with two tag inputs.

### Status Action Buttons (`#status-grid-1`)

Eight preset filter buttons. Clicking a button toggles its `.selected` state; the URL regenerates immediately. Multiple buttons can be selected simultaneously.

| Button | Parameter appended |
|---|---|
| Assets Added Yesterday | `field=dateCreated&value=lastday` |
| Assets Added Last Week | `field=dateCreated&value=lastweek` |
| Watermarked Assets | `field=watermark&value=1` |
| Archived Assets | `field=archive&value=1` |
| Active Assets | `field=isActive&value=true` |
| Public Assets | `field=isPublic&value=1` |
| Limited Usage Assets | `field=keyVisual&value=1` |
| Sent for Review Assets | `field=audit&value=1` |

### URL Format — Panel 1

**With metaproperties** (uses `/search/set/` endpoint):
```
https://<domain>/search/set/?resetsearch&field=metaproperty_<Name>&value=<Option>&...&filterType=add
```

**Tags or status only** (uses `/search/media/` endpoint):
```
https://<domain>/search/media/?resetsearch&field=tags&value=<Tag>&...&filterType=add
```

The `resetsearch` parameter clears any existing active filters in the portal before applying the new ones. `filterType=add` applies the filters in addition mode.

---

## Panel 2: Keywords

### Keyword Rows (`#keyword-rows`)

Each row has a single text input. Keywords are URL-encoded (spaces become `%20`, etc.) but otherwise passed through as-is. Duplicate keywords are deduplicated using a JavaScript `Set`.

The parameter format is:
```
field=text&value=<keyword>
```

The `+` button appends a new keyword row.

### Status Action Buttons (`#status-grid-2`)

Identical set of eight buttons as Panel 1, operating independently.

### URL Format — Panel 2

Always uses `/search/set/`:
```
https://<domain>/search/set/?resetsearch&field=text&value=<keyword>&...&filterType=add
```

---

## Output Box (Click-to-Copy)

The generated link appears in a card at the bottom of each panel. Clicking anywhere on the card copies the URL to the clipboard using the `navigator.clipboard.writeText()` API.

States:
- **Empty state** — italic grey placeholder text, shown when no valid link can be generated
- **Link state** — blue clickable anchor tag showing the full URL
- **Copied state** — brief "Copied!" confirmation in green, reverts after 1.2 seconds

The output only shows a link when:
1. A valid domain is entered in the Portal URL field
2. At least one field (metaproperty+option pair, tag, keyword, or status button) has valid input

---

## URL Generation Logic (Detailed)

Both `generateLink()` and `generateKeywordLink()` follow the same pattern:

1. Read and validate `#baseURL` — abort with error if invalid or empty
2. Collect all non-empty field values into a `params` array
3. If no params were collected, show empty state
4. Assemble the final URL string and call `setLink()` to render it

The functions are called on every input event (keystroke) and on every status button click, so the output updates in real time.

---

## Dynamic Field Addition

All three `+` buttons work the same way:
1. Count existing rows to determine the next placeholder number
2. Create a new DOM element (`div.form-group.<type>-row`)
3. Inject input elements via `innerHTML`
4. Attach `input` event listeners to the new inputs
5. Append the row to the relevant container

New rows behave identically to the pre-rendered rows — they participate in URL generation immediately.

---

## Deployment Workflow

```
Edit files in Link-Builder-Staging-NEW/
        ↓
Preview with VS Code Live Server (right-click index.html → "Open with Live Server")
        ↓
Approve changes in staging
        ↓
Copy index.html, style.css, script.js to Link-Builder/ (production)
        ↓
Update CSS path: style.css?v=2 → /style.css?v=2
Increment version number if CSS changed (e.g. ?v=2 → ?v=3)
        ↓
Commit and push via GitHub Desktop
        ↓
GitHub Pages serves the updated production site
```

**Why two repositories?**  
Staging uses a relative CSS path (`style.css`) because it lives in a GitHub Pages sub-repo. Production uses a root-relative path (`/style.css`) required for the primary Pages domain. The two can't share the same file without modification.

**Why not open index.html directly in a browser?**  
Root-relative paths (starting with `/`) only resolve correctly when served by a web server. Opening `index.html` as a local file (`file://`) causes the CSS to 404. Use VS Code Live Server for local previewing.

---

## Google Analytics

GA4 property ID: `153293282`  
Measurement ID: `G-Q0S8FCFFCK`

The gtag.js snippet is included in the `<head>` of **production only** — both `index.html` and `terms/index.html`. It is intentionally absent from staging to avoid inflating analytics data with test traffic.

No personally identifiable information is collected. The tool never transmits any user-entered data (portal domains, metaproperty names, keywords) to Google Analytics.

---

## Known Constraints

- **No persistence** — all entered data is lost on page refresh. By design.
- **Capitalization matters** — metaproperty and option database names are case-sensitive in Bynder. The tool preserves whatever the user types; incorrect casing will produce a URL that returns no results.
- **Database names vs display names** — Bynder metaproperties have a display name and a separate database name. Users must enter the database name. If a metaproperty or option was ever renamed, only the current database name will work. Reference: [Bynder docs — database name](https://support.bynder.com/hc/en-us/articles/6190351879186-Create-Metaproperties#h_01KNM026Q3NVKNYR38QH1W90JJ)
- **Clipboard API** — `navigator.clipboard` requires a secure context (HTTPS or localhost). The tool works correctly on the production GitHub Pages URL and via Live Server. It will not work if opened as a plain `file://` URL.

---

## FAQ

**Q: The generated link opens the portal but shows no results.**  
A: Check that the metaproperty/option names exactly match the database names in Bynder (case-sensitive). Also confirm the portal domain is correct.

**Q: The URL field shows a red error.**  
A: The entered value doesn't match a valid domain format. Enter just the domain, e.g. `dam.bynder.com` — no `https://`, no trailing slashes.

**Q: The CSS looks wrong after a production deploy.**  
A: Increment the cache-busting version in the CSS `<link>` tag (e.g. `?v=2` → `?v=3`) in both `index.html` and `terms/index.html`, then redeploy.

**Q: Clicking the output box doesn't copy.**  
A: This requires a secure context. Ensure the page is loaded over HTTPS (production) or via Live Server (local). Opening the file directly won't work.
