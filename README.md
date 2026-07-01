# OrthoNow — Developer Assignment Submission

**Candidate:** Kamini Kumari
**Assignment:** Namoza Developer (Web Dev & Martech) — Position 1

## Contents
- `task1-schema.md` — GTM event schema + booking funnel dataLayer JSON + GA4 funnel setup
- `task2-landingpage.html` — Single-file "Book a Consultation" landing page (open directly in browser, no server needed)
- `task3-integration.md` — HubSpot + WhatsApp (Karix) + Google Ads integration write-up
- `pagespeed-screenshot.png` — PageSpeed Insights Mobile score for task2-landingpage.html (add this after testing)

## How to test the landing page
1. Open `task2-landingpage.html` directly in any browser (double-click the file).
2. Open DevTools Console (F12 → Console tab).
3. Fill in Name + a 10-digit phone number, click "Book My Consultation."
4. The page swaps to a thank-you message without reloading, and `window.dataLayer` will show the `consultation_form_submitted` push in the console — type `dataLayer` in the console to inspect it.

## Loom walkthrough
https://www.loom.com/share/4b4a3db542d540a0bd5203b310f9f5b0
https://www.loom.com/share/2a236cf91fc349148100a3ba8e32b490
