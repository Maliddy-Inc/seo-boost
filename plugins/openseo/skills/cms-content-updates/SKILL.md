---
name: cms-content-updates
description: "Turn OpenSEO site audit and Search Console findings into concrete edits to a Payload CMS site: find the pages worth fixing, map each URL to its CMS document, and save improved titles, meta descriptions, headings, alt text, and copy as drafts through Payload's MCP server. Use when the user wants SEO fixes applied to their CMS content, not just reported."
---

# OpenSEO CMS Content Updates

## Goal

Close the loop between SEO analysis and the content itself. Find the few pages where a content edit is the fix, prove it with OpenSEO data, write the edit, and save it in the CMS as a draft the user reviews before it goes live.

Use this when the user runs a site on Payload CMS with the Payload MCP plugin connected and asks to fix, improve, or update pages based on SEO data. If they only want to know what is wrong, use `seo-audit` instead. Other CMS MCP servers that expose find and update tools can follow the same steps; adapt the tool names and say which ones you used.

## Inputs and project context

- Domain and `projectId` (`list_projects`; if no project matches, `create_project`).
- Call `get_project_context` first. Use `business_overview` and key pages to judge which pages matter and to keep the brand voice. If `business_overview` is empty, infer it from the site, confirm it with the user in one question, and write it back with `update_project_context`.
- The CMS connection. Payload's MCP endpoint is `/api/mcp` on the Payload server, authorized with a user API key (`Authorization: users API-Key <key>`). The user needs `mcpPlugin` in their Payload config with `update` left enabled for the collections that hold pages. Ask for nothing else.
- On finish, append to the research log with `update_project_context`: `{ appendResearchLog: { summary: "CMS content updates: <domain>. <n> drafts saved: <URLs>. Recheck after <date>." } }`.

## Tools

### OpenSEO MCP

- `whoami`: confirm the connection and credits. If OpenSEO is not connected, stop and ask the user to connect it.
- `get_search_console_performance`: the primary signal. Use `dimensions: ["query", "page"]` over `last_28_days` (or a longer window for small sites) to see which queries each page actually earns impressions for. `minPosition`/`maxPosition`/`minImpressions` filter server-side. Missing access is a coverage gap; continue with audit data and say so.
- `get_search_opportunities`: when GA4 is also connected, pages ranking 4–20 scored by demand and business value. Prefer its order over your own when it is available.
- `list_site_audits`, `get_audit_issues`, `get_audit_pages`: reuse an audit under 14 days old. Otherwise ask before `run_site_audit`, since it spends credits, then poll `get_audit_status` with a minute or two between checks. Every issue carries a `how_to_fix`.
- `inspect_urls`: confirm index state and the Google-selected canonical for any page you plan to edit. A page that is not indexed or canonicalized elsewhere needs a technical fix first; an edit will not help it.
- `get_keyword_metrics`: demand for the query a rewrite targets, when Search Console impressions alone cannot tell you which phrasing people use.

### Payload MCP

- `getConfigInfo`: the collection and global slugs this API key can reach.
- `getCollectionSchema`: field names for a collection. Look for the URL field (usually `slug`, sometimes `path` or nested-docs `breadcrumbs`), the SEO group from Payload's SEO plugin (`meta.title`, `meta.description`, `meta.image`), the main content field, and whether the collection has drafts (a `_status` field).
- `findDocuments`: map a URL to a document, for example `where: { slug: { equals: "pricing" } }`. Read the current values before proposing any change.
- `updateDocument`: save an edit by `id`, with `draft: true`. Send only the fields you change.

Never call `deleteDocuments` or `createDocuments` in this workflow, and never change a `slug` or other URL field.

## Workflow

### 1. Connect and orient

`whoami`, resolve the project, `get_project_context`, then `getConfigInfo`. If the Payload tools are missing, stop and tell the user what to set up (the plugin, an API key, and the MCP client entry). Do not fall back to editing the site some other way.

Call `getCollectionSchema` for each collection that renders public pages. Write down, per collection: URL field and how it maps to a path, SEO fields, content field type, drafts on or off, and locales if the config has them.

### 2. Find pages where content is the fix

Gather candidates from free reads first:

- **Low click-through**: pages with many impressions in positions 1–10 and a click-through rate well below their neighbours. The usual fix is a title and meta description that answer the query the page actually earns impressions for.
- **Near misses**: pages averaging positions 4–20 for queries that fit the business. The fix is usually content that answers the searcher's question more completely: a missing section, a clearer heading, a direct answer near the top.
- **Audit issues with a content fix**: `missing-title`, `duplicate-title`, `title-too-long`, `title-too-short`, `missing-meta-description`, `duplicate-meta-description`, `meta-description-too-long`, `meta-description-too-short`, `missing-h1`, `multiple-h1`, `heading-order-skip`, `thin-content`, `images-missing-alt`, `no-outgoing-links`.

Leave technical issues (server errors, redirects, canonical conflicts, noindex, blocked crawls, broken links in templates) out of the edit plan. List them in the summary as "needs a developer" instead.

Rank candidates by search demand that fits the business, then by how directly an edit fixes the observed problem. Keep the plan to the ten or so pages that matter most; a site-wide find-and-replace is not this workflow.

### 3. Map each URL to its document

For every candidate, find its document with `findDocuments` on the URL field. Handle the home page (often a `home` slug or a global), trailing slashes, nested paths, and locale prefixes. If a URL maps to no document or to more than one, do not guess: list it as unmapped and move on.

Read the current document. Compare the CMS values with what the audit saw on the live page; if they differ, the page template may override the CMS field or a newer draft may exist. Ask before editing such a page.

### 4. Write the edits

For each page, write one row in a plan table: URL | document (collection and id) | field | current | proposed | evidence (query, impressions, clicks, CTR, position, date range, or the audit issue).

Rules for the edits:

- Titles: lead with the query the page earns impressions for, in the words searchers use. Keep the site's existing brand suffix convention. Aim for roughly 50–60 characters; never pad to hit a length.
- Meta descriptions: say what the page offers and who it is for, in about 140–160 characters. A meta description influences clicks, not rankings; say so if the user expects a ranking change.
- Body copy: add or tighten the specific section the searcher is missing. Do not rewrite a page that already ranks in the top three for its main query; protect it instead.
- Rich text: Payload stores Lexical rich text as a JSON tree. Copy the existing tree, change only the nodes you mean to change, and keep every other node, format flag, and link as it was.
- Alt text: describe what the image shows and why it is on the page. Skip decorative images.
- Write in the site's language and voice. No keyword stuffing, no invented facts, prices, statistics, or claims about the product. If an edit needs a fact you do not have, ask.

Show the plan table to the user and wait for approval before writing anything. The user can approve all, some, or none of the rows.

### 5. Save as drafts

- If the collection has drafts, call `updateDocument` with `draft: true` and only the approved fields. Payload saves a new draft version and leaves the published page unchanged.
- If the collection has no drafts, every update goes live immediately. Say this plainly and get a second, explicit approval for those rows, or leave them as copy for the user to paste.
- Save one document first, read it back with `findDocuments` (with `draft: true`), and confirm the fields changed as intended and nothing else moved. Then save the rest.
- If an update fails, report the error for that row and continue with the others. Do not retry with broader data.
- Do not publish. Publishing is the user's step in the Payload admin, after they review the drafts.

### 6. Hand over

Write back the research-log entry from "Inputs and project context". Set the recheck date four to six weeks out: Google needs time to recrawl, and Search Console data lags about three days.

When the user returns for the recheck, compare `get_search_console_performance` for the edited pages over equal-length windows before and after the publish date. One window is not a trend; seasonality and algorithm updates also move these numbers.

## Output format

Reply in chat, short and scannable:

1. **Saved as drafts**: a table of URL | field | before | after, with a link to each document in the Payload admin when you know the admin URL (`<admin>/collections/<slug>/<id>`).
2. **Not changed**: unmapped URLs, rows the user declined, and technical issues that need a developer, one line each with the reason.
3. **Next steps**: review and publish the drafts, and the recheck date.

If the user also wants a shareable record, deliver it through `seo-report`, saving with `skill: "cms-content-updates"`.

## Guardrails

- Every edit traces back to a specific observation in OpenSEO data. No speculative "SEO best practice" rewrites.
- Observations are not causes. A low click-through rate can come from rich results or ads above the page, not only from the title.
- The user's CMS is production data. Read before you write, write only approved fields, and prefer drafts every time.
- Calm, plain tone. Gloss terms of art on first use: meta description, click-through rate, canonical, alt text.
