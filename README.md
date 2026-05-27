# Event Horizon

> One QR code at the gate. Scan it, and every vendor, every stage, every washroom shows up on a searchable map in the attendee's phone browser. **No app, no download, no signup.**

🔗 **Live:** [event-horizon.to](https://event-horizon.to)
📝 **Case study:** [event-horizon.to/build](https://event-horizon.to/build) — story 01 (schema) live; stories 02–04 (Leaflet, realtime, AI co-pilot) drafting

> This is a **showcase repository**. The product is closed-source. What's here: overview, architecture, stack, and the implementation work I'd point a hiring manager or collaborator at.

---

## Screenshots

![Attendee map](./screenshots/map.png)
![Organizer editor](./screenshots/editor.png)
![Geofence-aware wayfinding](./screenshots/wayfinding.png)

> Placeholder image refs. Drop PNGs into `./screenshots/` with these names.

---

## The wedge

Built for **vendor-dense outdoor events**. Street festivals, food truck rallies, ribfests, markets, fairs — events where the floor plan *is* the product.

At a 50-vendor food festival, the average attendee finds maybe eight or ten booths before they call it a night. The rest of the vendors are two aisles over, unseen, through a crowd. That's a navigation problem, and a paper map at the entrance doesn't solve it. Event Horizon does:

- **One QR code at the gate** → live searchable map in the phone browser
- **Search** for *sliders*, filter to *open now*, tap a pin for hours and menu photos
- **Inside the perimeter**, the app takes over from Google Maps and shows live distance + cardinal direction ("40m northeast, past the main stage") — turn-by-turn is for driving *to* the event, not getting around *inside* it
- **Outside the perimeter**, Google Maps handles wayfinding to the gate

That perimeter handoff is the literal "event horizon" the product is named after.

---

## Stack

| Layer | Choice |
| --- | --- |
| Framework | Next.js 15 (App Router) on Vercel |
| UI | React 19, Tailwind v4, shadcn/Radix, Framer Motion, `vaul` mobile drawer |
| Map | Leaflet + React Leaflet, `@geoman-io/leaflet-geoman-free` (organizer drawing), Turf for in-polygon checks |
| State | Zustand (+ `zundo` for undo/redo in the editor), `@dnd-kit` for reordering |
| Data | Supabase Postgres + Auth + Realtime, RLS-first, single `event` schema |
| AI | OpenAI for the organizer-side co-pilot (schema-aware tool-calling) |
| Email | Resend (organizer notifications, daily schedule digest, opt-in capture) |
| Maps PDF | `pdf-lib` for the printable QR kit organizers hand to volunteers |
| PWA | Serwist |
| Testing | Jest + Testing Library, Playwright e2e |
| Deploy | Vercel; canonical domain `event-horizon.to` |

---

## Architecture at a glance

- **Single Postgres schema** (`event`) for events, POIs, zones, vendors, performers, schedule items. Org-scoped RLS via `event.organization_members`; attendee reads via published-state RLS.
- **Two discriminators on `pois`**: `entity_type` (coarse — decides which feature modules apply) and `category` (fine — drives icon and visual treatment). One row, two axes. Took three refactors to land — full walkthrough in the case study.
- **JSONB with a rule.** Anything rendered but never queried lives in `pois.data`. Anything filtered/indexed/sorted/joined earns its own column. Sibling tables (`vendors`, `performers`, `schedule_items`) only for entities with their own lifecycle.
- **Server Actions for organizer mutations.** API route handlers reserved for webhooks, OG images, the QR-kit PDF endpoint, and the AI co-pilot stream.
- **Anonymous engagement.** Comments, photos, favorites, and "live" activity work without an attendee account — privacy-safe `sessionId` hashing in `engagement_progress` keeps the auth-on-action wedge intact.
- **QR kit pipeline.** A single endpoint generates a print-ready PDF with per-zone QR codes, all resolving through `NEXT_PUBLIC_APP_URL` so the URLs are stable across pilots.

---

## Implementation details worth a look

### Geofence-aware wayfinding
When the attendee is outside the event polygon, the app delegates directions to Google Maps. When they cross the perimeter, the UI flips to in-event mode: distance in meters + cardinal direction to the selected POI. Turf does the point-in-polygon check; the polygon itself is organizer-drawn in the editor with Geoman.

### POI schema, v0 → v3
The longest-running design problem in the codebase. v0 was a single table with a `poi_type` enum and a fat `metadata` JSONB. v1 split the blob into per-type columns and made it worse. v2 collapsed back to one `data` column with discipline, reversed the vendor relationship (one vendor → many POIs), and added a CHECK constraint so application code didn't have to remember it. v3 split the discriminator once feature modules outgrew categories.

Full walkthrough with ERDs and the actual migrations: [event-horizon.to/build](https://event-horizon.to/build)

### Bending Leaflet to the product
React Leaflet is great until you want z-index between marker layers, organizer-drawn polygon zones, SSR safety with a library that reaches for `window` at import, and pin grouping that doesn't fall apart at festival scale. Story 02 of the case study walks through what worked.

### Realtime that doesn't set the DB on fire
Broadcast channels for ephemeral live activity, DB subscriptions only for state the client must trust, a freshness budget on the rest. Story 03.

### Migrations safe to re-run
Every block in non-trivial migrations is wrapped in `DO $$ BEGIN IF EXISTS ... END $$;` so half-applied states recover cleanly. Big refactors snapshot the affected tables into a `backup.*` schema at the top of the file. The rollback plan is part of the migration.

### Moat features (post-MVP, in flight)
A thin layer turning every event into an acquisition channel for the next one — without breaking the no-download, no-signup wedge:

- **Post-event "near you" discovery.** A printed QR on a telephone pole outlasts the event; scanning a post-event QR returns *"this one's done, here's what's near you next."*
- **Single-field opt-in email capture.** Optional, post-use. No accounts.
- **Organizer post-event dashboard + CSV export.** What the organizer takes back to their board to justify the next pilot.

---

## Project status

- **MVP live.** Map, organizer editor, anonymous engagement, organizer analytics, live broadcasts, geofence-aware wayfinding, QR kit PDF.
- **Pilot stage.** 2026 Ontario rollout — food truck festivals, ribfests, street festivals around the Toronto metro.
- **Case study in progress.** Story 01 (schema) live; stories 02–04 drafting.

---

## About

Built solo by **Cristian Tohatan** — senior full-stack (C#/.NET + React/TypeScript), recent work in AI-augmented engineering.

- 🌐 [cristian-tohatan.com](https://cristian-tohatan.com)
- 💼 [LinkedIn](https://linkedin.com/in/cristian-tohatan)

Source is closed. Happy to walk through the codebase live for serious conversations — reach out via the portfolio.
