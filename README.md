# PRINTIFY DZ — Custom DTF & UV DTF printing

Arabic-first storefront, French/English translations, Vite + React + TypeScript + Tailwind v4, Supabase, and Vercel serverless API routes. The supplied brand reference is used as identity inspiration only; it is not represented as verified product photography.

## 1. Complete application directory

```text
api/
  db-client.js              # Existing resilient server-only Supabase client
  db-wake.js                # Existing recovery helper
  catalog.js                # Database-backed garments, pricing, 58 wilayas, verified portfolio
  cart.js                   # Session-scoped cart create/read/update/delete
  communes.js               # Cascading commune lookup
  upload.js                 # Signed original-artwork / private-receipt uploads
  orders.js                 # Server-validated checkout and recalculated totals
  track.js                  # Exact order number AND phone required
src/
  App.tsx                   # Main entry
  main.tsx                  # React bootstrap
  index.css                 # Tailwind v4
  printify.css              # Responsive studio-grade design system
  components/
    Brand.tsx               # CMYK dot-wave logo and honest blank illustrations
    Layout.tsx              # RTL-aware sticky header, cart drawer, footer, reusable UI
    PrintifyApp.tsx          # Route shell and loading/error handling
  lib/
    api.ts                  # Typed API and signed-storage upload helpers
    context.tsx             # Locale/RTL wrapper, live catalog and persistent cart
    i18n.ts                 # Complete AR / FR / EN interface dictionaries
    placement.ts            # Aspect-correct artwork fitting and safe-area clamping
    types.ts                # Catalog, pricing, cart, placement and order types
  pages/
    Home.tsx                # Storefront and quality sample order flow
    Studio.tsx              # Interactive HTML5 canvas DTF design studio
    Portfolio.tsx           # Verified-only portfolio, filters and macro zoom
    Checkout.tsx            # 58-wilaya checkout, communes, COD and transfer receipt
    Track.tsx               # Phone-protected order tracking with 30-second refresh
public/
  favicon.svg
  images/blank-hoodie.webp   # AI-generated blank used ONLY as illustrative studio preview
supabase/
  schema.sql                # Complete schema, indexes, RLS and storage configuration
  seed.sql                  # Complete exported garments, pricing, 58 rates and 1,541 communes
scripts/
  seed-communes.mjs          # Reproducible commune import + SQL seed export
  test-api.mjs              # Real Supabase end-to-end resource tests
  test-browser.py           # Deployed browser smoke + responsive screenshots
  verification.json         # Latest automated API verification summary
```

## 2. Database and initial data

See `supabase/schema.sql` and `supabase/seed.sql` for copy-pasteable SQL. The live tables are provisioned with Supabase tools. No ORM or direct database connection is used. Existing deployments do not need to run these scripts again.

`garment_catalog` has blank oversized hoodie, classic tee, cap, transfer-only and sample pack configurations. `print_pricing` has LOGO 10×10, A4 21×29.7 and A3 29.7×42 cm. `wilaya_shipping_rates` includes the requested 01–58 destination codes. `communes` contains 1,541 municipalities from the open-source `othmanus/algeria-cities` 58-wilaya dataset. The selected shipping coverage is an explicit merchant configuration, not a claim about the latest administrative division.

**All seeded commercial prices, blank availability and delivery rates are illustrative configuration, NOT verified merchant tariffs or guaranteed stock.** This is clearly stated in the UI. Update them before commercial launch. Carrier names indicate a desk-delivery option, not a live courier integration.

`portfolio_items` intentionally has zero rows. Add authentic merchant photos with multilingual titles, category, print type and `verified=true` to publish. The site never invents customer work. The hoodie hero and sample-sheet artwork are visibly labeled illustrative previews. No sewing or tailoring is offered.

## 3. Translations and RTL

`src/lib/i18n.ts` contains AR / FR / EN dictionaries. `StoreProvider` sets `html.lang` and `html.dir` synchronously on locale changes and persists the chosen language. Arabic defaults to RTL, French and English to LTR. CSS logical properties and direction-aware arrows adapt all layouts. Product names and geographic data come from the database.

## 4. Studio and checkout

- Native HTML5 canvas with pointer/touch dragging, scaling, rotation, centering and reset.
- Drop or choose transparent PNG, SVG or PDF (first page preview), maximum 20 MB. Original file is kept in Storage, not replaced with the preview.
- Blank color selection, front/back/sleeve/cap/transfer areas and garment size.
- Artwork aspect ratio is preserved. Rotated bounds are fitted and center clamped inside the safe print area.
- The selected size is the billed print format. DPI is conservative, measured against the full selected format. PDF/SVG are flagged for preflight review rather than falsely claiming raster DPI. Placement coordinates are normalized to the safe area on a 600×650 illustration canvas; use the artwork layout helper for an identical preview. Physical production must be reviewed by the print operator.
- Database prices are recalculated in the API; client totals are never trusted. Cart creation requires a valid blank/color/area/format and original uploaded artwork.
- Cart is persistent in Postgres and scoped by an unguessable browser UUID (guest capability token). No customer accounts are required.
- Checkout validates 10-digit Algerian 05/06/07 phone numbers and full home address; all 58 wilayas have dependent commune suggestions plus a manual-entry fallback. Home / desk rates are read from Postgres. Douera pickup is available for Algiers with zero shipping.
- COD is default. BaridiMob/CIB means manual transfer after telephone confirmation, **not** an online bank/payment API. No banking details are fabricated, and no online charge occurs. Optional receipts go into a **private** bucket; only their internal storage references are stored with orders.
- Public tracking requires a cryptographically random order number AND matching phone; it returns only minimal non-sensitive status details. No public order listing endpoint exists. Status refreshes every 30 seconds while the tracking result is open.
- Operators update status in the secured Supabase dashboard (`pending`, `printing`, `pressing`, `shipped`, `delivered`). There is deliberately no unauthenticated admin endpoint. Transfer-only orders may skip heat pressing when updated by an operator.

## Local build

```bash
npm install
npm run build
# Reproducible data import, only in a trusted server environment:
node --env-file=.env scripts/seed-communes.mjs
# Real database tests (creates and cleans up one test order and cart):
node --env-file=.env scripts/test-api.mjs
```

Required server variables: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`. Never expose the service-role key to Vite or the browser. Frontend operations use same-origin `/api/*` routes. Deploy to Vercel for serverless API execution; `vite` alone does not run the API routes.

## Security and launch checklist

- RLS denies all direct anon table access. Server routes use the supplied resilient service-role client and perform validation. Cart mutation is always filtered by session UUID, tracking by phone AND order number.
- Signed uploads avoid sending large base64 payloads through serverless functions. Artwork paths use random UUIDs. Original artwork is publicly readable by an unguessable URL so the print preview can use it. Receipts remain private and are never returned by tracking.
- Confirm merchant tariffs, real blank inventory, business contact details, carrier coverage, lead times, care instructions, refund/cancellation policy and legal/privacy notices with PRINTIFY DZ before opening commercial orders.
- Configure Vercel Firewall rate limits / abuse controls for public upload, cart, checkout and tracking endpoints before public marketing. Signed upload credentials are guest-accessible by design.
- Production artwork preflight should inspect embedded raster resolution in PDFs/SVGs, licensed artwork and color profiles; a vector file does not guarantee every embedded image is print-ready.
- Establish private receipt retention and abandoned-upload cleanup policies. Checkout currently uses separate Supabase insert/cleanup operations rather than a database transaction: review retry/idempotency and reconciliation requirements for commercial-scale traffic.
- The SQL file includes unique constraints, checks and lookup indexes for fresh deployments. Review/apply migrations in the secured database dashboard for any additional launch hardening of an existing instance.
- No email/SMS notifications, automatic bank verification or courier API booking are claimed or simulated.
