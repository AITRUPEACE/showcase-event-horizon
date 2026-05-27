# Event Horizon

> A map-first product for vendor-dense events. Organizers publish a live event map; attendees open one link on their phone — no app install — and find vendors, food, stages, and amenities.

🔗 **Live:** [event-horizon.to](https://event-horizon.to)
📝 **Case study:** [event-horizon.to/build](https://event-horizon.to/build) — first post live, more drafting

> This is a **showcase repository**. The product is closed-source. What's here: an overview, the architecture, the stack, and the implementation details I'm most proud of (and a few I had to rip up and rebuild).

---

## Screenshots

![Map view](./screenshots/map.png)
![Editor](./screenshots/editor.png)
![Mobile drawer](./screenshots/mobile.png)

> Placeholder image refs. Drop PNGs into `./screenshots/` with these names.

---

## The product loop

```
Organizer creates and publishes a map
  → shares one event link
  → attendee opens it on phone
  → attendee finds places and uses the map
  → attendee engages (comments, photos, favorites, live activity)
  → organizer sees activity and analytics
```

The MVP is deliberately narrow: organizer-bought, attendee-facing, no-download. Vendors are organizer-managed POIs in this phase; vendor self-service is a later layer on top of the same data model.

---

## Stack

| Layer | Choice |
| --- | --- |
| Framework | Next.js 15 (App Router) on Vercel |
| UI | React 19, Tailwind v4, shadcn/Radix, Framer Motion |
| Map | Leaflet + React Leaflet, `@geoman-io/leaflet-geoman-free` for org-side drawing, Turf for geometry |
| State | Zustand (+ `zundo` for undo/redo in the editor) |
| Data | Supabase Postgres, Supabase Auth, Supabase Realtime, RLS-first |
| AI | OpenAI for the org-side co-pilot (schema-aware tool-calling on top of Supabase) |
| Email | Resend (org notifications, daily schedule digest) |
| PWA | Serwist service worker; install-to-home-screen attendee path |
| Testing | Jest + Testing Library, Playwright for e2e |

---

## Architecture at a glance

- **Single Postgres schema (`event`)** owns the world: events, POIs, zones, vendors, performers, schedule items. RLS rules separate organizer reads/writes from attendee reads.
- **Two discriminators on `pois`**: `entity_type` (coarse, drives which feature modules apply) and `category` (fine, drives icon and visual treatment). Same row; two axes. This took three refactors to get right — see the case study.
- **JSONB with a rule.** Anything rendered but never queried lives in `pois.data`. Anything filtered, indexed, sorted, or joined on earns its own column. Sibling tables (`vendors`, `performers`, `schedule_items`) exist only for entities that have their own lifecycle (own auth, own RLS, own Stripe attachment).
- **Realtime that doesn't set the DB on fire.** Broadcast channels for ephemeral live activity, DB subscriptions only for state the client must trust, a freshness budget on the rest. Details in case-study story 03 (drafting).
- **Server Actions over API routes** for org-side mutations; route handlers reserved for webhooks, OG images, and the AI co-pilot stream.
- **PDF export for organizers** built with `pdf-lib` directly from the same POI geometry that renders on the web map.

---

## Implementation details worth a look

### POI schema, v0 → v3
The longest-running design problem in the codebase. v0 was a single `pois` table with a `poi_type` enum and a fat `metadata` JSONB column. v1 tried to fix the blob by splitting it into `vendor_data`, `stage_data`, `attraction_data`, `amenity_data` — which just moved the type soup one layer up. v2 collapsed them back into one `data` column with a discipline (render-only payload), reversed the vendor relationship (one vendor → many POIs via `pois.vendor_id`), and added a check constraint to keep the application code from having to remember it. v3 split the discriminator into two axes once the feature modules grew past the categories.

Full walkthrough with ERDs and the actual migrations: [event-horizon.to/build](https://event-horizon.to/build)

### Bending Leaflet to the product
React Leaflet is great until you want z-index between marker layers, polygon zones the user can draw in-app, SSR safety with a library that reaches for `window` at import time, and pin grouping that doesn't fall apart at festival scale. Story 02 of the case study walks through what worked.

### AI co-pilot in the org loop
Same MCP/agent pipeline pattern I built at Property Control, applied to my own product. The model gets read access to the event's schema and a narrow tool set for proposing edits; humans accept/reject. Story 04 of the case study.

### Migrations that are safe to re-run
Every block in non-trivial migrations is wrapped in `DO $$ BEGIN IF EXISTS ... END $$;` so a half-applied state recovers cleanly. Big refactors snapshot the affected tables into a `backup.*` schema at the top of the file. The rollback plan is part of the migration, not a separate document.

---

## Project status

- **MVP live.** Map, editor, attendee engagement, organizer analytics, live broadcasts.
- **Case study in progress.** Story 01 (schema) live, stories 02–04 drafting.
- **Pilot-stage commercials.** Organizer-, operator-, and sponsor-funded pilots before any vendor-marketplace plays.

---

## About

Built solo by **Cristian Tohatan** — senior full-stack (C#/.NET + React/TypeScript), recent work in AI-augmented engineering.

- 🌐 [cristian-tohatan.com](https://cristian-tohatan.com)
- 💼 [LinkedIn](https://linkedin.com/in/cristian-tohatan)

Source is closed. Happy to walk through the codebase live for serious conversations — reach out via the portfolio.
