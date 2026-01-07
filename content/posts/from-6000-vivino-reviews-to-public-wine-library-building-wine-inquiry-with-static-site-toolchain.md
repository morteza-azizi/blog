+++
date = '2026-01-08'
draft = false
title = 'From 6,000+ Vivino Reviews to a Public Wine Library: Building Wine Inquiry with a Static Site + a Toolchain'
tags = ['wine', 'static-site', 'hugo', 'software-architecture', 'pipeline', 'content-platform', 'data-migration']
+++

# From 6,000+ Vivino Reviews to a Public Wine Library: Building Wine Inquiry with a Static Site + a Toolchain

For the last ~15 years, wine has been one of my main passions. I've tasted a lot, learned a lot, and—most importantly—written down what I actually thought while drinking.

Over time, those notes grew into ~6,000 reviews inside Vivino.

And that's where a problem started:

My reviews were useful, but they were trapped in an app.

Browsing them the way I wanted (by region, grape, producer, my verdict) wasn't great.

Sharing a clean, consistent page with my notes and rating wasn't great either.

And I didn't want years of work to live behind someone else's product decisions.

So I started building Wine Inquiry: a public, browsable wine library based on my personal tasting notes.

[https://www.wineinquiry.com/](https://www.wineinquiry.com/)

This post is the story of how it started, how I chose the approach, and how I built a toolchain that makes publishing sustainable—without turning it into a giant frontend project.

## The real goal was not "a website"

A website is easy.

The real goal was:

**Turn a private archive of messy, real-life data into a public resource I can maintain long-term.**

That required a workflow I could repeat in small batches—because migrating 6,000 items in one go is how projects die.

My preferred working style:

- process 20–30 reviews at a time
- publish continuously
- improve the site iteratively

## Constraints that shaped everything

I approached this like any architecture decision: define constraints first.

### 1) SEO matters

If I want people (and future YouTube viewers) to discover the content, the site needs:

- real URLs
- real HTML
- content Google can index reliably

A pure client-side app that loads from an API can work, but it introduces extra moving parts and SEO complexity I didn't want.

### 2) Low operations overhead

This isn't a startup with a 24/7 ops team. It's a passion project that should stay cheap to run and easy to deploy.

### 3) I didn't want a "frontend career change"

I'm a software architect with strong backend roots (Go/C#). I wanted to build something clean, stable, and maintainable—without signing up for endless UI framework churn.

### 4) Images were messy

My wine label photos lived in Google Drive with filenames that didn't map to wines.
So I needed a way to reliably link:

wine review → correct label image

without manual renaming hell

## The solution: Static publishing + a toolchain

The winning pattern for me was:

- **Static site** for publishing (fast, SEO-friendly, simple hosting)
- **Structured pipeline** for ingestion/enrichment (repeatable, incremental)

Hugo became the publishing engine, and I built a set of tools around it.

### High-level architecture

```mermaid
flowchart TB
  CSV[Vivino export (CSV)] --> IMP[Importer\nnormalize + map fields]
  IMP --> LEDGER[(JSON Ledger\nprogress + mappings)]
  LEDGER --> ENR[Enricher UI\nassign label images]
  ENR --> LEDGER
  LEDGER --> ORG[Organizer\nfetch/resize/publish images]
  ORG --> LEDGER
  LEDGER --> EXP[Exporter\ngenerate Hugo content]
  EXP --> HUGO[Hugo build]
  HUGO --> HOST[Static hosting / CDN\nwineinquiry.com]
```

This architecture is intentionally boring:

- no database required for the public site
- no backend required to render tasting notes
- but enough structure that publishing doesn't become painful

## The "JSON ledger" idea

Markdown is great for publishing.
Markdown is not great as a "workflow database."

So I keep a growing ledger.json that tracks:

- what's imported
- what still needs images
- what's ready to publish
- what image URL belongs to what wine page
- (later) YouTube video IDs

This lets me process batches of 20–30 reviews and always know what's done.

The key rule:

**Markdown is the content**

**JSON is the workflow + mapping**

That prevents "two sources of truth" from drifting.

## The toolchain I built

I ended up with a small pipeline of tools that match my workflow:

### 1) Importer (Vivino CSV → structured data)

- reads Vivino export CSV
- normalizes fields
- generates stable IDs/slugs
- writes records into the JSON ledger
- produces (or updates) Markdown for Hugo

### 2) Enricher UI (the image linking problem)

This is the "unsexy but essential" part.

Because my photos weren't named in a useful way, I built a UI that lets me:

- search/select a wine review
- paste/select the correct Google Drive image
- save that association into the ledger

This turned a nightmare ("rename thousands of images") into a manageable workflow.

### 3) Organizer (make images public and consistent)

This tool:

- downloads the selected images
- resizes/converts them into consistent formats
- republishes them to a public location
- updates the ledger with stable public URLs

This was the point where I hit one of the biggest real-world issues...

## What broke in production (and what I learned)

Building is easy. Shipping reveals reality.

### Issue 1: Google Drive auth walls

At first, label images sometimes redirected visitors to Google login.
That makes your site look broken instantly.

**Fix:** the organizer had to publish images with the right permissions and stable URLs.

**Lesson:**
If it's public content, treat images like a real asset pipeline, not a "personal file share."

Long-term I'll likely move images to a proper object store/CDN (S3/R2 style), but the immediate fix was: no auth walls.

### Issue 2: Tiny data formatting bugs destroy trust

I hit bugs like:

- duplicated producer names
- vintages rendered twice ("2022 (2022)")
- broken category URLs due to slug handling ("Rosé" → encoding issues)

**Lesson:**
The exporter must enforce display rules.
You cannot rely on "data is mostly consistent."

### Issue 3: UX bugs matter more than fancy features

Example: showing "No wines found" while wines were visible.

**Lesson:**
One visible glitch costs more credibility than ten missing features.

## How I think about the site now

Wine Inquiry is not "a blog."

It's a reference library:

- browse by type (red/white/sparkling/rosé)
- filter by country/region/grape
- see my rating, verdict, and notes
- later: connect the same page to a YouTube video

The long-term play is:

- **the website** becomes the archive and discovery engine (SEO)
- **YouTube** becomes distribution and personality
- **the toolchain** keeps publishing sustainable

## What's next

The next steps are clear:

### YouTube channel

- every video maps to one or more wine pages
- wine pages embed the video (and become the canonical reference)

### Interactive layer (without rewriting the site)

- keep the public pages static
- add an optional overlay later for comments/favorites
- a small API can power that, without turning everything into a SPA

### Build an owned audience

- email list on the site (weekly recommendations + deep dives)
- social platforms become distribution, not dependency

## Why this project matters to me

This is where my two passions meet:

Wine taught me patience, sensory attention, and the value of detailed notes over time.

Software architecture taught me systems thinking: constraints, trade-offs, and designing workflows that scale.

Wine Inquiry is my attempt to build something:

- useful
- honest
- and sustainable

If you're building something similar—turning personal data into a public resource—my biggest advice is:

**Don't just build the website. Build the pipeline.**

Wine Inquiry: [https://www.wineinquiry.com/](https://www.wineinquiry.com/)
