# Task 01 — GTM Event Schema (OrthoNow)

## Full Event Tracking Table

| Event Name | Trigger Type | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `booking_step_complete` | Custom Event (dataLayer push) | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration report, "Booking Funnel" audience |
| `booking_confirmed` | Custom Event (dataLayer push on final submit) | `clinic_location`, `specialty`, `preferred_date`, `lead_source` | Conversions report, imported into Google Ads |
| `call_now_click` | Click Trigger (Just Links / Click ID = `call-now-btn`) | `page_location`, `clinic_name` (if on clinic page), `button_position` (header/footer/sticky) | Engagement report, "High Intent" audience |
| `whatsapp_chat_open` | Click Trigger (Click URL contains `wa.me`) | `page_location`, `device_category`, `time_on_page` | Engagement report |
| `patient_guide_download` | Custom Event (dataLayer push after gated form submit) | `form_name`, `pdf_name`, `lead_phone_provided` (boolean, not actual number) | Conversions report, Lead Magnet audience |
| `clinic_page_view` | History Change / Page View Trigger filtered by URL path `/clinics/*` | `clinic_name`, `clinic_city`, `page_location` | Engagement → Pages report, Location-based remarketing audience |
| `blog_scroll_depth` | Scroll Depth Trigger (25/50/75/90%) | `percent_scrolled`, `article_title`, `article_category` | Engagement report, Content Affinity audience |

---

## Booking Form Funnel Tracking (3 Steps)

**Important note:** GTM cannot natively detect "step 2 of a form" on its own — a multi-step form is usually one single-page app state change (no URL change, no native click event tied to "step complete"). GTM only listens to what's pushed into the `dataLayer` or to DOM events it can see (clicks, URL changes, visible HTML elements). For step-level tracking, the **front-end developer must manually push to `dataLayer`** at the moment each step is validated and completed. This is a dev-side requirement, not something GTM auto-detects.

### Trigger Setup in GTM
- Create a **Custom Event trigger** named `Booking Step Complete`, listening for `event` = `booking_step_complete`.
- One trigger handles all 3 steps; GTM differentiates them using the `step_number` parameter inside the push.
- A separate **Custom Event trigger** `Booking Confirmed` listens for `event` = `booking_confirmed` (final submit).

### dataLayer Push — Step 1 (Clinic + Specialty selected)
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint Care"
}
```

### dataLayer Push — Step 2 (Contact details entered)
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "Indiranagar - Bengaluru",
  "preferred_date": "2026-07-04"
}
```

### dataLayer Push — Step 3 (Booking confirmed)
```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar - Bengaluru",
  "specialty": "Knee & Joint Care",
  "preferred_date": "2026-07-04"
}
```

### Surfacing Drop-off in GA4 Funnel Exploration
1. In GA4 → Explore → create a **Funnel Exploration**.
2. Add 3 steps using the `booking_step_complete` event filtered by `step_number = 1`, `step_number = 2`, and the `booking_confirmed` event as the final step.
3. Enable "Show elapsed time" to see how long users take between steps.
4. GA4 will automatically show the % drop-off between each step (e.g., 100% → 62% → 41%), which tells the marketing team exactly where users abandon — most commonly between step 1 and step 2 in appointment forms.
5. Breakdown the funnel by `clinic_location` dimension to see if drop-off is worse for specific clinics (could indicate slow-loading clinic-specific availability data).

---

## Which Conversion to Import into Google Ads

**Recommended: `booking_confirmed`** (not `call_now_click` or `patient_guide_download`).

**Why:** `booking_confirmed` is the closest dataLayer event to an actual paying patient — it represents a completed appointment, not just intent. `call_now_click` is tempting because phone calls are common in healthcare, but it's a weak signal on its own — a click on "Call Now" doesn't confirm the call connected or led to a booking, so it inflates conversion volume with low-quality data and would mislead the campaign's optimization (Google Ads' Smart Bidding would start optimizing for clicks, not patients). `patient_guide_download` is a top-of-funnel lead magnet, not a sales-qualified action. `booking_confirmed` gives Google Ads Smart Bidding a clean, high-intent signal to optimize toward — which directly improves the 2.1% conversion problem the client is trying to fix.
