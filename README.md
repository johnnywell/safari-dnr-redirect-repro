# Safari: matching DNR redirect rules no longer combine

A minimal repro for a behavior change in WebKit's `declarativeNetRequest`. The change landed in
[320739@main](https://commits.webkit.org/320739@main) for
[bug 313963](https://bugs.webkit.org/show_bug.cgi?id=313963)
([PR #73161](https://github.com/WebKit/WebKit/pull/73161)). It is in Safari Technology Preview
(WebKit 21626.1.8.19.2). It is not in Safari 27.0 (WebKit 21625.1.29.18.28).

## What changed

**Before:** Safari applied every matching `redirect` action in turn. With
`transform.queryTransform.removeParams` redirects the effect adds up: each matching rule removes
its own parameters.

**After:** Safari applies only one redirect per load.
- It picks the highest priority. On a tie, the later-declared rule wins.
- A rule that matches but has nothing to remove still wins.
- The result isn't run through the rules again.

The check is in
[`ContentExtensionsBackend::processContentRuleListsForLoad`](https://github.com/WebKit/WebKit/blob/09e9de9492291368b7300a9da5ee6d0d6d7bd6ca/Source/WebCore/contentextensions/ContentExtensionsBackend.cpp#L316-L327).
It reads a flag shared across every loaded rule list, and each extension has its own list. So
once one extension's redirect is recorded, other extensions' redirects for that load are skipped
too. Which extension comes first depends on the order of a hash map.

## The extensions

| Folder | Rules (all `main_frame`, priority 1) |
|---|---|
| `extension-a/` | **1:** `urlFilter: "*"`, removes `aaa`. **2:** `requestDomains: ["example.com"]`, removes `bbb` |
| `extension-b/` | **1:** `urlFilter: "*"`, removes `ccc` |

## Running it

1. Safari → Settings → Advanced → turn on **Show features for web developers**. Also turn on
   **Show full website address**, so the address bar shows the query string.
2. Settings → Developer → turn on **Allow unsigned extensions**. Then click **Add Temporary
   Extension…** and choose `extension-a`.
3. Allow it on example.com and example.org: click its toolbar button → **Always Allow…**.
4. Turn off any other extension that redirects, such as URL cleaners and ad blockers. Their
   redirects would interfere with the results.
5. Run tests 1 and 2. Then add `extension-b` the same way and run test 3.

## Tests

| # | Loaded | Open | If every matching redirect applies | If only the first applies |
|---|---|---|---|---|
| 1 | A | `https://example.com/?aaa=1&bbb=2&id=1` | `?id=1` | `?aaa=1&id=1` (rule 2 won the tie) |
| 2 | A | `https://example.com/?aaa=1&id=1` | `?id=1` | `?aaa=1&id=1` (rule 2 matched, removed nothing, and still won) |
| 3 | A + B | `https://example.org/?aaa=1&ccc=3&id=1` | `?id=1` | `?aaa=1&id=1` or `?ccc=3&id=1` (one extension's redirect dropped) |

### Observed

| # | Safari 27.0 (21625.1.29.18.28) | Safari Technology Preview (21626.1.8.19.2) |
|---|---|---|
| 1 | _pending_ | _pending_ |
| 2 | _pending_ | _pending_ |
| 3 | _pending_ | _pending_ |

### Already observed with a production extension

[Unfurl](https://unfurl.software) is a Safari URL cleaner with one generic rule (~400 params),
~1,200 site rules and ~25 URL-pattern rules, all `queryTransform.removeParams` at priority 1. It
was tested on 2026-09-29, and a local server logged what each browser sent:

| Request query | Safari 27.0 | Safari Technology Preview |
|---|---|---|
| `?from=glia&utm_source=em&id=1` | `?id=1` | `?utm_source=em&id=1` |
| `?utm_source=em&af_xp=1&id=1` | `?id=1` | `?utm_source=em&af_xp=1&id=1` |

The second row's site rule matched `&af_xp=` but only removes `c` and `pid`, so it removed
nothing. It still suppressed the generic rule.

## Why it matters

URL cleaners use a generic rule plus site-specific rules; AdGuard's `$removeparam` lists convert
the same way. Under the new behavior, every site with its own rule loses the generic cleaning.
For Unfurl that includes YouTube, Amazon, Facebook, LinkedIn and eBay. The workaround is to copy
the generic params into every site rule, which takes Unfurl's ruleset from 0.32 MB to 6.69 MB of
JSON.

[w3c/webextensions#468](https://github.com/w3c/webextensions/issues/468) discusses the
cross-browser direction: apply all equal-priority redirect rules at once. Firefox supports it.
Browser vendors have also
[agreed not to standardize](https://developer.chrome.com/docs/extensions/reference/api/declarativeNetRequest)
the order of equal-priority rules. So applying every equal-priority `transform` redirect, as
Safari 27.0 effectively does, conflicts with nothing that was agreed.
