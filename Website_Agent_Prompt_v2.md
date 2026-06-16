# Nobel School Website — Full Project Brief v2

## What We're Building

A fast, modern, prestigious marketing website for **Nobel** (Mental Arithmetic and Emotional Intelligence School), based in Georgia. The site is not just a brochure — it is the central conversion hub of a full marketing funnel. Every paid ad, every organic search, every shared link lands here and must turn a visiting parent into a registered lead.

Reference style: https://ucmasgeorgia.com — but significantly faster, cleaner, more modern, and built to convert.

---

## Diagnosed Problems This Site Must Solve

- **100% Meta dependency** — Meta creates awareness but not demand. Parents actively searching in Google find nothing.
- **60–70% of leads are lost** — no follow-up system, no nurture, no retargeting.
- **No Meta Pixel** — retargeting and Lookalike audiences are impossible without it. Current ads point to a Google Form where no Pixel fires — Meta has zero conversion signal.
- **No dedicated landing page** — ad traffic goes to Google Forms, not a branded page. Parents arriving from ads are ready to register. They don't need to browse.
- **No Google presence** — parents searching "აბაკუსი", "მენტალური არითმეტიკა", "გონებრივი ანგარიში" in Georgian cannot find Nobel.

---

## Goals

- Establish Nobel as a prestigious, trustworthy educational institution
- Capture leads via a dedicated `/trial-class` registration page — this is the most important page on the site
- Enable full-funnel tracking: every visit, click, and form submission measurable
- Enable retargeting: parents who visit but don't register must be reachable again via Meta and Google ads
- Support SEO: permanent organic traffic through Georgian-language blog content
- Perform well under high traffic — no crashes, no slowdowns
- Appeal to younger generations of parents — modern aesthetics, fast UX, mobile-first

---

## Build Philosophy — Design First, Then Tracking

**The build happens in two clearly separated phases.**

**Phase 1 — Design.** Build the entire site visually. All pages, all components, all layouts. No tracking code, no environment variables, no GTM scripts. Pure Astro + Tailwind. The goal is a pixel-perfect, fully navigable site deployed to Vercel.

**Phase 2 — Tracking integration.** Once the design is locked, wire in the full tracking layer in a single focused session: GTM script in BaseLayout, all event pushes in the form component, CAPI serverless function, environment variables in Vercel dashboard.

These phases are independent by design. The tracking layer does not require any structural changes to components. It slots in cleanly. This approach means design decisions are never blocked by tracking complexity, and tracking is never installed on a moving target.

**The one structural decision made during Phase 1 (not after):** `/trial-class` uses `TrialLayout.astro` — a stripped layout with no navbar links, only logo and phone number. This is a separate file from `BaseLayout.astro`. This decision is made at project setup, before any design work begins. Changing it later requires refactoring every component that imports the wrong layout.

---

## Tech Stack

### Framework — Astro
- Static site generator — purpose-built for marketing/content sites
- Ships zero JavaScript by default — fastest possible load times
- AI generates Astro well, simple syntax, rarely gets it wrong
- Blog support built-in via Markdown content collections — no CMS needed initially
- Install: `npm create astro@latest`

### Hosting — Vercel (FREE)
- Purpose-built for modern static frameworks
- Global CDN — handles any traffic spike automatically
- Automatic deploy on every GitHub push
- Preview deployments before going live
- **Do NOT use GoDaddy hosting** — built for PHP/WordPress, will underperform with Astro
- Keep GoDaddy for the domain only

### Domain
- Already registered on GoDaddy (nobeli.ge)
- After deploying to Vercel: point GoDaddy DNS A record and CNAME to Vercel's values
- Vercel handles SSL (HTTPS) automatically, free

### Environment Variables
All sensitive values are stored as environment variables — never hardcoded.
- `PUBLIC_GTM_ID` — GTM container ID (e.g. GTM-XXXXXXX)
- `PUBLIC_GA4_ID` — GA4 measurement ID (e.g. G-XXXXXXX)
- `PUBLIC_WEB3FORMS_KEY` — Web3Forms API key
- `META_PIXEL_ID` — Meta Pixel ID (used in CAPI serverless function)
- `META_ACCESS_TOKEN` — Meta system user access token (CAPI only, server-side)

**Critical:** `.env` works locally. On Vercel, these must be added manually in Project Settings → Environment Variables in the Vercel dashboard. They do not sync from your local machine. Add all of them before the first production deploy.

---

## Analytics & Tracking Layer

Everything runs through Google Tag Manager. The marketing team manages tags without touching code.

### Google Tag Manager (GTM)
- Single `<script>` in `BaseLayout.astro` `<head>` — all tracking loads through it
- Marketing team adds/edits tags independently without developer involvement
- Container manages: GA4, Meta Pixel, Clarity, conversion events, future tools

### Google Analytics 4 (GA4) — loaded via GTM
- Pure event model — every interaction is an event with up to 25 custom parameters
- Custom dimensions must be **registered in GA4 admin before launch** — data is collected regardless, but invisible in reports until registered. No retroactive backfill.
- Register these custom dimensions before first deploy:
  - `child_age_group` — maps to age dropdown value (e.g. "6-8")
  - `utm_source`, `utm_medium`, `utm_campaign` — from URL params
  - `form_location` — which page the form was submitted from
- Connects to Google Search Console and Google Ads

### Meta Pixel — loaded via GTM
- Fires `PageView` automatically on every page
- Fires `ViewContent` on `/trial-class` and `/courses` page load
- Fires `Lead` on successful form submission (standard event — not custom)
- Enables retargeting and Lookalike audiences
- **Without Pixel, Meta's algorithm optimizes for clicks only, not conversions**

### Conversions API (CAPI) — Vercel serverless function
CAPI sends conversion events from your server directly to Meta, bypassing browser restrictions, iOS 14 ATT, and ad blockers. iOS 14 blocks ~35% of client-side Pixel events. CAPI recovers that signal.

Architecture: Web3Forms submit → Vercel serverless function at `/api/capi` → Meta Graph API endpoint.

The function sends hashed user parameters to maximize Event Match Quality (EMQ):
- Hash with SHA-256: email, phone, first name, last name
- Also send: `client_ip_address`, `client_user_agent`, `fbp` cookie value (read from request)
- EMQ of 8–9 vs 4–5 can mean 20–40% better campaign performance on the same budget

**Event deduplication:** Both Pixel and CAPI fire on the same form submission. To prevent Meta double-counting, both must carry the same `event_id`. Generate a UUID on form submit, pass it to both the GTM dataLayer push and the CAPI serverless function call.

```js
// Form submit handler — generate once, use in both places
const eventId = crypto.randomUUID();

// 1. Push to dataLayer (GTM fires Pixel Lead event with this event_id)
window.dataLayer.push({
  event: 'lead_submission',
  event_id: eventId,
  child_age_group: formData.childAge,
  utm_source: urlParams.get('utm_source') || '',
  utm_medium: urlParams.get('utm_medium') || '',
  utm_campaign: urlParams.get('utm_campaign') || ''
});

// 2. Send to CAPI serverless function
fetch('/api/capi', {
  method: 'POST',
  body: JSON.stringify({
    event_id: eventId,
    email: formData.email,
    phone: formData.phone,
    first_name: formData.parentName.split(' ')[0],
    fbp: getCookie('_fbp'),
    client_ip: '', // populated server-side from request headers
    user_agent: navigator.userAgent
  })
});
```

### Microsoft Clarity — loaded via GTM
- Free forever, no quota
- Session recordings: watch real parents navigate the form
- Heatmaps: see exactly where parents click, scroll, and drop off
- Integrates with GA4: sessions flagged in GA4 link to recordings in Clarity
- Add via GTM as a Custom HTML tag — 5 minutes

### Consent Mode v2
- Required for GA4 and Meta to use modeled conversion data (fills gaps where consent not given)
- Use Cookiebot free tier (covers under 100 pages) or Klaro (open-source, self-hosted, no limits)
- Load consent platform via GTM before all other tags
- Without it: both GA4 and Meta silently underreport conversions for users who don't consent

---

## DataLayer Schema

**Design this before writing any component code.** Every `dataLayer.push()` in the codebase must follow this schema. Inconsistent keys break GTM triggers and make GA4 dimensions unreliable.

```js
// Standard structure for all events
window.dataLayer.push({
  event: '<event_name>',        // required, snake_case
  event_id: '<uuid>',           // required for form events (deduplication)
  page_type: '<page_type>',     // 'landing', 'home', 'courses', 'blog', 'contact'
  form_location: '<location>',  // 'trial_class_page', 'contact_page'
  child_age_group: '<group>',   // '3-5', '6-8', '9-12', '13+' — from dropdown
  utm_source: '<source>',       // read from URL params at page load
  utm_medium: '<medium>',
  utm_campaign: '<campaign>'
});
```

Named events used in this project:
- `page_view` — automatic via GTM Enhanced Measurement
- `view_content` — fires on `/trial-class` and `/courses` load
- `lead_submission` — fires on successful Web3Forms submit
- `phone_click` — fires when user taps a `tel:` link
- `whatsapp_click` — fires when user taps the WhatsApp button
- `form_field_focus` — fires when user focuses first form field (tracks form starts)
- `cta_click` — fires on any CTA button click, with `cta_location` parameter

---

## Site Structure

### LAYOUTS (defined at project setup, before design work)

**`BaseLayout.astro`** — used by all pages except /trial-class
```astro
---
// Props every page passes
const { title, description, ogImage, noindex = false } = Astro.props;
---
<html lang="ka">  <!-- Georgian language — critical for SEO -->
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>{title} | Nobel სკოლა</title>
  <meta name="description" content={description} />
  <link rel="canonical" href={Astro.url} />
  {noindex && <meta name="robots" content="noindex, nofollow" />}
  <!-- Open Graph -->
  <meta property="og:title" content={title} />
  <meta property="og:description" content={description} />
  <meta property="og:image" content={ogImage ?? '/og-default.jpg'} />
  <meta property="og:url" content={Astro.url} />
  <meta property="og:type" content="website" />
  <!-- GTM script goes here in Phase 2 -->
  <!-- JSON-LD schema goes here -->
</head>
```

**`TrialLayout.astro`** — used only by /trial-class
- Stripped header: Nobel logo (links to homepage) + phone number only
- No navbar, no footer links, no distractions
- Focus is entirely on the registration form
- Same `<html lang="ka">`, canonical, OG tags as BaseLayout

### PRIORITY PAGE: /trial-class — Registration Landing Page

**This is the most important page on the entire site.**
- Every paid ad (Meta, Google, YouTube) must point here — never to the homepage
- Uses `TrialLayout.astro` — not BaseLayout
- Above the fold: headline + registration form. Nothing else competing for attention.
- Below fold: 2–3 video testimonials (YouTube embeds), key benefit stats
- On submit: redirect to `/thank-you` (not inline message — see below)

**Form fields:**
- Parent name (text)
- Child name (text)
- Child age group (dropdown: 3–5 / 6–8 / 9–12 / 13+) — **must be dropdown, not free text**. Free text produces inconsistent values ("7", "7 years", "seven") that break GA4 dimensions and Meta audience segmentation.
- Phone number (tel)
- Email (email)
- Hidden fields (populated by JS on page load — never shown to user):
  - `utm_source`, `utm_medium`, `utm_campaign`, `utm_content` — read from `window.location.search`
  - `event_id` — UUID generated on submit (for CAPI deduplication)

**On submit:**
1. Generate `event_id` UUID
2. Push `lead_submission` event to dataLayer (GTM fires Pixel Lead + GA4 event)
3. POST to `/api/capi` serverless function with hashed user data + event_id
4. Redirect to `/thank-you`

**ViewContent event:** fires on page load via GTM page_type variable trigger (not in component code).

### /thank-you — Conversion Confirmation Page

Dedicated page, not an inline message. This enables clean GA4 goal tracking (pageview on /thank-you), Google Ads conversion import, and Meta pixel ViewContent signal for post-conversion audiences.

- Uses `BaseLayout.astro` with `noindex={true}` — excluded from search index and sitemap
- Message: thank parent, set expectation ("we'll call within 24 hours"), one WhatsApp CTA
- `noindex` prevents Google from indexing a page with no value for organic visitors

### 1. Home (/)
- Hero — headline, subheadline, CTA button → `/trial-class`
- Partner/school logos strip
- Benefits section — icons + short descriptions (focus, memory, confidence, creativity)
- About section — what Nobel is, what makes it different from regular tutoring
- Stats counter — animated numbers (children trained, years active, partner schools)
- Video testimonials — YouTube embeds, not self-hosted. Reliable, fast, no storage cost.
- FAQ accordion
- Final CTA section → `/trial-class`
- Footer

### 2. About (/about)
- School story, mission, values
- Team section — photos, names, credentials
- Partnership logos

### 3. Courses (/courses)
- Programs listed by age group
- Each course: name, description, duration, outcomes
- CTA on each course → `/trial-class`
- `ViewContent` event fires on page load (configured in GTM, not in component)

### 4. Blog (/blog)

**URL slug strategy — romanized, decided before first post:**
Blog post slugs use romanized Georgian, not Georgian script.
- ✅ `/blog/mentaluri-aritmetika-bavshvebistvis`
- ❌ `/blog/მენტალური-არითმეტიკა-ბავშვებისთვის`

Georgian script URLs work technically but render as percent-encoded gibberish when shared on WhatsApp and Viber. Romanized slugs share cleanly. This applies to all blog posts. **Do not mix strategies** — changing slugs after indexing breaks inbound links.

Content Collections are built into Astro 2.0+ — **do not run `npx astro add @astrojs/content-collections`**, it does not exist as a package. Just create `src/content/config.ts`:

```ts
import { defineCollection, z } from 'astro:content';

const blog = defineCollection({
  type: 'content',
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    updatedDate: z.coerce.date().optional(),
    ogImage: z.string().optional(),
  })
});

export const collections = { blog };
```

- Georgian-language articles targeting search keywords
- Examples: "მენტალური არითმეტიკა ბავშვებისთვის", "აბაკუსი — რა სარგებელი აქვს", "როგორ ვითარდება ყურადღება"
- Every article ends with CTA → `/trial-class`
- First 3 articles at launch. Then 1 per month minimum.
- Each post: Open Graph tags, Article JSON-LD schema, meta description

**Article JSON-LD schema template:**
```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "{{title}}",
  "datePublished": "{{pubDate}}",
  "dateModified": "{{updatedDate}}",
  "author": { "@type": "Organization", "name": "Nobel სკოლა" },
  "publisher": {
    "@type": "Organization",
    "name": "Nobel სკოლა",
    "logo": { "@type": "ImageObject", "url": "https://nobeli.ge/logo.svg" }
  }
}
```

### 5. Events (/events)
- Competitions, masterclasses, national/international achievements
- Photo and video gallery (YouTube embeds for video)
- Builds credibility and social proof

### 6. Contact (/contact)
- Address, phone, email
- Google Maps embed
- Social media links: Facebook, Instagram, TikTok, YouTube
- Secondary registration form — same Web3Forms setup, same UTM hidden fields

---

## GTM Tag Architecture

Configure these in GTM after Phase 1 design is complete.

### Triggers
| Trigger name | Type | Condition |
|---|---|---|
| All pages | Page view | Always |
| Trial class page | Page view | Page path equals `/trial-class` |
| Courses page | Page view | Page path equals `/courses` |
| Lead submission | Custom event | Event name equals `lead_submission` |
| Phone click | Click | Click URL contains `tel:` |
| WhatsApp click | Custom event | Event name equals `whatsapp_click` |
| Form field focus | Custom event | Event name equals `form_field_focus` |
| CTA click | Custom event | Event name equals `cta_click` |
| Scroll 50% | Scroll depth | Scroll depth threshold 50% |
| Scroll 90% | Scroll depth | Scroll depth threshold 90% |

### Tags
| Tag | Trigger | Notes |
|---|---|---|
| GA4 Configuration | All pages | Measurement ID from variable |
| GA4 — ViewContent | Trial class page, Courses page | Event name: `view_content` |
| GA4 — Lead | Lead submission | Event name: `lead`, include child_age_group, utm params |
| GA4 — Phone click | Phone click | Event name: `phone_click` |
| GA4 — Scroll depth | Scroll 50%, Scroll 90% | For blog engagement |
| Meta Pixel — Base | All pages | PageView fires automatically |
| Meta Pixel — ViewContent | Trial class page | Standard ViewContent event |
| Meta Pixel — Lead | Lead submission | Standard Lead event, pass `event_id` for dedup |
| Clarity | All pages | Custom HTML tag with Clarity script |

---

## WhatsApp Floating Button

Move this from "Future Expansions" — it belongs in Phase 1 design. Georgian parents use WhatsApp constantly. A parent who won't fill a form will often message directly.

```html
<!-- Place before </body> in BaseLayout and TrialLayout -->
<a
  href="https://wa.me/995XXXXXXXXX"
  class="whatsapp-fab"
  aria-label="WhatsApp-ით დაგვიკავშირდით"
  onclick="window.dataLayer.push({ event: 'whatsapp_click' })"
  target="_blank"
  rel="noopener"
>
  <svg><!-- WhatsApp icon --></svg>
</a>
```

Style as a fixed circle in bottom-right corner. Tailwind: `fixed bottom-6 right-6 z-50`.

---

## Marketing Funnel This Site Supports

**Awareness** (brings parents to the site)
- TikTok organic content — child progress videos, classroom clips
- YouTube — long-form content, appears in Google search results
- Meta Ads — video and Reels format only (static ads perform 30–40% worse)
- Google Display Remarketing — Nobel banners follow parents across other websites after they visit

**Consideration** (parents actively searching)
- Google Search Ads — keywords: "აბაკუსი", "მენტალური არითმეტიკა", "გონებრივი ანგარიში ბავშვებისთვის", "Nobel სკოლა". Budget: 15–20₾/day to start. These leads are more valuable than Meta leads — the parent was already searching.
- Google Maps / Business Profile — appears in local map pack. Parent can call directly from Google without visiting the site.
- Blog / SEO — permanent organic presence, no ongoing ad spend required

**Conversion** (parent registers)
- `/trial-class` landing page — the entire site points here
- Free trial class / masterclass offer — lowest friction entry point
- Phone follow-up — personal, direct, in Georgian
- Email nurture sequence — automated follow-up after form submission (Brevo free tier)

**Retention & Growth**
- Referral program — parent refers a friend, receives one month credit
- Parent community groups (Facebook/WhatsApp) — teachers + parents, news, updates
- Student progress videos — UGC-style, authentic, powerful trust signal
- B2B partnerships — schools, kindergartens

---

## Design Direction

- **Modern and prestigious** — not corporate, not toy-like. Warm, confident, trustworthy
- **Mobile-first** — the majority of Georgian parents will arrive on phone, especially from Meta and TikTok ads
- **Fast perceived performance** — above-the-fold content loads instantly; images lazy-loaded below fold
- **Georgian language** — primary language is Georgian. Font must support Georgian script. Use **Noto Sans Georgian** — free, reliable, widely supported. Set `<html lang="ka">` on every page.
- **Video-forward** — parent testimonial videos (YouTube embeds) are the highest-trust content. Feature them prominently.
- Animations: subtle and purposeful — entrance animations on scroll, counter animations on stats. Not flashy.
- Color palette: Nobel brand colors. Define hex codes before development starts.

---

## AI-Assisted Development — How to Do It Right

**The problem with AI-built sites:** people generate everything at once, paste without reviewing, have no version control, and never test. That's why they break.

**Phase 1 — Design workflow:**
1. Set up project pipeline first (`npm create astro@latest` → GitHub → Vercel) — confirm empty site deploys before writing a single component
2. Create `BaseLayout.astro` and `TrialLayout.astro` as first files — every other component uses one of these two
3. Build section by section — one component at a time, test before moving on
4. Commit to GitHub after every working section
5. Test on mobile after every section
6. Use Vercel preview deployments to review before pushing to main

**Phase 2 — Tracking integration workflow:**
1. Add all environment variables to Vercel dashboard first
2. Add GTM script to `BaseLayout.astro` and `TrialLayout.astro`
3. Add dataLayer pushes to form component
4. Deploy and verify GTM Preview mode shows events firing correctly
5. Configure all GTM tags and triggers (see GTM Tag Architecture section)
6. Deploy CAPI serverless function, test with Meta Test Events tool
7. Verify event deduplication in Meta Events Manager (both Pixel and CAPI events visible, count as one)

**What AI generates well in Astro:**
- Component structure and layouts
- Tailwind CSS responsive styling
- Web3Forms integration
- GTM / GA4 / Meta Pixel script setup
- CAPI serverless function (Vercel edge function format)
- JSON-LD schema markup
- Open Graph meta tags
- Blog with Markdown content collections
- Animations with vanilla CSS

---

## What You Do NOT Need (Right Now)

- **Server-side GTM (sGTM)** — adds infrastructure cost and complexity not justified at current scale. Revisit when running 3+ ad platforms simultaneously (Meta + Google + TikTok + LinkedIn). Client-side GTM + CAPI serverless function covers the same attribution needs.
- **Database / Supabase** — pure marketing site, no backend logic needed. Revisit when adding student accounts or enrollment tracking
- **WordPress** — heavy, slow, constant security maintenance, not worth it
- **GoDaddy hosting** — keep domain only, host on Vercel
- **Drag-and-drop builders** (Framer, Webflow) — platform lock-in, limited control, doesn't fit your workflow

---

## Future Expansions (Optional, Later)

- **sGTM (Server-side GTM)** — when running Meta + Google + TikTok + LinkedIn simultaneously. Single event fan-out, first-party long-lived cookies (bypasses Safari ITP 7-day limit), PII stripping before forwarding to vendors.
- **BigQuery export from GA4** — once 3–6 months of data accumulated. JOIN GA4 event table against Supabase registrations by email — full attribution chain from ad campaign to enrolled student to LTV. Free tier covers Nobel's volume.
- **CMS (Sanity or Decap CMS)** — lets non-technical staff add blog posts and update content without touching code
- **Email automation (Brevo)** — automated follow-up sequence triggered on form submission. Free tier covers Nobel's volume.
- **Stripe** — for online course fee collection
- **GA4 Predictive Audiences** — activates automatically once 1,000+ returning converters in 28-day window. Publishes purchase probability audiences to Google Ads for smart bidding.

---

## Phased Checklist

### Phase 0 — Accounts & Infrastructure (do immediately, before design)
- [ ] Create Meta Business Manager account (if not already done properly)
- [ ] Create Meta Pixel in Events Manager — save Pixel ID
- [ ] **Verify nobeli.ge domain in Meta Business Manager** — add DNS TXT record to GoDaddy NOW. Takes 1–3 days to propagate. This is the bottleneck.
- [ ] Create GTM container — save GTM-XXXXXXX ID
- [ ] Create GA4 property — save G-XXXXXXX measurement ID
- [ ] Register GA4 custom dimensions: `child_age_group`, `utm_source`, `utm_medium`, `utm_campaign`, `form_location`
- [ ] Create Web3Forms account — save API key
- [ ] Create GitHub repository
- [ ] Create Vercel account, connect to GitHub
- [ ] Create Meta system user + access token for CAPI (in Meta Business Manager → System Users)

### Phase 1 — Design Build
- [ ] `npm create astro@latest` — confirm empty site deploys to Vercel before writing components
- [ ] Create `BaseLayout.astro` with `lang="ka"`, canonical URL, OG tags, noindex prop, robots meta
- [ ] Create `TrialLayout.astro` — stripped header, no navbar
- [ ] Add all environment variables to Vercel dashboard (not just .env)
- [ ] Build `/trial-class` page — form with age dropdown, UTM hidden fields, redirect to `/thank-you`
- [ ] Build `/thank-you` page — with `noindex={true}`, exclude from sitemap
- [ ] Build Homepage — all sections, all CTAs → `/trial-class`
- [ ] Build `/about`, `/courses`, `/events`, `/contact` pages
- [ ] Add WhatsApp floating button to BaseLayout and TrialLayout
- [ ] Build Blog structure with Content Collections (`src/content/config.ts`)
- [ ] Confirm romanized URL slugs on all blog posts
- [ ] Set up `LocalBusiness` JSON-LD schema in BaseLayout head
- [ ] Test full site on mobile (every page)
- [ ] Commit working state to GitHub
- [ ] Point GoDaddy domain DNS to Vercel — confirm nobeli.ge live with HTTPS

### Phase 2 — Tracking Integration
- [ ] Add GTM `<script>` to `BaseLayout.astro` and `TrialLayout.astro` heads
- [ ] Add dataLayer pushes to form component (lead_submission, form_field_focus, event_id)
- [ ] Add UTM capture JS to form (reads URL params → hidden fields)
- [ ] Deploy and verify: GTM Preview mode shows all events firing correctly
- [ ] Configure all GTM tags and triggers (see GTM Tag Architecture section above)
- [ ] Add Microsoft Clarity tag in GTM (Custom HTML)
- [ ] Add Consent Mode v2 (Cookiebot or Klaro) — load before all other GTM tags
- [ ] Build CAPI serverless function at `/api/capi.js` — hash params (SHA-256), POST to Meta Graph API
- [ ] Test CAPI with Meta's Test Events tool in Events Manager
- [ ] Verify deduplication: Pixel Lead + CAPI Lead both appear in Events Manager, counted once
- [ ] Configure phone click tracking trigger in GTM (Click URL contains `tel:`)
- [ ] Configure scroll depth tracking in GTM (50%, 90% thresholds)
- [ ] Create Lead conversion event in Meta Ads Manager (using Pixel Lead event)

### Phase 3 — Content & SEO Launch
- [ ] Publish first 3 Georgian blog articles
- [ ] Generate and verify sitemap (confirm `/thank-you` is excluded)
- [ ] Create Google Search Console property, verify ownership, submit sitemap
- [ ] Set up Google Business Profile (separate from site)

### Phase 4 — Meta Ads Switch
- [ ] Update all Meta ad destination URLs: Google Forms → `nobeli.ge/trial-class`
- [ ] Add UTM params to all ad URLs: `?utm_source=meta&utm_medium=paid&utm_campaign=NAME`
- [ ] Switch Meta campaign objective to Conversion → Lead event (not Traffic)
- [ ] Keep Google Forms active for 48 hours as fallback
- [ ] Monitor Events Manager for 24 hours — confirm Lead events arriving from ad traffic

### Phase 5 — Advanced Tracking (after site stabilizes, ~4–6 weeks post-launch)
- [ ] Set up GA4 Funnel Exploration report: direct URL → /trial-class → form_field_focus → lead_submission
- [ ] Set up form abandonment tracking in GTM (blur events on individual fields)
- [ ] Link GA4 → Google Ads account (for remarketing audiences + smart bidding signal)
- [ ] Set up Google Search Ads once GA4 has conversion data
- [ ] Set up Brevo email sequence: auto-reply on submit → follow-up day 1 → follow-up day 3

---

## Content Needed Before Development Starts

- School logo — SVG preferred, PNG fallback
- Brand colors — hex codes
- School name in Georgian and English
- Hero headline and subheadline (Georgian)
- Benefits list — 6–8 items with short descriptions
- About/story text
- Course descriptions with age groups (3–5, 6–8, 9–12, 13+)
- Partner/school logos
- Team photos and bios
- Parent video testimonials — minimum 2–3 for launch (upload to YouTube, embed via YouTube player)
- Contact details: address, phone, email
- Social media links: Facebook, Instagram, TikTok, YouTube
- First 3 blog article topics and text
- WhatsApp business number
