# Task 03 — Integration Design (OrthoNow: Landing Page → HubSpot → WhatsApp → Google Ads)

## End-to-end architecture

I would use a **direct API call from the landing page's serverless backend, not Zapier/Make and not HubSpot's native embedded form.** Here's the order of operations:

1. **Form submit (browser)** — JS captures Name + Phone, fires the `consultation_form_submitted` dataLayer push (for GTM/GA4/Google Ads), and sends the data via `fetch()` to a lightweight serverless function (Vercel/Cloud Function) acting as our own middleware endpoint — not directly to HubSpot from the client.
2. **Serverless function** — this single endpoint does three things in sequence:
   - Calls the **HubSpot Contacts API** (`POST /crm/v3/objects/contacts`) using phone as the dedup key (see below), setting Name, Phone, Clinic Preference, Source = "Google Ads - Consultation Landing Page", Lead Status = "New Enquiry".
   - Calls **Karix's WhatsApp Business API** to send the confirmation template message.
   - Logs the event to our own database/sheet as a fallback record before responding success to the browser.
3. **Google Ads conversion** fires client-side via the GTM tag already wired to the `consultation_form_submitted` trigger — this doesn't depend on the backend call succeeding, so a CRM/WhatsApp failure never costs us the ad spend optimization signal.

**Why a direct API call over Zapier/Make:** Zapier/Make add 1-3 minutes of polling-based latency by default (their non-instant triggers run on a schedule), which directly threatens the 2-minute WhatsApp SLA. A direct API call from a serverless function executes in under a second and gives us full control over retries, logging, and error handling — Zapier abstracts those away exactly when we need visibility into failures.

## Biggest failure point + fallback

The biggest risk is **silent failure of the HubSpot or Karix API call** — the browser shows "thank you" successfully, but the backend call could fail (HubSpot rate limit, Karix template rejected, network blip), and nobody on the OrthoNow team would know a lead was lost. **Fallback:** the serverless function writes every submission to a simple persistent log (e.g., Google Sheet or DB row) *before* attempting the HubSpot/Karix calls, marked "pending." If either call fails, it retries twice with exponential backoff; if it still fails, it triggers a Slack alert to the ops team with the lead's phone number so they can manually follow up within minutes, and flags the row "failed — manual action needed." This guarantees no lead silently disappears even if the integration breaks.

## What could break the 2-minute WhatsApp SLA, and monitoring

Things that could break the SLA: Karix API downtime, the WhatsApp template message being unapproved/rejected by Meta for that send, the serverless function cold-starting on its first call after idle (adds 1-2 seconds, usually fine, but compounds with other delays), or a backlog if many leads submit simultaneously during a campaign spike. **Monitoring:** I'd log a timestamp at "form submitted" and at "WhatsApp API confirmed sent," and run a scheduled check (every 5 minutes) that flags any record where the gap exceeds 2 minutes or where the WhatsApp status is still "pending." This feeds a simple dashboard (or Slack alert) so the team catches SLA breaches the same day, not when a patient complains weeks later.

---

## The phone number deduplication trap

**HubSpot's default deduplication key is email, not phone.** Since OrthoNow's form only collects Name + Phone (no email), if I create contacts using the standard `POST /crm/v3/objects/contacts` call without handling this explicitly, two patients submitting with the same phone number but different names will create **two separate contact records** — HubSpot won't recognize them as the same person.

**My fix:** Before creating a contact, I'd first call `GET /crm/v3/objects/contacts/search` filtering on the `phone` property (or use a custom unique property `phone_normalized` — storing the phone in a single consistent format like `+91XXXXXXXXXX` to avoid format mismatches like spaces or missing country codes). If a match is found, I update that existing contact (overwrite name only if the existing one looks like a placeholder, otherwise log both names as a note) and update Lead Status/Clinic Preference. If no match, I create a new contact. This has to be a custom dedup check built into the integration logic — it does not happen automatically just because two records share a phone number.
