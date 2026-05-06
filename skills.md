# Forma Dev: React SEO Skill

You are running the Forma Dev React SEO workflow. Your job is to fix search engine invisibility in a client-side rendered React application without switching frameworks, modifying the web server, or using external rendering services.

Follow the steps below in order. Ask the user for missing context before proceeding. At each step, show the user what you found and what you changed.

---

## Context: why this is broken

React SPAs send a minimal HTML shell to the browser. JavaScript renders the actual content client-side. Crawlers do a simple HTTP request — they receive an empty `<div id="root"></div>` and have nothing to index.

A second problem compounds this: most React scaffolds (including Lovable) build navigation using `onClick={() => navigate('/page')}`. This renders as a `<button>` element. Crawlers discover pages by following `<a href>` links — buttons are completely invisible to them.

This skill fixes both without a framework switch.

---

## Constraints

You are working within a client-side rendered React app. You cannot:

- Add custom build scripts or prerender plugins
- Modify the web server or add middleware
- Use Rendertron (deprecated) or any external rendering service
- Switch to Next.js or any SSR framework

Work within Vite + React + Supabase architecture.

---

## Step 1 — Navigate audit and fix

Search every component and page file for `onClick` handlers that call `navigate()` for internal page links.

For every instance on a public-facing page (not behind auth):

**Convert from:**
```jsx
<Button onClick={() => navigate('/pricing')}>See Pricing</Button>
```

**Convert to:**
```jsx
<Button asChild><Link to="/pricing">See Pricing</Link></Button>
```

This renders as a real `<a href="/pricing">` in the DOM — crawlable by bots, still handles client-side navigation for users.

Rules:
- Only convert public pages. Do not convert navigate calls inside authenticated or dashboard pages.
- Use React Router `<Link>` not plain `<a>` tags — plain `<a>` tags trigger full page reloads.
- After converting, verify with the user: right-click any CTA → Inspect Element → should show `<a href>` not `<button>`.

Show the user a full list of what you found and what you converted before making any changes.

---

## Step 2 — Route registry

Create `src/data/siteRoutes.ts` — a single source of truth for all public, indexable routes.

Each entry must include:

```typescript
{
  path: string,
  title: string,
  description: string,
  h1: string,
  expectedLinks: string[],
  lastModified: string,       // ISO date
  priority: number,           // 0.0–1.0
  changeFreq: 'always' | 'hourly' | 'daily' | 'weekly' | 'monthly' | 'yearly' | 'never'
}
```

This file is used by the sitemap generator, the SEO debug dashboard, and the prerender cache as a fallback when a fetch returns an empty shell.

---

## Step 3 — Prerender cache edge function

Create `supabase/functions/prerender/index.ts`.

The function must:

- Accept `?path=/some-page`, optional `?refresh=true`, optional `?bulk=true`
- Fetch the published site URL + path via HTTP GET
- Parse the returned HTML using string/regex to extract: title, meta description, H1, H2s, internal links
- Fall back to populating from the route registry when the fetch returns an SPA shell (no content in body)
- Cache results in a `prerender_cache` database table
- Compute a health status:
  - `green` — title, description, H1, and at least one internal link all present
  - `yellow` — some elements missing
  - `red` — critical elements missing
- Track source of each field: `fetched` or `registry_fallback`
- Include `extraction_notes` explaining what was found and where
- Require admin authentication
- Validate the path parameter to prevent SSRF: must start with `/`, no `@`, `//`, `\`, or protocol schemes
- Rate limit: 5 bulk scans per hour, 60 single scans per hour

Do not use Rendertron. Do not use any external rendering service.

---

## Step 4 — Database table

Create `prerender_cache` with these columns:

```sql
path                text PRIMARY KEY,
html_snapshot       text,
title               text,
meta_description    text,
h1                  text,
h2s                 jsonb,
internal_links      jsonb,
internal_link_count integer,
status              text,
rendered_at         timestamptz,
source              text,
health              text,
extraction_notes    jsonb,
registry_complete   boolean
```

RLS policy:
- Service role: read and write
- Admin role: read
- Anonymous: blocked

---

## Step 5 — Sitemap edge function

Create `supabase/functions/generate-sitemap/index.ts`.

Generates valid XML from the route registry with `<lastmod>`, `<priority>`, and `<changefreq>` per route. Requires admin auth.

Keep `robots.txt` pointing to the static `/sitemap.xml` — the edge function is a generation tool, not the live endpoint.

Ensure `robots.txt` blocks `/admin/` paths. Do not remove any existing disallow rules.

---

## Step 6 — SEO debug dashboard

Create `/admin/seo-debug`. Requires admin role check — not just authenticated.

The dashboard must include:

**Single page inspector**
Enter a path, scan it, see extracted title, description, H1, H2s, internal links with health badge and extraction notes showing source of each field.

**Crawl coverage table**
All routes from the registry showing health status, what is present and what is missing, last scanned date. Per-row refresh and bulk scan button.

**Sitemap regeneration button**

Health scoring logic must be based on what was actually extracted — not on whether the HTTP request returned a 200. A 200 response with an empty SPA shell is not healthy. Distinguish clearly between content found in fetched HTML vs content pulled from the registry as fallback.

---

## Step 7 — noscript fallback

Add a `<noscript>` block to `index.html` with:
- Site name
- Brief description
- Key internal links as real `<a href>` tags

This gives crawlers that do not execute JavaScript at least some content and links to follow.

---

## Step 8 — Remove lazy loading from public pages

Remove `React.lazy()` and `Suspense` from all public marketing pages. Only keep lazy loading for authenticated pages (dashboard, tools, settings).

Public page navigation should have zero loading flash.

---

## Step 9 — CORS and robots

Confirm CORS allowed origins cover:
- `.lovable.app` preview domains
- `.lovableproject.com` preview domains
- Your production domain

Ensure `robots.txt` blocks `/admin/`. Do not remove existing rules.

---

## Optional: CDN-level prerendering

If Google Search Console shows pages still missing or showing thin content after a few weeks, add bot routing via Cloudflare Workers.

**How it works:**
Move DNS to Cloudflare free tier. Create a Worker that checks the incoming user agent for bot signatures. Route bot requests to your prerender edge function. Humans get the normal SPA. Bots get complete HTML.

The prerender cache built in Step 3 slots directly into this — no additional work needed.

Do not use Prerender.io or any paid managed service unless the user explicitly requests it.

---

## Master prompt

Use this prompt to implement the full fix in Lovable or with Claude Code:

```
Our site is a client-side rendered React SPA. Search engine bots are receiving 
an empty <div id="root"></div> shell instead of rendered content. We need to fix 
crawlability without switching frameworks or requiring a full technical rebuild.

Constraints: we cannot add custom build scripts, modify the web server, or add 
middleware. Work within the existing Vite + React + Supabase architecture.

Please implement the following in order:

1. Navigate audit — find every onClick navigate() call on public pages and 
   convert to <Button asChild><Link to="..."> so crawlers see real <a href> tags. 
   Show me the full list before making changes. Do not touch authenticated pages.

2. Route registry — create src/data/siteRoutes.ts as a single source of truth 
   for all public routes, with path, title, description, H1, expected links, 
   lastModified, priority, changeFreq.

3. Prerender edge function — Supabase edge function that fetches each page, 
   parses HTML for SEO elements, falls back to route registry when the fetch 
   returns an empty shell, caches in prerender_cache table, tracks health status 
   and source. No Rendertron. No external rendering services. SSRF protection on 
   path parameter. Rate limited.

4. prerender_cache table — path (PK), title, meta_description, h1, h2s, 
   internal_links, internal_link_count, status, rendered_at, source, health, 
   extraction_notes, registry_complete. RLS: service role read/write, admin read, 
   block anonymous.

5. Sitemap edge function — generates XML from route registry with lastmod, 
   priority, changefreq. Admin auth required. robots.txt points to /sitemap.xml.

6. SEO debug dashboard at /admin/seo-debug — admin role check, single page 
   inspector, crawl coverage table, sitemap regeneration. Health scoring based 
   on extracted content, not HTTP status. Distinguish fetched vs registry fallback.

7. noscript fallback in index.html — site name, description, key internal links 
   as real <a> tags.

8. Remove React.lazy() from all public marketing pages. Keep lazy loading only 
   for authenticated pages.

9. CORS: cover .lovable.app, .lovableproject.com, and production domain.
   robots.txt: block /admin/. Do not remove existing rules.

After each step, confirm what was changed before moving to the next.
```

---

## How to verify it worked

**View page source** (not Inspect Element)
Right-click any page → View Page Source. You should see your `<title>`, meta description, Open Graph tags, and the `<noscript>` block with internal links. The `<div id="root">` being empty outside of noscript is expected for an SPA.

**Check links from the command line**
```bash
curl -s https://yoursite.com/ | grep '<a href'
```
You should see links from the noscript block at minimum.

**Inspect a CTA button**
Right-click any call-to-action → Inspect Element. Should show `<a href="/path">` not `<button>`.

**Count internal links in the DOM**
Open dev tools console and run:
```js
document.querySelectorAll('a[href^="/"]').length
```
Should be 10+ on any page with navigation and CTAs.

**Check your sitemap**
Visit `yoursite.com/sitemap.xml` directly. Every public page should be listed.

**Google's Rich Results Test**
Go to `search.google.com/test/rich-results` and enter your URL. Shows you what Googlebot actually sees.

**Google Search Console**
Submit your sitemap. Use URL Inspection on 5–10 key pages. Watch for pages moving from "not indexed" to indexed over the following days.

---

## Common failure modes

**Lovable suggests Rendertron**
Rendertron's public hosted instance is deprecated. If Lovable builds your prerender function around a Rendertron endpoint it will fail silently. Tell it explicitly to use direct HTTP fetch with regex parsing, falling back to route registry data.

**Debug dashboard shows everything green but nothing is indexed**
The health check is marking pages as healthy based on HTTP 200 status, not on whether content was actually extracted. Make sure health scoring checks for non-empty H1, non-empty meta description, and at least one internal link — not just a successful request.

**Navigation converting to plain `<a>` tags causes page reloads**
Plain `<a href>` tags trigger full page reloads. React Router `<Link>` renders as `<a>` in the DOM but handles navigation client-side. Always use `<Link>`, not `<a>`.

**Lazy loading causes flash after fixing navigation**
Remove `React.lazy()` from public pages. The flash becomes noticeable after converting to `<Link>` because transitions happen instantly. Lazy loading is fine for authenticated pages only.

**SSRF on the prerender function**
The function accepts a path parameter. Validate it strictly: must start with `/`, no `@`, `//`, `\`, or protocol schemes. An unvalidated path parameter allows an attacker to make your edge function fetch arbitrary URLs.
