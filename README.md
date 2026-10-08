# PrimeBuild Engineering website

A two-route marketing site and project inquiry form for PrimeBuild Engineering.

- `/` is the homepage with anchor sections.
- `/contact` is the project inquiry page.
- `POST /api/inquiry` is a Vercel serverless function that validates the form and emails it to the team.

Stack: React 18, TypeScript, Vite 6, Tailwind CSS 4, React Router 7, Zod. Fonts (Sora and Inter) are bundled with the site, so there are no calls to Google Fonts.

## Requirements

- Node.js 20 or newer
- npm 10 or newer

## Run locally

```bash
npm install
cp .env.example .env.local     # optional for the front end, required to test the form
npm run dev                    # http://localhost:5173
```

`npm run dev` serves the front end only. The form posts to `/api/inquiry`, which Vite does not run. To test the form end to end locally:

```bash
npm i -g vercel
vercel dev                     # runs the site and the /api function together
```

Without the API running, the form will show "Your inquiry was not sent" and offer the email address. It never shows success unless the server confirms delivery.

## Commands

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the Vite dev server |
| `npm run build` | Type-check, build to `dist/`, then run the post-build step |
| `npm run preview` | Serve the production build locally |
| `npm run typecheck` | TypeScript check for the app, scripts and API |
| `npm test` | Component tests (Vitest and Testing Library) |
| `npm run test:api` | Tests the API handler with a mocked network |
| `npm run check` | Runs all of the above in order |
| `npm run assets` | Regenerates `og-image.png`, `favicon-32.png` and `apple-touch-icon.png` |

The post-build step (`scripts/postbuild.mjs`) replaces `__SITE_URL__` in the HTML, writes a route-specific `dist/contact/index.html` so `/contact` has its own title and metadata for crawlers, and writes `sitemap.xml` and `robots.txt`.

## Project structure

```
api/inquiry.ts              Serverless function: validate, spam checks, send email
src/
  App.tsx, main.tsx         Routes (/ and /contact, plus a 404 fallback)
  index.css                 Design tokens, section themes, buttons, form controls
  components/               Header, Footer, Layout, Button, Wordmark, InquiryForm, Turnstile, Rule
  pages/Home.tsx            Homepage, composed from pages/home/*
  pages/home/               Hero, Needs, Services, ProjectTypes, Deliverables, Work, Why, Process, Faq, FinalCta
  pages/Contact.tsx         Inquiry page
  data/content.ts           All homepage copy in one place
  data/workSamples.ts       Work samples (see "Adding real work samples")
  graphics/                 SVG hero illustration and three sample graphics
  lib/inquirySchema.ts      Form schema shared by the browser and the API
  lib/site.ts               Site constants, nav, optional hero photo
  hooks/                    useDocumentMeta, useInView
  test/                     Component tests
scripts/                    postbuild.mjs, make-brand-assets.mjs, test-api.ts
public/                     favicon, social image, icons
vercel.json                 Clean URLs, SPA fallback, security headers
```

## Environment variables

Copy `.env.example` for local work. In Vercel, add them under Project Settings, then Environment Variables. **Never commit `.env` files.** `.gitignore` already excludes them.

| Variable | Where it is used | Required | Notes |
| --- | --- | --- | --- |
| `SITE_URL` | Build | Recommended | Your public URL, for example `https://www.example.com`. Used for canonical links, Open Graph, sitemap and robots. If empty on Vercel, `VERCEL_PROJECT_PRODUCTION_URL` is used. |
| `VITE_CONTACT_EMAIL` | Browser | No | Public email shown on the site. Defaults to `proengineers.team@gmail.com`. |
| `RESEND_API_KEY` | Server only | **Yes, for the form** | From resend.com. |
| `INQUIRY_FROM_EMAIL` | Server only | **Yes, for the form** | A sender on a domain you verified in Resend, for example `PrimeBuild Website <inquiries@yourdomain.com>`. |
| `INQUIRY_TO_EMAIL` | Server only | No | Delivery address. Defaults to `proengineers.team@gmail.com`. Comma-separate for several. |
| `TURNSTILE_SECRET_KEY` | Server only | Strongly recommended | Cloudflare Turnstile secret. When set, the server rejects any submission without a valid token. |
| `VITE_TURNSTILE_SITE_KEY` | Browser | Needed if the secret is set | Turnstile site key. Public by design. |
| `ALLOWED_ORIGINS` | Server only | No | Comma-separated origins allowed to call the API. By default only the same host is allowed. |

Anything starting with `VITE_` is bundled into the browser code. Do not put secrets in those.

## Set up the inquiry form

The form delivers email through [Resend](https://resend.com). It is **not delivering anything until you complete these steps.**

1. Create a Resend account and an API key.
2. Add and verify a sending domain in Resend (DNS records). Set `INQUIRY_FROM_EMAIL` to an address on that domain.
   - For a first test only, Resend offers a shared test sender (`onboarding@resend.dev`). As of this writing it can only send to the email address of your own Resend account. Check Resend's current docs, and use a verified domain for production.
3. Create a free Cloudflare Turnstile widget (Cloudflare dashboard, Turnstile). Add your production domain. Copy the site key and secret key into the variables above.
4. Add all variables in Vercel and redeploy.
5. **Send a real test inquiry** from the live `/contact` page and confirm it arrives at `proengineers.team@gmail.com` (check spam too). Reply to it: the reply goes to the person who filled in the form, because the API sets `reply_to`.

How the form protects itself:

- The same Zod schema validates in the browser and on the server.
- A hidden honeypot field, a minimum fill time, same-origin checks, and a best-effort rate limit (serverless instances do not share memory, so Turnstile is the main control).
- Turnstile verification on the server when `TURNSTILE_SECRET_KEY` is set.
- All user text is HTML-escaped in the email. Header values cannot contain line breaks.
- The API returns `{ ok: true }` only after Resend accepts the message. A missing configuration returns `503`, a provider failure returns `502`, and the page shows a clear "not sent" message with the email address as a fallback.
- Submissions are not stored anywhere. They exist only in the email. The server logs error codes but not form contents.

File attachments are **not** supported in this version. Secure uploads need a storage service and virus scanning, which should be a deliberate decision. The contact page tells people we will explain how to share files after replying. If you want uploads later, Vercel Blob or S3 with presigned URLs are the usual route.

## Push to GitHub

```bash
git init
git add .
git commit -m "Initial PrimeBuild Engineering site"
git branch -M main
git remote add origin https://github.com/<your-account>/<your-repo>.git
git push -u origin main
```

Before the first push, run `git status` and confirm no `.env` files are staged.

## Deploy to Vercel

1. In Vercel, choose **Add New, Project**, then import the GitHub repository.
2. Framework preset: **Vite**. Build command: `npm run build`. Output directory: `dist`. These are detected automatically.
3. Add the environment variables from the table above, including `SITE_URL`.
4. Deploy.
5. Add your custom domain under Project Settings, Domains. Then set `SITE_URL` to that domain and redeploy so the sitemap and canonical links use it.
6. Test the form on the live site (see above).
7. Submit `https://<your-domain>/sitemap.xml` in Google Search Console.

## Replacing placeholder content

**Hero image.** The hero uses an SVG illustration, labeled as an illustration. To use a photograph you own or are licensed to use, put it in `public/images/` and set `HERO_PHOTO` in `src/lib/site.ts`.

**Work samples.** The three work-sample graphics are generated illustrations, each labeled "Sample graphic". To add a real drawing, edit `src/data/workSamples.ts` and add an item with `kind: "project"`, an `image`, and a `permission` note recording that you have the right to show it. TypeScript will not compile a project item without that field. Once every item is a project, the section's intro text switches automatically.

**Copy.** All homepage text is in `src/data/content.ts`. The copy describes how a service like this typically works (scope confirmed before work, plan-check support within agreed scope, quote after review). **The owner should confirm that each statement matches how PrimeBuild actually operates** before launch.

## Professional and legal notes

- The site does not state credentials, license numbers, years in business, awards, client names, testimonials, or project statistics, because none were supplied.
- Stamps, signed calculations and sealed plans are described as depending on scope, jurisdiction, licensing, and confirmation by an appropriately licensed professional.
- The site does not guarantee permit approval or state-by-state availability.
- There is no privacy policy page, because the brief limits the site to two designed routes. The contact page explains how inquiry details are used. If your jurisdiction or customers require a formal privacy policy, add one.

## Accessibility

Semantic landmarks and heading order, skip link, visible focus rings, keyboard-operable tabs, accordion and mobile menu (with Escape and focus containment), labeled form controls, an error summary that receives focus, `aria-invalid` and `aria-describedby` on fields, reduced-motion support, and text colors that meet WCAG AA contrast (checked numerically). A full audit with a screen reader and a browser tool such as axe or Lighthouse is still recommended before launch.

## Known limitations

- Email delivery, Turnstile and the Vercel function have **not** been tested against live services in this build environment. Only a mocked test of the API handler has been run. Do the live test described above.
- No real photographs or project drawings were supplied, so none are included.
- The site was verified by type-checking, a production build, component tests and numeric contrast checks. It has not been viewed in a real browser by the builder. Review it at phone, tablet and desktop widths before launch.
- The in-memory rate limiter is best effort only.
