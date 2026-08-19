---
name: Enrich a business email into a contact and company
description: >-
  Turn a single business email address into a verified contact record plus the
  full company record behind it, using UpLead's Combined API, without spending a
  credit on an unverifiable email.
api: openapi/uplead-combined-api-openapi.yml
operations:
  - combinedSearchGet
  - combinedSearchPost
  - getCredits
generated: '2026-08-13'
method: generated
source: openapi/uplead-combined-api-openapi.yml + https://docs.uplead.com/
---

# Enrich a business email into a contact and company

Use this when you have a work email address and need both the person and their
employer in one call.

## Before you start

- Base URL: `https://api.uplead.com/v2/`
- Send the API key **bare** in the `Authorization` header — `Authorization: myapikey`.
  There is no `Bearer` prefix. Every call needs it, including `getCredits`.
- This call spends credits. Read `conventions/uplead-conventions.yml#billing_semantics`
  before running it in a loop.

## Steps

1. **Check the balance.** Call `getCredits` (`GET /credits`). The response is
   `{"data": {"email": ..., "credits": <int>}}`. If `credits` is 0, stop — every
   search below will return nothing useful.

2. **Enrich the email.** Call `combinedSearchGet` (`GET /combined-search?email=...`)
   or `combinedSearchPost` (`POST /combined-search` with `{"email": "..."}`).
   `email` is the only parameter and it is required.

3. **Read the result.** On 200 you get
   `{"data": {...person fields..., "company": {...company fields...}}, "userInfo": {"availableCredits": <int>}}`.
   The company is embedded whole — do **not** make a second `companySearchGet`
   call for it, that would spend another credit for data you already hold.

4. **Check `email_status` before you trust or bill the record.**
   - `valid` — real inbox. Charged.
   - `accept_all` — the domain affirms everything, so it cannot be fully verified. Charged.
   - `unknown` — verification did not complete. Not charged. Treat as unusable for send.
   - `invalid` — not a real email. Not charged.

5. **Track spend.** `data.userInfo.availableCredits` on this response is the running
   balance. Carry it forward instead of calling `getCredits` again.

## Failure handling

- `400` — the `email` parameter is missing or malformed. Not retryable as-is.
- `401` — the key is wrong or absent. Check the `Authorization` header has no `Bearer` prefix.
- `403` — the key is fine but the account is paused or has no subscription.
- `429` — over 500 requests/minute. Sleep for the `Retry-After` value in seconds, then retry.
- `50X` — retry with backoff; check <https://status.uplead.com/> before escalating.
- **200 with an empty `data`** is the normal "no match" outcome, not an error, and
  costs nothing. Test the payload, not just the status.

## Do not

- Do not retry a search on an ambiguous timeout expecting idempotency — UpLead
  documents no idempotency key. A duplicate hit on the *same* record is not
  double-billed, but a different record is.
- Do not call `personSearchGet` and `companySearchGet` separately when one email
  is all you have. `combinedSearchGet` is the single-credit path.
