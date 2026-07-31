# Project Handoff — Shopify Theme `minimog-4-1-0` (chz119-e1)

Created: 2026-08-01

## Goal
Extract the live Shopify theme from store `chz119-e1` (`minimog-4-1-0`), keep the code in a GitHub repo, and eventually push a **customized version back to Shopify** to get our own coded website live on the store.

## Status Summary
- ✅ Theme extracted (all 366 files) — as-is
- ✅ Uploaded to GitHub: https://github.com/hanzlahsfc/shopify (branch `main`, commit `fac318c`)
- ✅ Shopify CLI 4.6.0 installed + authenticated to the store
- ⚠️ `shopify theme check` reports `ValidJSON` errors in `config/settings_schema.json` — **unresolved**, see below
- ⏳ Not yet done: push custom theme back to Shopify

## Key Locations
| What | Path |
|---|---|
| Local repo (git) | `C:\Users\Hanzlahsfc\Desktop\shopify-repo` |
| Original extraction | `C:\Users\Hanzlahsfc\Desktop\Shopify_chz119-e1_theme` |
| Exploration scripts | `C:\Users\Hanzlahsfc\Desktop\Part66_Exploration\` (all `shopify_*.py`) |
| CDP bridge host | `C:\Users\Hanzlahsfc\Desktop\devtools-extension-main\host\host.js` |
| Downloaded app bundles | `C:\Users\Hanzlahsfc\Desktop\Part66_Exploration\osw_*.js`, `osw_chunks\` |

## Environment
- Windows, PowerShell 5.1
- Node v24.14.1, npm 11.11.0
- Shopify CLI 4.6.0 (installed globally via `npm i -g @shopify/cli`)
- Git user: `Hanzlah <mhanzlahirfan@gmail.com>`
- CDP Bridge: HTTP `localhost:1232` (native-messaging Chrome extension). Extension ID `emkkmmifjnjhhkfihhehdnpkpapfbdpp`. If tabs show `attached:false` or `/command` returns 500, kill the node host process on port 1232 — the extension reconnects and re-attaches automatically (~20s).

## Store / Themes
- Store: `chz119-e1.myshopify.com` (admin: `admin.shopify.com/store/chz119-e1`)
- `minimog-4-1-0` — **live** — id `138225385557`
- `susie` — unpublished — id `138149920853`

## GitHub
- Remote: https://github.com/hanzlahsfc/shopify.git
- Local clone: `Desktop\shopify-repo`, branch `main`, tracks `origin/main`
- Contains full theme: `assets` (119), `snippets` (107), `sections` (71), `templates` (33), `locales` (31), `config` (2), `layout` (2), `blocks` (1)
- `_filelist.json` (extraction manifest w/ sizes + md5) was excluded from git; original copy is in `Shopify_chz119-e1_theme\`

## How the Theme Was Extracted (for reference / re-extraction)
1. CDP Bridge drives Chrome tab `242859233` = admin editor page `.../themes/138225385557/editor`.
2. Admin GraphQL endpoint (from inside that tab):
   - URL: `/store/chz119-e1/api/unstable/graphql.json`
   - Auth: **`X-CSRF-Token` header + `credentials: include`** (cookies). Token lives in the page's embedded `script[type="text/json"]` → `csrfToken`. No App Bridge session token needed.
   - Client headers: `Content-Type: application/json`
3. List files:
   ```graphql
   query Files($themeId: ID!, $first: Int!){ onlineStore { theme(id:$themeId){ id files(first:$first){ nodes{ filename size checksumMd5 } } } } }
   ```
   - Correct GID: `gid://shopify/OnlineStoreTheme/138225385557` (NOT `gid://shopify/Theme/...`)
4. Fetch body per file (field alias `f0`,`f1`,… for batching 25/call):
   ```graphql
   query F($themeId:ID!){ onlineStore{ theme(id:$themeId){ id f0: file(filename:"..."){ body{ __typename
     ... on OnlineStoreThemeFileBodyText { content }
     ... on OnlineStoreThemeFileBodyBase64 { contentBase64 }
     ... on OnlineStoreThemeFileBodyUrl { url } } } } } }
   ```
   - Most files → text. Binary (2 PNGs) → base64. 3 large JS bundles (`photoswipe.js`, `swiper.js`, `vendor.js`) → signed GCS URL, fetched separately from browser context.
5. Writer: `Part66_Exploration\shopify_download_all.py` (+ `shopify_fetch_url_files.py` for the 3 URL files).
6. Verification: text/liquid files' local MD5 == Shopify's stored `checksumMd5` (exact). JSON files are pretty-printed with Shopify's auto-generated `/* ... */` header (normal) — valid JSON after stripping header.

## Shopify CLI — Authenticated Commands
```powershell
shopify theme list --store=chz119-e1.myshopify.com
shopify theme pull --store=chz119-e1.myshopify.com -t 138225385557 -o <dir>   # pull live theme
shopify theme push --store=chz119-e1.myshopify.com -u -t <name>               # push to a NEW unpublished theme
shopify theme dev --store=chz119-e1.myshopify.com                              # live-sync preview theme
shopify theme publish -t <id>                                                  # publish to live
shopify theme check                                                           # validate local theme
```
Note: newer CLI requires `--store` flag; login is cached after first auth.

## ⚠️ Known Issue (NEXT SESSION: resolve first)
`shopify theme check` fails with `ValidJSON: Incorrect type. Expected "string".` in `config/settings_schema.json` around lines 228–248 (`"content": { "en": "..." }`).
- Likely cause: minimog uses a newer settings schema format with locale objects; older CLI validator expects plain strings. OR the file genuinely needs the `"en"` wrapper removed/flattened.
- **Do NOT push to live until this is understood** — a bad `settings_schema.json` can break the theme editor.
- Verify: `shopify theme check` (above) and open the theme editor in admin to confirm settings load.
- Possible fix: flatten `"content": {"en": "..."}` → `"content": "..."` if the live theme editor actually uses the plain form. Compare against `config/settings_data.json` and the actual admin editor behavior before editing.

## Next Steps (suggested order)
1. Resolve the `settings_schema.json` ValidJSON issue (see above).
2. Decide deployment strategy — safest path:
   - `shopify theme dev` to preview edits live in real time, OR
   - Edit in the repo, `shopify theme push -u` to a disposable unpublished theme, review via `shopify theme open`, then `shopify theme publish`.
3. Remember: **data lives in the store**, not the theme. Products/collections/pages/blogs/images must be created in admin. App-dependent features (Judge.me, FoxKit, SocialShopWave, SSW — snippets exist in the code) need those apps installed.
4. After editing, commit changes in `shopify-repo` and push to GitHub (branch `main`).
5. Consider running `shopify theme dev` + the "Preview" workflow for the fastest iteration loop.

## Useful CDP/Extraction Notes (if re-extraction needed)
- Re-attach all tabs after bridge hiccup: kill PID on `:1232` (host restarts, extension re-attaches).
- Admin embedded state (csrfToken/sessionId) read from `document.querySelectorAll('script[type="text/json"]')`.
- OOPIF (online-store-web app iframe) is unreachable via the bridge; the CSRF-token GraphQL trick above bypasses it entirely — don't waste time on the iframe.
