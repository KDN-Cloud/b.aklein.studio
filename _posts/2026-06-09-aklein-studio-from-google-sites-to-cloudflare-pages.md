---
layout: post
title: "From Google Sites to Cloudflare Pages for the Studio"
date: 2026-06-09
author: Anthony Klein
description: >
  After years on Google Sites, I finally rebuilt aklein.studio from the ground up
  as a fully custom static site on Cloudflare Pages. Dark terminal aesthetic, persistent
  audio player, schema markup, a live wire feed, and a self-hosted mix archive backed
  by Cloudflare R2. Here is how it came together.
tags:
  - cloudflare
  - cloudflare-pages
  - cloudflare-workers
  - cloudflare-r2
  - google-sites
  - site-migration
  - static-site
  - aklein-studio
  - lounge24
  - deep-house
  - web-development
  - custom-domain
  - audio-player
  - rss-feed
  - seo
  - schema
  - self-hosted
  - studio-website
  - dj-site
---

Google Sites had been the home of aklein.studio for longer than I care to admit. It was fine when I needed something up fast. A logo, some links, a few pages. It did the job but it was never really mine. The design was constrained by whatever Google decided was acceptable. The typography was not mine. The layout was not mine. I could not add a persistent audio player. I could not write custom CSS. I could not drop in a Cloudflare Worker to pull live RSS feeds. I could not do any of the things I actually wanted to do with a music and broadcast site.

The platform debt had been building for a while. I kept telling myself I would migrate it eventually. Eventually finally arrived.

## Starting From Zero

The brief I gave myself was simple. Dark terminal aesthetic. Monospace typography. Green accent color. Fast. No frameworks, no build pipeline, no npm install. Just HTML, CSS, and vanilla JavaScript deployed as a static site on Cloudflare Pages.

The font stack landed on Rajdhani for display and Share Tech Mono for all the monospace elements. The background is a near-black `#0a0a0a` with a subtle radial gradient in the top right corner bleeding in a faint green from the accent color `#00cc44`. The whole thing is built around a CSS custom property system so colors, spacing, and typography stay consistent across every page.

The hero section sets the tone immediately. A dark studio console image at low brightness and saturation sits behind the text. Large spaced-out monospace lettering with alternating white and green accent words. The pattern that started on the home page carried through to every other page on the site for consistency.

## Cloudflare Pages and the Persistent Player

One of the things that had been impossible on Google Sites was a persistent audio player for Lounge24 Radio. The station runs 24/7 on AzuraCast and I wanted a player bar that stayed at the bottom of every page across navigation, not one that reset every time you clicked a link.

Cloudflare Pages handles the static hosting. The player itself is built in vanilla JavaScript and uses the AzuraCast public API to pull the live station status, current track metadata, and stream health. It sits fixed at the bottom of the viewport and persists across all pages via a router script that swaps page content without full reloads.

The signal indicator in the player bar shows animated EQ bars when audio is playing, a pulsing yellow dot when the status is being checked, and a red offline state if the station goes down. All driven by the AzuraCast metadata endpoint polled on a short interval.

## Pages and Structure

The site has five main pages. Home, Profile, Live, Wire, and Mixes. Each one follows the same structural pattern: sticky nav, compact hero with the cover image and accent word subline, content sections below, and the shared footer with the KDN infrastructure callout.

The Profile page documents the artist background, the gear rig, and the studio setup. The Live page handles the AzuraCast embed and the Hopalong visualizer. The Wire page is a live signal feed pulling from six RSS sources through a Cloudflare Worker that caches results in KV storage and refreshes every six hours via a cron trigger. The Mixes page is the newest addition.

## Cloudflare Workers on Your Own Domain

One of the less obvious things about Cloudflare Workers is that they do not have to live on a `workers.dev` subdomain. Every Worker I built for this site runs under my own domain. The wire feed API sits at a workers.dev address internally but the mixes API serves files through `mixes.aklein.studio` as a proper custom domain. You connect a Worker to your own domain or subdomain through the Cloudflare dashboard under Workers and Pages, then add a route or a custom domain binding. Cloudflare provisions the SSL certificate automatically and the Worker starts handling requests at that address within minutes.

This matters more than it sounds. A URL like `mixes.aklein.studio` looks intentional and permanent. A raw workers.dev address looks like scaffolding. For anything user-facing or embedded in a page, running the Worker under your own domain is the right call. The setup is four or five clicks and Cloudflare handles the DNS record creation automatically if your domain is already proxied through them.

## The Purpose of mixes.aklein.studio

`mixes.aklein.studio` is not a page. It is a dedicated file delivery endpoint backed by R2 object storage. Its only job is to serve MP3 files and cover art images to the audio player running on `aklein.studio/mixes`. The two subdomains do completely different things. The main site handles pages, UI, and navigation. The mixes subdomain handles raw file serving. Keeping them separated means if the storage backend ever needs to change, authentication needs to be added to downloads, or the file delivery layer needs to be swapped out entirely, none of that touches the main site. One change in one place.

The R2 bucket custom domain binding in the Cloudflare dashboard is what connects the bucket to the subdomain. Files uploaded to the bucket at a path like `2024-03-24-tone-deep-house-mix/mix.mp3` become immediately accessible at `https://mixes.aklein.studio/2024-03-24-tone-deep-house-mix/mix.mp3`. Cloudflare handles SSL, CDN caching, and global delivery automatically.

Since `mixes.aklein.studio` is purely infrastructure and not meant to be browsed directly, anyone who types it into a browser would land on a raw R2 404 error. To handle that cleanly I added a Cloudflare redirect rule that sends root requests to `aklein.studio/mixes` with a 301 permanent redirect. The rule expression is deliberately narrow:

```
(http.host eq "mixes.aklein.studio" and http.request.uri.path eq "/")
```

The path condition locked to `/` is critical. Without it the redirect would intercept all requests to that subdomain including the MP3 and cover art file requests the audio player depends on. A broader rule would silently break every audio player on the mixes page. With the path scoped to the root only, the redirect fires for direct browser visits while every file path passes straight through to R2 untouched. Visitors who land on the subdomain get routed to the right place. The audio player keeps working.

## The Wire Feed

The Wire page was the most technically interesting build. Six RSS feeds from Mixmag, Attack Magazine, DJ TechTools, Digital DJ Tips, Ableton, and MusicRadar get fetched by a Cloudflare Worker, parsed, cleaned, sorted by publish date, and stored in KV. The page fetches the cached JSON on load and renders a hybrid layout: a featured story block on the left with a grid of source lanes on the right, plus five curated lower cards for news, gear, software, digging routes, and recommended reads.

Getting RSS XML parsing right in a Worker without any dependencies was a good exercise. CDATA unwrapping, HTML entity decoding, excerpt truncation, title fallback chains for feeds that use non-standard tag structures. All handled in pure JavaScript.

The KV caching layer is what makes the Wire page fast. Without it the Worker would hit six external RSS endpoints on every single page load, adding latency and creating a dependency on third party uptime. With KV the Worker fetches fresh data on a schedule and every visitor gets the cached result instantly. The cron trigger fires every six hours. The page never waits on an external source.

The Worker needs one KV namespace binding named `WIRE_FEED` and a cron trigger set to `0 */6 * * *` in the Worker settings. Here is the full source:

```javascript
const FEEDS = [
  { source: "Mixmag", category: "NEWS", feed: "https://mixmag.net/rss.xml", sourceUrl: "https://mixmag.net" },
  { source: "Attack Magazine", category: "PRODUCTION", feed: "https://www.attackmagazine.com/feed/", sourceUrl: "https://www.attackmagazine.com" },
  { source: "DJ TechTools", category: "WORKFLOW", feed: "https://djtechtools.com/feed/", sourceUrl: "https://djtechtools.com" },
  { source: "Digital DJ Tips", category: "LEARNING", feed: "https://www.digitaldjtips.com/feed/", sourceUrl: "https://www.digitaldjtips.com" },
  { source: "Ableton", category: "SOFTWARE", feed: "https://www.ableton.com/en/blog/feeds/latest/", sourceUrl: "https://www.ableton.com/en/blog/" },
  { source: "MusicRadar", category: "GEAR", feed: "https://www.musicradar.com/feeds/all", sourceUrl: "https://www.musicradar.com" }
];

const FEED_KEY = "wire-feed:latest";

function decodeHtml(value = "") {
  return value
    .replace(/<!\[CDATA\[([\s\S]*?)\]\]>/g, "$1")
    .replace(/<!\[CDATA\[/g, "").replace(/\]\]>/g, "")
    .replace(/&amp;/g, "&").replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">").replace(/&quot;/g, '"').replace(/&#39;/g, "'");
}

function stripTags(value = "") {
  return decodeHtml(value.replace(/<[^>]+>/g, " ").replace(/\s+/g, " ").trim());
}

function cleanTitle(value = "") {
  return stripTags(value).replace(/\u2013/g, "-").replace(/\u2014/g, "-").trim();
}

function cleanExcerpt(value = "") {
  if (!value) return "";
  let text = stripTags(value)
    .replace(/\]\]>/g, "")
    .replace(/\bContinue reading\.\.\.?$/i, "")
    .replace(/\bRead more\.\.\.?$/i, "")
    .replace(/\s+/g, " ").trim();
  if (!text || /^[^a-zA-Z0-9]+$/.test(text)) return "";
  if (text.length <= 180) return text;
  const slice = text.slice(0, 180);
  const boundary = slice.lastIndexOf(" ");
  return `${boundary > 0 ? slice.slice(0, boundary) : slice}...`;
}

function shortenTitle(value = "", maxLength = 108) {
  if (value.length <= maxLength) return value;
  const slice = value.slice(0, maxLength);
  const boundary = slice.lastIndexOf(" ");
  return `${boundary > 0 ? slice.slice(0, boundary) : slice}...`;
}

function extractTag(block, tag) {
  const match = block.match(new RegExp(`<${tag}[^>]*>([\\s\\S]*?)<\\/${tag}>`, "i"));
  if (!match) return "";
  return decodeHtml(match[1].trim());
}

function extractTitle(itemXml) {
  let raw = extractTag(itemXml, "title");
  if (raw) return cleanTitle(raw);
  raw = extractTag(itemXml, "media:title");
  if (raw) return cleanTitle(raw);
  raw = extractTag(itemXml, "atom:title");
  if (raw) return cleanTitle(raw);
  const cdataMatch = itemXml.match(/<!\[CDATA\[([\s\S]*?)\]\]>/);
  if (cdataMatch) return cleanTitle(cdataMatch[1]);
  return "";
}

function extractItems(xml) {
  return [...xml.matchAll(/<item\b[\s\S]*?<\/item>/gi)].map(m => m[0]);
}

function parsePubDate(pubDate) {
  const parsed = new Date(pubDate);
  return isNaN(parsed.getTime()) ? new Date().toISOString() : parsed.toISOString();
}

async function parseFeed(spec) {
  const response = await fetch(spec.feed, {
    headers: { "User-Agent": "Mozilla/5.0 AKLEIN-STUDIO-WIRE-FEED" }
  });
  if (!response.ok) throw new Error(`Feed fetch failed: ${spec.feed} (${response.status})`);
  const xml = await response.text();
  return extractItems(xml).slice(0, 3).map(itemXml => ({
    source: spec.source, sourceUrl: spec.sourceUrl, category: spec.category,
    title: extractTitle(itemXml),
    shortTitle: shortenTitle(extractTitle(itemXml)),
    url: stripTags(extractTag(itemXml, "link")) || spec.sourceUrl,
    publishedAt: parsePubDate(extractTag(itemXml, "pubDate")),
    excerpt: cleanExcerpt(extractTag(itemXml, "description"))
  }));
}

async function buildPayload() {
  const items = [], failures = [];
  for (const spec of FEEDS) {
    try { items.push(...await parseFeed(spec)); }
    catch (error) { failures.push({ source: spec.source, error: String(error.message || error) }); }
  }
  items.sort((a, b) => new Date(b.publishedAt) - new Date(a.publishedAt));
  return { mode: "hybrid-feed-prototype", updatedAt: new Date().toISOString(), sources: FEEDS, items, failures };
}

async function refreshFeed(env) {
  const previousRaw = await env.WIRE_FEED.get(FEED_KEY);
  const previous = previousRaw ? JSON.parse(previousRaw) : null;
  const payload = await buildPayload();
  if ((!payload.items || payload.items.length === 0) && previous) {
    return { ok: false, usedFallback: true, payload: previous };
  }
  await env.WIRE_FEED.put(FEED_KEY, JSON.stringify(payload));
  return { ok: true, usedFallback: false, payload };
}

function jsonResponse(data, extraHeaders = {}) {
  return new Response(JSON.stringify(data, null, 2), {
    headers: {
      "content-type": "application/json; charset=utf-8",
      "cache-control": "public, max-age=300",
      "access-control-allow-origin": "*",
      ...extraHeaders
    }
  });
}

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (url.pathname === "/api/wire-feed.json") {
      const stored = await env.WIRE_FEED.get(FEED_KEY);
      if (stored) return new Response(stored, {
        headers: { "content-type": "application/json; charset=utf-8", "cache-control": "public, max-age=300", "access-control-allow-origin": "*" }
      });
      return jsonResponse({ mode: "hybrid-feed-prototype", updatedAt: null, sources: FEEDS, items: [], failures: [] }, { "cache-control": "no-store" });
    }
    if (url.pathname === "/api/refresh") {
      const result = await refreshFeed(env);
      return jsonResponse({
        ok: result.ok, usedFallback: result.usedFallback,
        updatedAt: result.payload?.updatedAt || null,
        itemCount: result.payload?.items?.length || 0
      }, { "cache-control": "no-store" });
    }
    return new Response("Not found", { status: 404 });
  },
  async scheduled(_controller, env, ctx) { ctx.waitUntil(refreshFeed(env)); }
};
```

## The Mix Archive

The most recent addition to the site is the Mixes page backed by Cloudflare R2 object storage. Fifteen recorded deep house sessions from 2023 and 2024 live in a dedicated R2 bucket at `mixes.aklein.studio`. Each mix gets its own folder following a `YYYY-MM-DD-tone-mix-title` naming convention with `mix.mp3` and `cover.jpg` inside.

A second Cloudflare Worker called `aklein-mixes-api` reads the bucket on every request, lists all folders, parses the folder names into readable titles and dates, and returns structured JSON. The Mixes page fetches that JSON and renders a card grid: full bleed cover art with a dark overlay, the mix title and date overlaid on the image, a native HTML5 audio player, and a download button. No manual JSON editing, no redeployment. Upload a new folder to R2 and the mix appears on the next page load automatically.

The Worker needs one R2 bucket binding named `MIXES_BUCKET` pointing at your bucket. Here is the full source:

```javascript
// aklein-mixes-api
// R2 binding: MIXES_BUCKET
// Custom domain: mixes.aklein.studio serves files directly from R2

const CORS_HEADERS = {
  "access-control-allow-origin": "*",
  "access-control-allow-methods": "GET, OPTIONS",
  "access-control-allow-headers": "*"
};

function jsonResponse(data, status = 200, extraHeaders = {}) {
  return new Response(JSON.stringify(data, null, 2), {
    status,
    headers: { "content-type": "application/json; charset=utf-8", "cache-control": "public, max-age=300", ...CORS_HEADERS, ...extraHeaders }
  });
}

function parseSlug(slug) {
  const dateMatch = slug.match(/^(\d{4}-\d{2}-\d{2})-(.+)$/);
  if (!dateMatch) return { date: null, title: formatTitle(slug) };
  return { date: dateMatch[1], title: formatTitle(dateMatch[2]) };
}

function formatTitle(slug) {
  return slug.split("-").map((word, index) => {
    const cap = word.charAt(0).toUpperCase() + word.slice(1);
    if (/^vol\d+$/i.test(word)) return cap.replace(/vol(\d+)/i, "Vol. $1");
    if (/^ep\d+$/i.test(word)) return cap.replace(/ep(\d+)/i, "Ep. $1");
    if (index === 0 && word.toLowerCase() === "tone") return "Tone —";
    return cap;
  }).join(" ").replace(/\s+/g, " ").trim();
}

function formatDate(dateStr) {
  if (!dateStr) return "";
  const parsed = new Date(`${dateStr}T00:00:00Z`);
  if (isNaN(parsed.getTime())) return dateStr;
  return parsed.toLocaleDateString("en-US", { year: "numeric", month: "short", day: "numeric", timeZone: "UTC" });
}

function toPublicObjectUrl(key) {
  return `https://mixes.aklein.studio/${key.split("/").map(s => encodeURIComponent(s)).join("/")}`;
}

function pickAudioObject(objects) {
  const audioExtensions = [".mp3", ".m4a", ".aac", ".wav", ".ogg", ".flac"];
  const preferredNames = ["mix", "main", "session", "archive", "audio"];
  const candidates = objects.filter(obj => audioExtensions.some(ext => obj.key.toLowerCase().endsWith(ext)));
  if (!candidates.length) return null;
  const scored = candidates.map(obj => {
    const base = obj.key.toLowerCase().split("/").pop() || "";
    let score = 0;
    preferredNames.forEach((name, index) => { if (base.includes(name)) score += 100 - index * 10; });
    if (base.endsWith(".mp3")) score += 20;
    if (typeof obj.size === "number") score += Math.min(obj.size / 1000000, 50);
    return { obj, score };
  }).sort((a, b) => b.score - a.score);
  return scored[0].obj;
}

function pickCoverObject(objects) {
  const preferredOrder = ["cover.jpg", "cover.jpeg", "cover.png", "artwork.jpg", "artwork.jpeg", "artwork.png", "thumb.jpg", "thumb.jpeg", "thumb.png"];
  const imageCandidates = objects.filter(obj => /\.(jpg|jpeg|png|webp|avif)$/i.test(obj.key));
  if (!imageCandidates.length) return null;
  const exactMatch = preferredOrder.find(name => imageCandidates.some(obj => obj.key.toLowerCase().endsWith(`/${name}`)));
  if (exactMatch) return imageCandidates.find(obj => obj.key.toLowerCase().endsWith(`/${exactMatch}`)) || null;
  return imageCandidates[0];
}

async function getMixes(env) {
  const listed = await env.MIXES_BUCKET.list();
  const objects = listed.objects || [];
  const folders = new Set();
  for (const obj of objects) {
    const parts = obj.key.split("/");
    if (parts.length >= 2 && parts[0]) folders.add(parts[0]);
  }
  const mixes = [];
  for (const folder of folders) {
    const { date, title } = parseSlug(folder);
    const folderObjects = objects.filter(obj => obj.key.startsWith(`${folder}/`));
    const audioObject = pickAudioObject(folderObjects);
    if (!audioObject) continue;
    const coverObject = pickCoverObject(folderObjects);
    mixes.push({
      slug: folder, title, date, displayDate: formatDate(date),
      file: toPublicObjectUrl(audioObject.key),
      cover: coverObject ? toPublicObjectUrl(coverObject.key) : null,
      soundcloud: "https://soundcloud.com/aksf"
    });
  }
  mixes.sort((a, b) => (!a.date ? 1 : !b.date ? -1 : b.date.localeCompare(a.date)));
  return mixes;
}

export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    if (request.method === "OPTIONS") return new Response(null, { status: 204, headers: CORS_HEADERS });
    if (url.pathname === "/api/mixes.json") {
      try {
        const mixes = await getMixes(env);
        return jsonResponse({ ok: true, updatedAt: new Date().toISOString(), count: mixes.length, mixes });
      } catch (error) {
        return jsonResponse({ ok: false, error: String(error.message || error), mixes: [] }, 500);
      }
    }
    if (url.pathname === "/api/health") {
      return jsonResponse({ ok: true, service: "aklein-mixes-api", timestamp: new Date().toISOString() }, 200, { "cache-control": "no-store" });
    }
    return new Response("Not found", { status: 404 });
  }
};
```

The client side `mixes.js` script fetches the API and renders the card grid. Drop it in `assets/js/` and reference it from `mixes.html`. The API URL is the only thing to update for your own deployment.

```javascript
const MIXES_API = "https://aklein-mixes-api.YOUR-ACCOUNT.workers.dev/api/mixes.json";

function formatDate(dateStr) {
  if (!dateStr) return "";
  const parsed = new Date(`${dateStr}T00:00:00Z`);
  if (isNaN(parsed.getTime())) return dateStr;
  return parsed.toLocaleDateString("en-US", { year: "numeric", month: "short", day: "numeric", timeZone: "UTC" });
}

function renderMixCard(mix) {
  const date = mix.displayDate || formatDate(mix.date) || "";
  return `
    <article class="mix-card">
      <div class="mix-card__cover">
        <img class="mix-card__cover-img" src="${mix.cover}" alt="${mix.title} cover art"
          loading="lazy" onerror="this.src='${mix.coverPng}';this.onerror=null">
        <div class="mix-card__overlay">
          <p class="mix-card__label">TONE // SESSION</p>
          <h3 class="mix-card__title">${mix.title}</h3>
          <p class="mix-card__date">${date}</p>
        </div>
      </div>
      <div class="mix-card__player">
        <audio class="mix-card__audio" controls preload="none" aria-label="Play ${mix.title}">
          <source src="${mix.file}" type="audio/mpeg">
        </audio>
        <div class="mix-card__actions">
          <a class="mix-card__action" href="${mix.file}" download>DOWNLOAD</a>
          <a class="mix-card__action" href="https://soundcloud.com/aksf" target="_blank" rel="noopener noreferrer">SOUNDCLOUD</a>
        </div>
      </div>
    </article>`;
}

async function loadMixes() {
  const container = document.getElementById("mix-grid-container");
  if (!container) return;
  try {
    const response = await fetch(MIXES_API);
    if (!response.ok) throw new Error(`API returned ${response.status}`);
    const data = await response.json();
    const mixes = data.mixes || [];
    if (mixes.length === 0) {
      container.innerHTML = `<div class="mix-empty"><p class="mix-empty__eyebrow">ARCHIVE</p><p class="mix-empty__title">SESSIONS INCOMING.</p></div>`;
      return;
    }
    const grid = document.createElement("div");
    grid.className = "mix-grid";
    grid.innerHTML = mixes.map(renderMixCard).join("");
    container.innerHTML = "";
    container.appendChild(grid);
  } catch (error) {
    container.innerHTML = `<div class="mix-error"><p class="mix-error__label">SIGNAL ERROR</p><p class="mix-error__message">Could not load the mix archive. ${error.message}</p></div>`;
  }
}

document.querySelectorAll("[data-current-year]").forEach(el => { el.textContent = new Date().getFullYear(); });
if (document.readyState === "loading") { document.addEventListener("DOMContentLoaded", loadMixes); }
else { loadMixes(); }
```

## Schema and SEO

Getting the structured data right was worth spending time on. The site now has a full `@graph` in the JSON-LD covering `Person`, `MusicGroup`, `WebSite`, `BreadcrumbList`, and `RadioStation`. The BreadcrumbList is what triggered the green checkmarks in Google's Rich Results Test. The RadioStation type mapping to a Local businesses rich result was a bonus from having the Las Vegas address attached to the Person entity.

All three original pages plus Wire and Mixes have their own page-specific schema blocks with `ProfilePage`, `CollectionPage`, and page-level BreadcrumbList entries. Canonical URLs use clean paths without `.html` extensions matching what Cloudflare Pages serves.

## Configuring Workers in the Cloudflare Dashboard

Each Worker gets configured through the Cloudflare dashboard under Workers and Pages. The tabs that matter are Bindings, Settings, and Domains.

**Bindings** is where you connect external resources to the Worker. For `aklein-wire-feed-api` you add one KV namespace binding with the variable name `WIRE_FEED` pointing at the KV namespace you created. For `aklein-mixes-api` you add one R2 bucket binding with the variable name `MIXES_BUCKET` pointing at the `aklein-studio-mixes` bucket. Without these bindings the Workers throw errors at runtime because `env.WIRE_FEED` and `env.MIXES_BUCKET` would be undefined.

**Trigger events** under Settings is where the wire feed cron lives. Click Add, select Cron Trigger, and enter `0 */6 * * *` to fire every six hours. The mixes Worker has no cron since it reads R2 live on every request.

**Domains** is where you connect a custom subdomain. For the mixes API you can optionally add a route here pointing `mixes-api.aklein.studio` at the Worker instead of using the raw `workers.dev` URL. The R2 bucket custom domain is separate and lives in the bucket settings, not the Worker settings.

**Deploying via the dashboard editor** works for quick iterations but it does not scale well once you have more than one Worker. The better path is connecting each Worker to a GitHub repository via the Wrangler CLI and a `wrangler.toml` config file. When that is set up a push to the main branch triggers an automatic Worker deployment the same way a push to your Pages repo triggers a site deployment. The `wrangler.toml` for the mixes Worker looks like this:

```toml
name = "aklein-mixes-api"
main = "worker.js"
compatibility_date = "2024-01-01"

[[r2_buckets]]
binding = "MIXES_BUCKET"
bucket_name = "aklein-studio-mixes"
```

And for the wire feed Worker:

```toml
name = "aklein-wire-feed-api"
main = "worker.js"
compatibility_date = "2024-01-01"

[[kv_namespaces]]
binding = "WIRE_FEED"
id = "YOUR_KV_NAMESPACE_ID"

[triggers]
crons = ["0 */6 * * *"]
```

The KV namespace ID comes from the KV section of the Cloudflare dashboard. Once these configs are committed alongside the Worker source code, deployment becomes a single `wrangler deploy` command or an automatic GitHub Actions push. That is the next step for this setup — move both Workers off manual dashboard edits and into proper GitOps repos.

## The Infrastructure Stack

The full picture of what is running behind aklein.studio is worth documenting in one place.

Cloudflare Pages hosts the static site files with automatic deployment from the Git repository. Two Cloudflare Workers handle the dynamic layers: one for the wire feed with KV caching and cron refresh, one for the mix archive reading directly from R2. The R2 bucket stores all the MP3 files and cover art and serves them through the custom domain. The AzuraCast instance running on the homelab handles the live radio stream and metadata API. Cloudflare proxies all of it and handles SSL everywhere.

None of this required a server. No nginx, no VPS for the site itself, no monthly compute bill. The only infrastructure cost is the R2 storage which sits well within the free tier at current archive size.

## What Google Sites Could Not Do

A persistent audio player. Custom fonts with full control over weight and tracking. A dark background that actually renders correctly. Structured data and schema markup. A live RSS wire feed with KV caching. A self-hosted mix archive served from object storage. A sticky nav with dropdown menus and mobile panel animations. Rich Results passing in Google's test tool. An alternating accent word pattern in the hero carrying across every page. Workers running under your own domain instead of a generic subdomain.

None of that was possible on Google Sites. All of it is live now at [aklein.studio](https://aklein.studio).

The decks are warm and the signal is clean. 🎧
