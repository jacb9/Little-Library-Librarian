# Little Library Librarian — Greater Victoria

A progressive web app for cataloguing and searching little free libraries across Greater Victoria.

## How it works

The map shows every known little free library location as a **ghost pin** (grey) — these are
places we know exist, but no one has catalogued yet. Once someone scans a shelf, the pin turns
**green** and stays that way, with the photo and AI-read inventory visible to everyone.

A global search lets anyone look up a book title or author and see every library where it's
currently been spotted on a shelf.

## What's in this repo

```
index.html        — The full PWA (single file)
libraries.json     — 1,081 known library locations (name, coordinates, category, box photo)
manifest.json      — PWA install manifest
sw.js              — Service worker (offline support)
icons/             — App icons
```

## Quick start (GitHub Pages)

1. Push this folder to a GitHub repo
2. Settings → Pages → Source: `main` branch, `/ (root)`
3. Live at `https://yourusername.github.io/your-repo-name`

The map works immediately, showing all 1,081 locations as ghost pins. Scanning, search, and
photo upload need Supabase + an AI extraction endpoint (below).

## Set up Supabase

### 1. Create a project

https://supabase.com → new project. Note your **Project URL** and **anon key**.

### 2. Create tables

```sql
-- Inventory scans: one row per shelf photo + AI-read (and user-corrected) title list
create table inventories (
  id               bigserial primary key,
  library_id       bigint not null,
  photo_url        text not null,
  titles           text[] not null default '{}',
  ai_total_count   int default 0,   -- how many titles the AI originally returned
  ai_edited_count  int default 0,   -- how many the user edited or removed/added
  created_at       timestamptz default now()
);
create index on inventories (library_id, created_at desc);

-- Box exterior photos (separate from inventory photos)
create table box_photos (
  id          bigserial primary key,
  library_id  bigint not null,
  photo_url   text not null,
  created_at  timestamptz default now()
);
create index on box_photos (library_id, created_at desc);

-- User-added libraries (not in the original 1,081)
create table libraries_user_added (
  id          bigserial primary key,
  name        text not null,
  category    text default 'Little Free Libraries',
  lat         double precision not null,
  lng         double precision not null,
  created_at  timestamptz default now()
);
```

For better full-text/fuzzy search at scale, also enable the trigram extension and add a GIN index:

```sql
create extension if not exists pg_trgm;
create index on inventories using gin (titles);
```

(The app currently does fuzzy matching client-side in JavaScript, which is fine under a few
thousand inventory rows. Move matching into a Postgres function using `pg_trgm` once the dataset
grows past that.)

### 3. Create storage bucket

Storage → New bucket → name `lll-photos`, set **Public**.

```sql
create policy "Public read" on storage.objects
  for select using (bucket_id = 'lll-photos');
create policy "Public upload" on storage.objects
  for insert with check (bucket_id = 'lll-photos');
```

### 4. Configure the app

In `index.html`:

```javascript
const SUPABASE_URL = 'https://xxxx.supabase.co';
const SUPABASE_ANON_KEY = 'your-anon-key';
const SUPABASE_ENABLED = true;
```

## Set up AI shelf-reading

The app calls `EXTRACTION_ENDPOINT` with `{ image: base64string }` and expects back
`{ titles: ["Book Title 1", "Book Title 2", ...] }`.

The simplest way to host this is a Supabase Edge Function that calls the Anthropic API with
vision. Example function (`supabase/functions/extract-titles/index.ts`):

```typescript
import Anthropic from "npm:@anthropic-ai/sdk";

const anthropic = new Anthropic({ apiKey: Deno.env.get("ANTHROPIC_API_KEY") });

Deno.serve(async (req) => {
  const { image } = await req.json();

  const msg = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    messages: [{
      role: "user",
      content: [
        { type: "image", source: { type: "base64", media_type: "image/jpeg", data: image } },
        { type: "text", text: "List every book title you can read on the spines in this photo. Return ONLY a JSON array of strings, nothing else. If you can't read a spine clearly, skip it rather than guessing." }
      ]
    }]
  });

  const text = msg.content.find(b => b.type === "text")?.text || "[]";
  let titles = [];
  try {
    titles = JSON.parse(text.replace(/```json|```/g, "").trim());
  } catch {
    titles = [];
  }

  return new Response(JSON.stringify({ titles }), {
    headers: { "Content-Type": "application/json" }
  });
});
```

Deploy with `supabase functions deploy extract-titles`, set the `ANTHROPIC_API_KEY` secret,
then point the app at it:

```javascript
const EXTRACTION_ENDPOINT = 'https://xxxx.supabase.co/functions/v1/extract-titles';
const EXTRACTION_ENABLED = true;
```

### Accuracy tracking

Every saved inventory stores `ai_total_count` and `ai_edited_count`. Query the ratio over time
to see how much correction each scan needs in practice — this is the number that tells you
whether the AI read is actually pulling its weight:

```sql
select
  avg(ai_edited_count::float / nullif(ai_total_count, 0)) as avg_edit_rate,
  count(*) as total_scans
from inventories;
```

## What's deliberately not built yet

- User accounts / attribution
- Library claiming by owners
- "Library is empty" one-tap signal
- Author/publisher seeding campaigns
- Affiliate links from titles to retailers
- Server-side fuzzy search (pg_trgm) — current matching is client-side JS, fine at small scale

These are straightforward additions once the core scan-and-search loop is proven out.
