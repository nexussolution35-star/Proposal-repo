# Nexus Solution Proposal — Lovable / Supabase handoff

This is a single self-contained `index.html` proposal app. Everything client-facing is
**already built and working with no backend** — the goal in Lovable is only to (1) swap the
storage stub for Supabase so you get short links + a saved-clients dashboard, and (2) connect
the `nexussolution.cloud` domain. Keep the HTML; do **not** rebuild it in React.

---

## What already works here (no credits needed in Lovable)

- **Setup gate** (per-demo onboarding): niche, client name, city, website link, one-time
  reveal offer, logo upload. Persists to `localStorage`.
- **Dynamic templating**: niche/client/city/link/offer populate the whole page + the
  architecture modal, live-preview iframe, section links (`/section-ID`), pricing, etc.
- **"✓ Presentation complete" button** → saves the demo and shows a **sendable client link**.
- **Client view**: opening `…/?c=<token>` applies that client's saved config, **hides the
  setup gate and all agent buttons**, and shows the read-only proposal with their prices.
- With no backend, the config is encoded directly into the `?c=` link (works today).

The only limitation of the no-backend mode: an uploaded **logo** makes the link long, and
there's no saved-clients list. Supabase fixes both.

---

## The ONLY integration point (one cheap Lovable prompt)

In `index.html`, search for **`window.NexusStore`**. It looks like this:

```js
window.NEXUS_SUPABASE = { url:'', anonKey:'' };           // paste Supabase creds
window.NexusStore = {
  save: function(cfg){ return Promise.resolve(buildShareLink(cfg)); },  // -> shareable URL
  load: function(){ var s=getShareParam(); return s ? decodeCfg(s) : null; }
};
```

Replace it with the Supabase-backed version below (and add the CDN script in `<head>`):

```html
<!-- in <head> -->
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```

```js
window.NEXUS_SUPABASE = { url:'https://YOUR-PROJECT.supabase.co', anonKey:'YOUR-ANON-KEY' };
var _sb = window.supabase.createClient(NEXUS_SUPABASE.url, NEXUS_SUPABASE.anonKey);

window.NexusStore = {
  // Save the demo config, return a short shareable link
  save: function(cfg){
    var id = (crypto.randomUUID ? crypto.randomUUID() : String(Date.now()));
    return _sb.from('proposals').insert({ id: id, config: cfg }).then(function(){
      return location.origin + location.pathname + '?c=' + id;
    });
  },
  // If the URL has ?c=<id>, fetch that client's config (returns a Promise OR null)
  load: function(){
    var m = location.search.match(/[?&]c=([^&]+)/); if(!m) return null;
    return _sb.from('proposals').select('config').eq('id', decodeURIComponent(m[1]))
      .single().then(function(r){ return r.data ? r.data.config : null; });
  },
  // Saved-clients dashboard (the "📁 Saved clients" button)
  list: function(){
    return _sb.from('proposals').select('id, config, created_at').order('created_at',{ascending:false}).limit(60)
      .then(function(r){ return (r.data||[]).map(function(row){
        return { client:(row.config||{}).client, niche:(row.config||{}).niche,
                 link: location.origin+location.pathname+'?c='+row.id, ts: row.created_at }; }); });
  }
};
```

> Note: the app already `Promise.resolve()`s `NexusStore.load()`/`save()`, so returning a
> Promise from Supabase works without further changes. If `load()` returns a Promise, wrap the
> `init()` client-view branch in `Promise.resolve(fromStore).then(...)` — tell Lovable: *"make
> init() await NexusStore.load() before deciding agent vs client view."* (≈3 lines.)

---

## Supabase table (run in the SQL editor)

```sql
create table proposals (
  id text primary key,
  config jsonb not null,
  client text generated always as (config->>'client') stored,
  niche  text generated always as (config->>'niche')  stored,
  created_at timestamptz default now()
);

alter table proposals enable row level security;

-- Demo-friendly policies (anyone with the link can read; app can insert).
create policy "public read"  on proposals for select using (true);
create policy "public insert" on proposals for insert with check (true);
```

That's enough for: save on "Presentation complete", open by `?c=id`, and a dashboard
(`select id, client, niche, created_at from proposals order by created_at desc`).

---

## Optional: saved-clients dashboard (one more prompt)

> "Add an agent-only 'Saved clients' button next to '⚙ Edit demo' that lists rows from the
> Supabase `proposals` table (client, niche, date) with a Copy-link and Open button per row."

---

## Assets to bring into the Lovable project (public folder)

- `nexus-favicon.svg`
- `kca-assets/guarantee_icons.webp`
- `kca-assets/king-icon.webp`
- `kca-assets/client-logo.png`

(All other kca-assets were removed from the design.)

---

## Domain (`nexussolution.cloud`)

In Lovable → Project → **Settings → Domains** → add `nexussolution.cloud` (or a subdomain like
`proposal.nexussolution.cloud`). Lovable shows the DNS record (A/CNAME) to add at your domain
registrar; once it verifies, HTTPS is automatic.

---

## Suggested Lovable opening prompt (paste this)

> "I'm importing a finished single-file static site (`index.html`) — keep it as-is, don't
> convert it to React. Host it, then: (1) connect Supabase and create a `proposals` table
> (`id text pk, config jsonb, created_at`); (2) in `index.html` replace the `window.NexusStore`
> stub with Supabase `save`/`load` as documented in the file's comments and make `init()` await
> `NexusStore.load()`; (3) connect my domain `nexussolution.cloud`. Don't change any other
> markup, styles, copy, or pricing."
