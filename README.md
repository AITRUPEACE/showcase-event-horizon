# Event Horizon

> One QR code at the gate. Scan it, and every vendor, every stage, every washroom shows up on a searchable map in the attendee's phone browser, without an app install.

🔗 **Live:** [event-horizon.to](https://event-horizon.to)
📝 **Case study:** [event-horizon.to/build](https://event-horizon.to/build). Story 01 (schema) is live, stories 02 to 04 (Leaflet, realtime, AI co-pilot) are drafting.

> This is a showcase repository. The product itself is closed-source. What's here is an overview, the architecture, the stack, and the implementation work I'd point a hiring manager or collaborator at.

---

## Screenshots

![Attendee map](./screenshots/map.png)
![Organizer editor](./screenshots/editor.png)
![Geofence-aware wayfinding](./screenshots/wayfinding.png)

Placeholder image refs. PNGs go in `./screenshots/` with these names.

---

## The wedge

Event Horizon is built for vendor-dense outdoor events: street festivals, food truck rallies, ribfests, markets, fairs. Events where the floor plan is basically the product.

At a 50-vendor food festival, the average attendee finds maybe eight or ten booths before they call it a night. The rest of the vendors are two aisles over, unseen, through a crowd. That's a navigation problem, and the paper map at the entrance ended up crumpled in a stroller cupholder twenty minutes ago. Event Horizon is one QR code at the gate that gives the attendee a live, searchable map of the whole event in their phone browser. They can search for *sliders*, filter to *open now*, tap a pin for hours and menu photos, and see at a glance that the vendor they're looking for is 40 meters northeast, past the main stage.

The interesting part is what happens when the attendee crosses the event perimeter. Outside the perimeter, the app hands off to Google Maps for driving directions to the gate. Inside the perimeter, the app takes over with live distance and cardinal direction. Turn-by-turn is for driving *to* the event; distance plus direction is what actually works when you're walking through a crowd. That perimeter handoff is the literal "event horizon" the product is named after.

---

## Stack

| Layer | Choice |
| --- | --- |
| Framework | Next.js 15 (App Router) on Vercel |
| UI | React 19, Tailwind v4, shadcn/Radix, Framer Motion, `vaul` mobile drawer |
| Map | Leaflet + React Leaflet, `@geoman-io/leaflet-geoman-free` for organizer drawing, Turf for in-polygon checks |
| State | Zustand (with `zundo` for undo/redo in the editor), `@dnd-kit` for reordering |
| Data | Supabase Postgres, Supabase Auth, Supabase Realtime, RLS-first, single `event` schema |
| AI | OpenAI for the organizer-side co-pilot (schema-aware tool-calling) |
| Email | Resend for organizer notifications, daily schedule digest, opt-in capture |
| Maps PDF | `pdf-lib` for the printable QR kit organizers hand to volunteers |
| PWA | Serwist |
| Testing | Jest with Testing Library, Playwright e2e |
| Deploy | Vercel, canonical domain `event-horizon.to` |

---

## Architecture at a glance

The whole product lives in a single Postgres schema (`event`) covering events, POIs, zones, vendors, performers, and schedule items. Organizer access is scoped through `event.organization_members`, attendees read through published-state RLS, and engagement (comments, photos, favorites, "live" activity) works without an attendee account using privacy-safe `sessionId` hashing.

The most interesting structural decision is on the `pois` table. It carries two discriminators: `entity_type` is the coarse one (about eight values, decides which feature modules apply), and `category` is the fine one (a long tail, decides icon and visual treatment). Same row, two axes. It took three refactors to land there, and the full walkthrough with ERDs is in the case study.

JSONB is allowed, with a rule. Anything rendered but never queried lives in `pois.data`. Anything filtered, indexed, sorted, or joined on earns its own column. Sibling tables (`vendors`, `performers`, `schedule_items`) exist only for entities that have their own lifecycle: their own auth, their own RLS, their own Stripe attachment.

Mutations go through Server Actions. The API route handlers are reserved for webhooks, OG images, the QR-kit PDF endpoint, and the AI co-pilot stream. The QR-kit endpoint generates a print-ready PDF with per-zone codes that all resolve through `NEXT_PUBLIC_APP_URL`, so the URLs are stable across pilots and re-prints.

---

## Implementation details worth a look

### Geofence-aware wayfinding

The organizer draws the event perimeter as a polygon in the editor using Geoman. At runtime, Turf does the point-in-polygon check on the attendee's location. Outside the polygon, the directions panel delegates to Google Maps. Inside the polygon, the UI flips to in-event mode and shows distance in meters with a cardinal direction to the selected POI. Two different navigation modes, decided by one geometric check.

### POI schema, v0 to v3

The longest-running design problem in the codebase. v0 was a single table with a `poi_type` enum and a fat `metadata` JSONB column. v1 tried to fix the blob by splitting it into per-type columns, which made it worse. v2 collapsed back to a single `data` column with discipline, reversed the vendor relationship (one vendor, many POIs) and added a CHECK constraint so the application code didn't have to remember it. v3 split the discriminator once the feature modules grew past the categories.

The full walkthrough is at [event-horizon.to/build](https://event-horizon.to/build).

### Bending Leaflet to the product

React Leaflet is great until you want z-index between marker layers, organizer-drawn polygon zones, SSR safety with a library that reaches for `window` at import, and pin grouping that doesn't fall apart at festival scale. Story 02 of the case study walks through what worked and what I had to rewrite.

### Realtime that doesn't set the DB on fire

Broadcast channels for ephemeral live activity, DB subscriptions only for state the client must trust, and a freshness budget on the rest. Covered in story 03.

### Migrations that are safe to re-run

Every block in a non-trivial migration is wrapped in `DO $$ BEGIN IF EXISTS ... END $$;` so a half-applied state recovers cleanly. Big refactors snapshot the affected tables into a `backup.*` schema at the top of the file. The rollback plan is part of the migration, not a separate document I'd forget about.

### Moat features (post-MVP, in flight)

A thin layer on top of the existing map that turns each event into an acquisition channel for the next one, while keeping the no-account, no-download attendee wedge intact:

1. **Post-event "near you" discovery.** A printed QR on a telephone pole outlasts the event. Scanning a post-event QR returns *"this one's done, here's what's near you next."*
2. **Single-field opt-in email capture.** Optional, post-use, no accounts.
3. **Organizer post-event dashboard with CSV export.** What the organizer takes back to their board to justify the next pilot.

---

## Project status

The MVP is live: map, organizer editor, anonymous engagement, organizer analytics, live broadcasts, geofence-aware wayfinding, QR-kit PDF.

The current pilot phase is 2026 Ontario rollout: food truck festivals, ribfests, and street festivals around the Toronto metro.

The case study is in progress. Story 01 is live; stories 02 to 04 are drafting.

---

## About

Built solo by Cristian Tohatan. Senior full-stack background (C#/.NET and React/TypeScript), recent work in AI-augmented engineering.

- 🌐 [cristian-tohatan.com](https://cristian-tohatan.com)
- 💼 [LinkedIn](https://linkedin.com/in/cristian-tohatan)

Source is closed. Happy to walk through the codebase live for serious conversations; the portfolio site has the easiest way to reach me.
