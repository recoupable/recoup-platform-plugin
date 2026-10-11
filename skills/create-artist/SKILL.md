---
name: create-artist
description: End-to-end playbook for creating, identifying, and enriching a new artist account. Use when the user asks to create, add, onboard, or set up a new artist — phrases like "create artist", "onboard X", "add this artist", "set up a new artist", or any task that starts a brand-new artist record from a name. The skill drives 8 sequential steps (create → Spotify match → PATCH profile → Recoup research → Spotify catalog → web socials search → PATCH socials → synthesize KB) from a `RECOUP.md` checklist scaffolded by the `artist-workspace` skill, ticking each box and persisting captured values back to the file as it goes.
---

# Create New Artist

The canonical recipe used internally by Recoup's chat agent. Follow it step-by-step to bring a brand-new artist account up to "researched + enriched" parity from a sandbox or any external agent.

The chain is **8 sequential steps**. Long deterministic chains executed from prose memory tend to drop steps — the agent reads the doc once, runs a couple of calls, and forgets the rest. Drive the work from the `RECOUP.md` checklist file scaffolded by the `artist-workspace` skill: tick each box and persist captured values back to the frontmatter as you go. The file IS the workflow state, and a fresh turn can resume by reading it.

## Prerequisites

- `$RECOUP_API_KEY` (x-api-key) or `$RECOUP_ACCESS_TOKEN` (Privy Bearer JWT) for `api.recoupable.dev`
- `$RECOUP_ORG_ID` — the org the artist should belong to (recommended in sandboxes)
- An artist name to create (e.g. `ARTIST_NAME="The Weeknd"`)
- The artist's `RECOUP.md` already scaffolded (see `artist-workspace` skill, Step 0)

Select exactly one REST credential header before the examples below:

```bash
if [ -n "$RECOUP_API_KEY" ]; then
  AUTH=(-H "x-api-key: $RECOUP_API_KEY")
elif [ -n "$RECOUP_ACCESS_TOKEN" ]; then
  AUTH=(-H "Authorization: Bearer $RECOUP_ACCESS_TOKEN")
else
  echo "Authenticate REST through the approved real-account setup flow." >&2
  exit 1
fi
```

The flow has three phases, all driven from the single checklist file:

1. **Create + identify** — `POST /api/artists`, then find the canonical Spotify match
2. **Enrich** — `PATCH` the artist with image/label/socials, then run Recoup research (available profile/activity/playlists) plus a web search for narrative context and additional socials
3. **Synthesize + persist** — generate a knowledge-base report, save it (RECOUP.md tree or hosted URL), then optionally `PATCH` the `knowledges` array

**Don't run this chain under a throwaway `agent+` account.** Artist data created against an `agent+...@recoupable.com` email is permanently isolated to that account and can't be recovered if the API key is lost. Only run after the user has authenticated with their real email.

## Resuming a partial setup

If `$ARTIST_DIR/RECOUP.md` already exists, do not re-scaffold and do not re-run completed steps. Read the file, find the first unchecked item, and resume from there using the values already saved in the frontmatter:

```bash
# Show the next unchecked step
grep -n '^- \[ \]' "$ARTIST_DIR/RECOUP.md" | head -1
```

If every item is checked, the artist is fully set up — confirm with the user before doing anything else.

## Step 1: Create the artist

```bash
ARTIST_RESPONSE=$(curl -sS -X POST "https://api.recoupable.dev/api/artists" \
  "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg name "$ARTIST_NAME" --arg org "$RECOUP_ORG_ID" \
        '{name: $name, organization_id: $org}')")

ARTIST_ID=$(echo "$ARTIST_RESPONSE" | jq -r '.artist.account_id')
```

Capture `account_id` as `$ARTIST_ID` — every subsequent step needs it. `organization_id` is optional but should be included when running inside a sandbox so the artist is scoped to the right org.

Full request/response schema: `https://docs.recoupable.dev/api-reference/artists/create`.

**After this step:** write `artistId: $ARTIST_ID` into the frontmatter and tick `- [ ] 1.` → `- [x] 1.` in `RECOUP.md`.

## Step 2: Find the canonical Spotify match

```bash
SPOTIFY=$(curl -sS -G "https://api.recoupable.dev/api/spotify/search" \
  "${AUTH[@]}" \
  --data-urlencode "q=$ARTIST_NAME" \
  --data-urlencode "type=artist" \
  --data-urlencode "limit=10")
```

Inspect the returned candidates' names, profile links, images and catalog.
Set `SELECTED_SPOTIFY_ARTIST_ID` to the verified match; popularity alone does not
establish identity. If candidates remain ambiguous, resolve that before saving
the artist's profile.

```bash
printf '%s' "$SPOTIFY" | jq '.artists.items[]? | {id, name, external_urls, images}'
```

If the verified artist is absent from the first ten results, repeat the search
with a more specific name or known catalog context and inspect the new results.
Do not substitute another artist. After verifying the candidate, export its fields:

```bash
: "${SELECTED_SPOTIFY_ARTIST_ID:?Set the verified Spotify artist ID first}"
MATCH=$(printf '%s' "$SPOTIFY" | jq --arg id "$SELECTED_SPOTIFY_ARTIST_ID" \
  '.artists.items[]? | select(.id == $id)')

SPOTIFY_ARTIST_ID=$(printf '%s' "$MATCH" | jq -r '.id // empty')
SPOTIFY_PROFILE_URL=$(printf '%s' "$MATCH" | jq -r '.external_urls.spotify // empty')
SPOTIFY_IMAGE_URL=$(printf '%s' "$MATCH" | jq -r '.images[0].url // empty')

[ -n "$SPOTIFY_ARTIST_ID" ] || { echo "No verified Spotify match for $ARTIST_NAME"; exit 1; }
```

Also keep `genres`, `followers.total`, and `popularity` from `$MATCH` for the KB report later.

Full schema: `https://docs.recoupable.dev/api-reference/spotify/search`.

**After this step:** write `spotifyArtistId`, `spotifyProfileUrl`, and `imageUrl` into the frontmatter, save `genres` / `followers.total` / `popularity` to the `## Notes` section, and tick `- [ ] 2.` → `- [x] 2.`.

## Step 3: Set basic profile + Spotify URL

One `PATCH` covers the image and the Spotify social URL. Use **uppercase** platform keys in `profileUrls` (the API matches platforms case-sensitively — see the platform key reference at the bottom of this file).

```bash
curl -sS -X PATCH "https://api.recoupable.dev/api/artists/$ARTIST_ID" \
  "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg image "$SPOTIFY_IMAGE_URL" --arg url "$SPOTIFY_PROFILE_URL" \
        '{image: $image, profileUrls: {SPOTIFY: $url}}')"
```

This single endpoint replaces what the chat tool chain runs as four separate MCP calls (`update_account_info` ×2, `update_artist_socials` ×2). Add `label` only after verifying it in the research (step 4); profile coverage may be partial. Full body schema: `https://docs.recoupable.dev/api-reference/artists/update`.

**After this step:** tick `- [ ] 3.` → `- [x] 3.`.

## Step 4: Run Recoup research

Resolve the verified Spotify artist to an opaque Recoup research ID, then read
available profile fields, activity and playlist placements. Coverage varies by
artist and platform: an empty successful response is a data gap, not a request to
warm a cache. Use a web search for biography, press and collaborations.

The four structured reads below cost $0.05 each when successful, or $0.20 total
(200,000 micro-dollar credits). Web research is billed separately; check the
current credit reference before running it. Check HTTP status before extracting
IDs or making dependent calls. This helper
allows at most three attempts for network failures or `5xx`, each with a
90-second timeout. A `402` surfaces `billingUrl`; a `429` stops the step and prints
`Retry-After` for a later manual retry; it does not wait or retry automatically. Leave the step unchecked on errors.

```bash
BASE="https://api.recoupable.dev/api"
research_request() {
  local path=$1 response body status attempt headers retry_after
  shift
  headers=$(mktemp) || return 1
  for attempt in 1 2 3; do
    if response=$(curl -sS --max-time 90 -D "$headers" -w $'\n%{http_code}' \
      "${AUTH[@]}" "$BASE/research/$path" "$@"); then
      status=${response##*$'\n'}
      body=${response%$'\n'*}
      case "$status" in
        200) rm -f "$headers"; printf '%s' "$body"; return 0 ;;
        402)
          rm -f "$headers"
          printf 'Research needs credits: %s\n' \
            "$(printf '%s' "$body" | jq -r '.billingUrl // .error // "Check billing"')" >&2
          return 1 ;;
        429)
          retry_after=$(awk 'tolower($1)=="retry-after:" {sub(/^[^:]*:[[:space:]]*/, ""); sub(/\r$/, ""); print}' "$headers")
          rm -f "$headers"
          printf 'Research step stopped. Retry-After: %s. Resume manually when allowed.\n' "${retry_after:-not provided}" >&2
          return 1 ;;
        5??) ;;
        *) rm -f "$headers"; printf 'Research HTTP %s; resolve before resuming.\n' "$status" >&2; return 1 ;;
      esac
    fi
    if [ "$attempt" -lt 3 ]; then sleep "$((attempt * 2))"; fi
  done
  rm -f "$headers"
  echo 'Research unavailable after three attempts.' >&2
  return 1
}
```

### 4a: Resolve the research artist ID

```bash
LOOKUP=$(research_request lookup --get \
  --data-urlencode "spotifyId=$SPOTIFY_ARTIST_ID") || exit 1

RESEARCH_ARTIST_ID=$(printf '%s' "$LOOKUP" | jq -r \
  '[.id, .artist_info.id] | map(select(type == "string") | select(test("\\S"))) | first // empty')
```

Save this value as `researchArtistId`. It is separate from the Recoup artist
account UUID and Spotify ID. Do not coerce it to a number or copy an older
saved research ID into it; re-resolve and verify older workspaces. If a successful
lookup has no usable `id`, record that gap and skip 4b–4d, then run 4e.

See [Artist Lookup](https://docs.recoupable.dev/api-reference/research/lookup) for the response contract.

### 4b–4d: Read available structured signals

```bash
if [ -n "$RESEARCH_ARTIST_ID" ]; then
  PROFILE=$(research_request profile --get \
    --data-urlencode "id=$RESEARCH_ARTIST_ID") || exit 1
  CAREER=$(research_request career --get \
    --data-urlencode "id=$RESEARCH_ARTIST_ID") || exit 1
  PLAYLISTS=$(research_request playlists --get \
    --data-urlencode "id=$RESEARCH_ARTIST_ID") || exit 1
fi
```

Inspect the fields actually returned. Profile data may be partial; `career` is
an available activity feed, not a guaranteed complete career timeline.
`playlists` returns `placements`, which may be empty. Do not turn missing data
into zero or infer a label, achievement or placement that is absent.

See [Artist Profile](https://docs.recoupable.dev/api-reference/research/profile),
[Artist Activity](https://docs.recoupable.dev/api-reference/research/career) and
[Playlist Placements](https://docs.recoupable.dev/api-reference/research/playlists).

### 4e: Search for narrative, press and collaborations

```bash
RESEARCH_WEB=$(research_request web -X POST \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg name "$ARTIST_NAME" \
        '{query: ($name + " biography press recent collaborations")}')") || exit 1
```

Verify source links before asserting a fact. See [Web Search](https://docs.recoupable.dev/api-reference/research/web).

**After this step:** persist `researchArtistId` when available. Save the returned
payloads, source links and coverage gaps under `## Research` in `RECOUP.md`, with
one subsection per request. Tick step 4 only after the successful reads and any
intentional structured-data skip have been recorded.

## Step 5: Pull the Spotify catalog

```bash
TOP_TRACKS=$(curl -sS -G "https://api.recoupable.dev/api/spotify/artist/topTracks" \
  "${AUTH[@]}" \
  --data-urlencode "id=$SPOTIFY_ARTIST_ID")

ALBUMS=$(curl -sS -G "https://api.recoupable.dev/api/spotify/artist/albums" \
  "${AUTH[@]}" \
  --data-urlencode "id=$SPOTIFY_ARTIST_ID")

# For each notable album, drill in (ALBUM_ID from $ALBUMS):
ALBUM_DETAIL=$(curl -sS -G "https://api.recoupable.dev/api/spotify/album" \
  "${AUTH[@]}" \
  --data-urlencode "id=$ALBUM_ID")
```

`$SPOTIFY_ARTIST_ID` is the value you exported in step 2.

Full schemas: top tracks (`/api-reference/spotify/artist-top-tracks`), albums (`/api-reference/spotify/artist-albums`), album detail (`/api-reference/spotify/album`).

**After this step:** populate the artist's `releases/` folder — write one `releases/{release-slug}/RELEASE.md` per album (album slugs are bare, `-ep` / `-single` / `-compilation` suffixes for other types) using the per-album `GET /api/spotify/album?id=$ALBUM_ID` response, and write the top tracks snapshot to `releases/top-tracks.md`. `RELEASE.md` is the 18-section master release-management document — Step 5 fills the Spotify-derivable fields and leaves the rest as `⚠️ TBD`. The full template + the field-by-field Spotify mapping live in the `artist-workspace` skill at `references/release-template.md`. Tick `- [ ] 5.` → `- [x] 5.`.

## Step 6: Search the web for additional socials

```bash
SOCIALS_SEARCH=$(curl -sS -X POST "https://api.recoupable.dev/api/research/web" \
  "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg name "$ARTIST_NAME" \
        '{query: ($name + " official instagram tiktok twitter youtube")}')")
```

Parse the results to extract any new social URLs whose host matches the platform reference table below.

**After this step:** save the discovered URLs (one per platform) into the `## Notes` section so step 7 can read them without re-querying. Tick `- [ ] 6.` → `- [x] 6.`.

## Step 7: PATCH the artist with the discovered socials

```bash
curl -sS -X PATCH "https://api.recoupable.dev/api/artists/$ARTIST_ID" \
  "${AUTH[@]}" \
  -H "Content-Type: application/json" \
  -d '{
    "profileUrls": {
      "INSTAGRAM": "https://instagram.com/...",
      "TIKTOK": "https://tiktok.com/@...",
      "TWITTER": "https://x.com/...",
      "YOUTUBE": "https://youtube.com/@..."
    }
  }'
```

Only include keys for platforms you actually found URLs for. The PATCH preserves existing socials for keys you omit, so this is safe to run incrementally.

Full body schema: `https://docs.recoupable.dev/api-reference/artists/update`.

**After this step:** tick `- [ ] 7.` → `- [x] 7.`.

## Step 8: Synthesize the knowledge base

Combine the structured research outputs (`## Research` section — profile, career, playlists, web), the Spotify catalog (`releases/`), and the discovered socials into a comprehensive markdown report. Recommended sections:

- Artist biography and origin
- Discography highlights (top tracks + key albums)
- Spotify presence (genres, listener count, notable playlists)
- Social media footprint
- Recent activity / press
- Notable collaborations and achievements

Append the report to the same `RECOUP.md` you scaffolded in Step 0 — add it as a `## Knowledge base` section below the checklist. The path is `artists/$ARTIST_SLUG/RECOUP.md`.

This dovetails with the `artist-workspace` skill's filesystem conventions, so future sandbox sessions can read both the checklist and the KB without needing a hosted URL.

**After this step:** tick `- [ ] 8.` → `- [x] 8.`. With every box ticked, the artist is fully set up.

## Platform key reference

`profileUrls` keys are **uppercase** platform identifiers, inferred from the URL host. Recognized platforms:

| URL contains | `profileUrls` key |
| --- | --- |
| `spotify.com` | `SPOTIFY` |
| `instagram.com` | `INSTAGRAM` |
| `tiktok.com` | `TIKTOK` |
| `x.com`, `twitter.com` | `TWITTER` |
| `youtube.` | `YOUTUBE` |
| `apple.com` | `APPLE` |
| `facebook.com` | `FACEBOOK` |
| `threads.net`, `threads.com` | `THREADS` |

URLs that don't match any of these are silently skipped — they won't be saved.

## What this chain doesn't enforce

There's no server-side orchestrator forcing each step to run in order — the chain is honor-system. The `RECOUP.md` checklist is what gives you determinism in practice: as long as you tick boxes and persist values after each step, a fresh turn (or a different agent) can pick up exactly where the last one stopped. If you skip a checkbox or skip the persist, the next turn won't know that step ran and may either redo it or, worse, treat downstream calls as ready when they aren't.

A few constraints to honor:

- **Run steps in order.** The frontmatter values written at one step are the inputs for later steps.
- **Don't continue past a 4xx/5xx without recovery.** Leave the box unchecked, write the error to `## Notes`, and resolve before resuming.
- **Treat the file as the source of truth.** If something isn't on disk, don't assume it ran.
