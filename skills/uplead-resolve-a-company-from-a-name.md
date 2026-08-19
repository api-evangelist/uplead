---
name: Resolve a company from a name, then enrich it
description: >-
  Take a messy company name, resolve it to a canonical domain and logo, then pull
  the full firmographic record — spending one credit instead of guessing at a
  domain and paying for a miss.
api: openapi/uplead-company-api-openapi.yml
operations:
  - companyNameToDomainGet
  - companyNameToDomainPost
  - companySearchGet
  - companySearchPost
  - getCompanyLogo
generated: '2026-08-13'
method: generated
source: openapi/uplead-company-api-openapi.yml + https://docs.uplead.com/
---

# Resolve a company from a name, then enrich it

Company lookups are far more accurate by domain than by name. UpLead's own
documentation says so: "For precise results it's recommended to use a domain
name." This skill gets you to a domain first.

## Before you start

- `Authorization: <api key>` header, bare, on every call.
- `companySearchGet` spends a credit per company record returned.
- The logo host is a different origin: `https://logo.uplead.com`, not
  `https://api.uplead.com/v2`.

## Steps

1. **Resolve the name to a domain.** `companyNameToDomainGet`
   (`GET /company-name-to-domain?company_name=amazon`) or
   `companyNameToDomainPost` (`POST /company-name-to-domain` with
   `{"company_name": "amazon"}`). `company_name` is required.

   The response is `{"data": {"company_name": ..., "domain": ..., "logo": ...}}`.

   Read the caveat UpLead publishes with it: the match is on **exact company
   name** and returns the most prominent company by website traffic, so a
   non-unique name can resolve to the wrong entity. Compare the returned
   `company_name` against your input before proceeding.

2. **Enrich by domain.** `companySearchGet`
   (`GET /company-search?domain=amazon.com`) or `companySearchPost`
   (`POST /company-search` with `{"domain": "amazon.com"}`).

   `companySearchGet` accepts exactly one of `domain`, `company`, or `id`. Use
   `domain`. Use `id` when you already hold an UpLead company UUID from a prior
   call — it is the only exact key.

3. **Read the record.** 32 attributes across identity, location, contact,
   firmographics (`employees`, `revenue`, `year_founded`, `type`), classification
   (`industry`, `sic_code`, `sic_description`, `naics_code`, `naics_description`),
   public markets (`ticker`, `exchange`), seven social URLs, and `alexa_rank`.

   `employees` and `revenue` are **banded strings**, not numbers —
   `10001+`, `1b+`. Do not parse them as integers.

4. **Use the logo if you are displaying the company.** The `logo` field is already
   an absolute URL. `getCompanyLogo` (`GET https://logo.uplead.com/{domain}`) is
   free and needs no key. UpLead requires attribution: a link back to uplead.com
   on any page displaying the logo, legible, at 12pt or larger — for example
   "Logos provided by UpLead". A `404` means no logo is held for that domain;
   fall back to your own placeholder.

5. **Find people at the company.** The company record has no contacts on it. Walk
   to people with `prospectorSearch` keyed on the same `domain`. See
   `skills/uplead-size-then-run-a-prospector-search.md`.

## Failure handling

- `400` — none of `domain`, `company` or `id` supplied on `companySearchGet`, or
  `company_name` missing on the name-to-domain call.
- `401` — bad key. `403` — paused account or no subscription.
- `429` — sleep `Retry-After` seconds.
- `200` with empty `data` — no match, no credit spent. Not an error.

## Do not

- Do not guess a domain from a company name yourself and call `companySearchGet`
  with it. A wrong domain that happens to match a real company still costs a
  credit and returns the wrong firmographics.
- Do not hotlink logos without the attribution UpLead requires.
