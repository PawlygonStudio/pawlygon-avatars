# Pawlygon Avatars

Remote data source for the **Pawlygon Avatar Board** VRChat world prefab.

This repo is published as a static site via GitHub Pages. The Unity/UdonSharp prefab downloads `avatars.json` at runtime using `VRCStringDownloader`, and (Phase 3) fetches thumbnails via `VRCImageDownloader`. Because the site is served from `*.github.io` — which is on VRChat's trusted-domain allowlist — no in-game consent prompt is triggered.

**Live URL pattern:** `https://<user>.github.io/<repo>/avatars.json`

---

## Schema

`avatars.json` is intentionally **flat**. Nested arrays are painful to parse in Udon's `VRCJson`, so `tags` is a comma-joined string rather than a JSON array.

| Field                | Type   | Notes                                                                                         |
| -------------------- | ------ | --------------------------------------------------------------------------------------------- |
| `version`            | int    | Schema version. Bump on **breaking** changes only (field removed/renamed/retyped).            |
| `updated`            | string | ISO date (`YYYY-MM-DD`). Informational only — not consumed by the prefab.                     |
| `avatars[]`          | array  | The avatar list.                                                                              |
| `avatars[].id`       | string | VRChat blueprint ID in the form `avtr_<uuid>`. **Must be a public avatar** (see below).       |
| `avatars[].name`     | string | Display name shown on the board row.                                                          |
| `avatars[].author`   | string | Creator handle. Free-form.                                                                    |
| `avatars[].thumbnail`| string | Absolute URL to a PNG on a VRChat-trusted host. Same-origin GitHub Pages is recommended.      |
| `avatars[].tags`     | string | Comma-joined list, e.g. `"quest,pc,fallback"`. **Not** an array.                              |

### Hard rule: avatars must be public

The board swaps the player's avatar via a hidden `VRCAvatarPedestal`. Private avatars are only visible to their owner — other players in the world will see the swap fail. **Only add avatars whose sharing setting is Public.**

---

## Adding a new avatar

1. Edit `avatars.json` — add a new object to the `avatars` array.
2. Drop a thumbnail PNG into `thumbs/` (see image requirements below).
3. Update the new entry's `thumbnail` field to point at the PNG's Pages URL.
4. Optionally bump the `updated` field to today's date.
5. Commit to `main`. The `deploy-pages.yml` workflow publishes automatically.

### Thumbnail image requirements

- **Format:** PNG
- **Hard cap:** 2048×2048
- **Target:** ≤ 256×256 and ≤ 100 KB per file

`VRCImageDownloader` has a 32 MB decode buffer and a **global rate limit of 1 image every 5 seconds**, shared across all downloads in the instance. Small files matter more than pixel resolution — a board with many avatars will take a noticeable amount of time to populate even with tiny thumbnails.

---

## Cache-busting

GitHub Pages sits behind Fastly, which caches responses for roughly **10 minutes**. After you push a change to `avatars.json`, the live URL will continue to serve the old body until the cache expires.

To force an immediate refresh in-game, append a version query parameter to the URL baked into the prefab's `[SerializeField] VRCUrl`:

```
https://<user>.github.io/<repo>/avatars.json?v=2
```

Bump `v` whenever you change the JSON and can't wait out the cache. Note that `VRCUrl` can only be constructed at Editor time in Udon, so this requires re-editing and rebuilding the prefab — reserve it for when the 10-minute wait is unacceptable.

---

## Schema version policy

- **Bump `version`** on breaking changes: field renamed, field removed, field type changed, meaning of an existing field changed.
- **Do not bump** on additive changes: adding a new field that older prefab versions can ignore, adding new avatars, fixing typos.

The prefab reads `version` so it can refuse to parse a JSON file it doesn't understand.

---

## Why GitHub Pages (and not gists)

Gists can host a single JSON file cleanly, but can't host image assets alongside it in a way that plays well with `VRCImageDownloader`. A regular repo + Pages gives us one origin for both the JSON and the thumbnails, which keeps everything inside the same trusted-domain allowlist entry.

---

## Pages URL placeholders

The `thumbnail` URLs in `avatars.json` currently contain literal `<user>` and `<repo>` placeholder text. Once the repo owner and name are finalized, **rewrite them to the real Pages origin** (e.g. `https://pawlygon.github.io/pawlygon-avatars.github.io/thumbs/avatar1.png`).
