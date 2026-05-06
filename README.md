# forma-dev-reactseo
Forma Dev: React SEO for anyone building on empty React shells like Lovable

# Forma Dev: React SEO

A Claude skill for fixing search engine invisibility in client-side rendered React apps. Works with any Vite, CRA, or Lovable project — anywhere your users see a full page but crawlers see an empty `<div id="root"></div>`.

Free. No subscription.

---

## The problem

React SPAs send a minimal HTML shell to the browser, then JavaScript renders everything else. For users this is fine. For search engine crawlers it's a disaster — they do a simple HTTP request, get an empty container, and have nothing to index.

It gets worse. Lovable and most React scaffolds build navigation using `onClick={() => navigate('/page')}` — which renders as a `<button>`, not an `<a href>`. Crawlers discover pages by following links. Buttons are invisible to them. Your entire internal link structure may not exist as far as Google is concerned.

This skill fixes both problems without switching frameworks or rebuilding your stack.

---

## What you get

- A full audit of every navigate call across your public pages
- A centralised route registry as the single source of truth for all indexable routes
- A prerender cache edge function with an SEO debug dashboard
- A dynamic sitemap generator
- A noscript fallback for raw HTML crawlers
- A master prompt you can paste directly into Lovable or use with Claude Code

---

## What it does not do

This skill works within the constraints of client-side rendered apps. It does not:

- Convert your app to Next.js or any SSR framework
- Add custom build scripts or modify your web server
- Use Rendertron (deprecated) or any external rendering service

It fixes crawlability through proper anchor tags, a sitemap, a noscript fallback, and an optional CDN-level prerender layer using Cloudflare Workers.

---

## How to install

1. Create a new Claude project
2. Upload `skill.md` as a knowledge file in the project
3. Paste your codebase context or describe your stack in the conversation
4. Run the audit prompt to start

---

## What you need before you start

- A React app (Vite, CRA, or Lovable) deployed somewhere
- Access to your Supabase project (for edge functions and the prerender cache table)
- Optionally: a Cloudflare account for CDN-level bot routing

---

## Repo structure

```
dev/
  README.md     ← you are here
  skill.md      ← load this into your Claude project
```

---

## Part of the Forma library

[getforma.co](https://getforma.co) — Claude skill bundles for CS, PMM, dev, and more.

Forma PMM (the paid bundle) is at [Gumroad](https://andreasaez.gumroad.com/l/enwoc) if you need the full product marketing workflow.
