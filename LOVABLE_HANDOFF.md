# Nexus Solution Proposal — Lovable / Supabase handoff

This bundle contains a finished proposal app. Read this first — it explains the **two ways**
to run it in Lovable and what's already done, so you spend the fewest credits.

> **Domain:** you are connecting `nexussolution.cloud` **yourself** in Lovable's settings.
> Nothing in this doc or the prompts asks the AI to touch domains.

---

## What's in this zip
- **`nexus-proposal.html`** — the finished app, **fully self-contained** (all images/branding
  inlined as data URIs, no external files). Use this for Path A (host as-is).
- **`index.html`** — the same app but with assets referenced as real files (more readable;
  better starting point if you port to React, Path B). Its assets sit next to it at the zip
  root, so it runs directly.
- **`nexus-favicon.svg`** + **`kca-assets/`** (`guarantee_icons.webp`, `king-icon.webp`,
  `client-logo.png`) — the raw image assets used by `index.html`.
- **`LOVABLE_HANDOFF.md`** — this file.

---

## Pick your path

### Path A — Host the static file as-is (+ Supabase)   ← safest, nothing breaks
Serve `nexus-proposal.html` as the site and only wire Supabase (below).
- ✅ Pixel-identical to what was tested; zero risk of regressions; cheapest.
- ✅ All per-client editing (niche, client, city, link, logo, **all prices**, services,
  Winning-Formula copy, payment link) already works **through the in-app setup form** — no
  code edits needed for day-to-day use.
- ⚠️ Deeper *structural/design* changes later are harder for Lovable's AI (it's one big file).

### Path B — Port to React (Lovable-native)
Have Lovable rebuild the page as React/Tailwind components.
- ✅ Future structural/design edits via Lovable's AI become much easier and more solid.
- ⚠️ This is a real migration of ~556 KB of bespoke CSS + a custom templating/“niche-sweep”
  engine + the gate, pricing wiring, save/share and client-view logic. It **will not be 1:1**
  unless QA'd carefully — highest risk of "breaking what we have", and the most credits.
- 👉 If you choose this, port **section by section**, keep `nexus-proposal.html` as the visual
  reference, and verify each piece (templating, pricing, gate, client view) still behaves.

**Recommendation:** start on **Path A** (live immediately, nothing breaks). Move to Path B only
when you actually hit an edit the form can't do — and do it deliberately, not as a big-bang.

Either way, the data layer (Supabase) is identical — do this:

---

## Supabase (same for both paths)

Run in the Supabase SQL editor:
```sql
create table proposals (
  id text primary key,
  config jsonb not null,
  created_at timestamptz default now()
);
alter table proposals enable row level security;
create policy "public read"   on proposals for select using (true);
create policy "public insert"  on proposals for insert with check (true);
```

The whole demo (niche, client, city, link, logo, every price, selected services,
Winning-Formula copy, payment link) is one `config` object — so the table never changes.

### Path A wiring (vanilla)
Add in `<head>`:
```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
```
Find `// NexusStore` in the file and replace the stub with:
```js
window.NEXUS_SUPABASE = { url:'https://YOUR-PROJECT.supabase.co', anonKey:'YOUR-ANON-KEY' };
var _sb = window.supabase.createClient(NEXUS_SUPABASE.url, NEXUS_SUPABASE.anonKey);
window.NexusStore = {
  save: function(cfg){
    var id = (crypto.randomUUID ? crypto.randomUUID() : String(Date.now()));
    return _sb.from('proposals').insert({ id:id, config:cfg })
      .then(function(){ return location.origin + location.pathname + '?c=' + id; });
  },
  load: function(){
    var m = location.search.match(/[?&]c=([^&]+)/); if(!m) return null;
    return _sb.from('proposals').select('config').eq('id', decodeURIComponent(m[1]))
      .single().then(function(r){ return r.data ? r.data.config : null; });
  },
  list: function(){
    return _sb.from('proposals').select('id, config, created_at').order('created_at',{ascending:false}).limit(60)
      .then(function(r){ return (r.data||[]).map(function(row){
        return { client:(row.config||{}).client, niche:(row.config||{}).niche,
                 link: location.origin+location.pathname+'?c='+row.id, ts: row.created_at }; }); });
  }
};
```
Then make `init()` await the (now async) `load()` before choosing agent-vs-client view (~3 lines):
the client-view branch should do `Promise.resolve(NexusStore.load()).then(cfg => …)`.

### Path B wiring (React)
Same three operations as a small `nexusStore.ts` using `@supabase/supabase-js`; call `save` on
"Presentation complete", `load` when `?c=<id>` is present (server/client component), and `list`
for the saved-clients screen.

---

## How the app already works (so you don't rebuild logic you don't need to)
- **Setup gate** (multi-step): niche, client, city, link, logo, payment link, **all prices**
  (setup, monthly, reveal offer, 5 stacked/scratched prices, 3 service prices), and
  **Winning-Formula copy** (section title/subtitle, 3 levers, sitemap page names).
- **Templating engine** rewrites the whole page per niche/client/city + the architecture modal.
- **"Presentation complete"** saves the demo (incl. services toggled live) and returns a
  client link. Opening `?c=<id>` = **read-only client view**: no edit/add-services/price-pencil
  controls; the edit button is replaced by an **"Activate My Engine"** button → the payment link.
- **Saved-clients** button lists past demos (via `NexusStore.list`).

---

## ▶ Recommended prompt — let Lovable choose the route (paste this)

> I'm uploading a finished proposal web-app as a zip. Inside: `nexus-proposal.html` (the complete
> app, fully self-contained — plain HTML/CSS/vanilla JS, all images/branding inlined, no external
> files); `index.html` (same app with assets as real files, more readable); and this
> `LOVABLE_HANDOFF.md` (Supabase data model + exact save/load/list code).
>
> Context: this is a live sales tool I'll keep editing inside Lovable going forward. I want the
> most solid, maintainable result, but it must NOT lose or break any current behaviour, design,
> copy, pricing, or branding.
>
> Your task: evaluate the codebase and CHOOSE the best way to run it here — either port it to your
> native React + Tailwind structure (better if I'll do ongoing structural edits in Lovable) OR
> host it as-is as a static site (lowest risk). Recommend the approach with a one-paragraph reason,
> then implement it. I trust your call — just preserve everything below.
>
> Must keep working exactly as today, whichever route you pick:
> - The multi-step "Set up this demo" gate: niche, client name, city, website link, logo upload,
>   payment/activation link, and ALL editable prices (one-time setup, monthly, reveal offer, the 5
>   stacked/scratched values, the 3 add-on service prices), plus the Winning-Formula copy (section
>   title/subtitle, the 3 levers, the sitemap page names).
> - The templating engine that rewrites the whole page AND the architecture modal per
>   niche/client/city, with sensible defaults when fields are blank.
> - "Presentation complete" → saves the demo (incl. add-on services toggled during the demo) and
>   produces a shareable client link.
> - Opening a saved link = read-only client view: setup gate, edit button, "+ Add services" button
>   and price-edit pencils all hidden; edit button replaced by an "Activate My Engine" button that
>   points to the payment link.
> - The "Saved clients" list of past demos.
> - Mobile/tablet responsiveness (the "Your Investment" pricing tiers must not clip).
>
> Data layer (regardless of route): create a Supabase table `proposals (id text primary key,
> config jsonb not null, created_at timestamptz default now())` with public read + insert RLS, and
> implement the app's `NexusStore` save/load/list against it exactly as documented below (each
> demo's full settings live in one `config` JSON; client link is `?c=<id>`; make the initial load
> await the async fetch before choosing agent-vs-client view).
>
> Constraints: don't restyle, rewrite copy, or change prices/branding — the design is final; the
> Nexus Solution assets are already included. If you port to React, keep it pixel-for-pixel, work
> section by section using `nexus-proposal.html` as the visual reference, and verify templating,
> pricing, gate, client view and saved-clients after each section. I'll connect the custom domain
> myself — don't configure domains.

---

## Alternate prompt — Path A (force static host only)

> "I'm uploading a finished, self-contained static site, `nexus-proposal.html` (plain
> HTML/CSS/vanilla JS, all assets inlined). Host it as the site exactly as-is — do not change
> markup, CSS, copy, pricing, or logic. Then connect Supabase: create a `proposals` table
> (`id text pk, config jsonb, created_at timestamptz`) with public read+insert RLS, add the
> supabase-js CDN script in <head>, and replace the `window.NexusStore` stub with the Supabase
> save/load/list shown in the file's comments / LOVABLE_HANDOFF.md, making `init()` await
> `NexusStore.load()`. Don't change anything else. I'll handle the domain myself."

## Suggested prompt — Path B (only if you chose the React port)

> "Port this static proposal (`index.html` + the assets in /assets) into a React + Tailwind app,
> keeping the design pixel-for-pixel. Preserve ALL behaviour: the multi-step setup gate, the
> per-niche templating of the whole page + architecture modal, every editable price and service,
> the Winning-Formula copy overrides, 'Presentation complete' save, the read-only client view
> (no edit/add-services controls, payment-link button), and the saved-clients list. Use Supabase
> for save/load/list against a `proposals` table (`id, config jsonb, created_at`). Work section
> by section and keep nexus-proposal.html as the visual reference. I'll handle the domain myself."
