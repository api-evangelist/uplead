---
name: Size a prospect segment before paying for it
description: >-
  Use count_only to price an UpLead prospector query for free, narrow the filters
  until the segment is the right size, then page through the results spending the
  minimum number of credits.
api: openapi/uplead-prospector-api-openapi.yml
operations:
  - prospectorSearch
  - prospectorProSearch
  - getCredits
generated: '2026-08-13'
method: generated
source: openapi/uplead-prospector-api-openapi.yml + https://docs.uplead.com/
---

# Size a prospect segment before paying for it

Prospector search bills per contact returned. The single most important habit
with this API is to count first and fetch second.

## Before you start

- `Authorization: <api key>` header on every call, bare, no `Bearer`.
- `prospectorProSearch` (`POST /prospector-pro-search`) is
  Professional/Elite/Enterprise only. On a lower plan it returns `403` even
  though the key is valid. `prospectorSearch` (`POST /prospector-search`) is the
  broader-availability endpoint.

## Steps

1. **Count for free.** Call `prospectorSearch` with `count_only: true` and your
   filters. The response is `{"data": {"count": <int>}}` and costs nothing.

   Minimum viable filter set for `prospectorSearch`: `domain` is **required**.
   Narrow with `job_function`, `job_sub_function`, `management_level`, `title` /
   `titles` + `title_search_mode`, `city`/`cities`, `state`/`states`,
   `country`/`countries`, `regions`, and `email_status`.

2. **Narrow until the count is affordable.** Each contact returned with
   `email_status` of `valid` or `accept_all` costs one credit. A count of 507 is a
   potential spend of 507 credits. Tighten `management_level` (`M`, `D`, `VP`,
   `C`, `CX`) or set `email_status: valid` to cut the segment before fetching.

3. **Exclude what you already own.** Upload suppression lists in the application
   first, then name them in `exclusion_list_names` so you do not re-pay for
   contacts you already have. (Re-purchasing the *same* record is not double
   billed, but excluding is cleaner and faster.)

4. **Fetch a page.** Drop `count_only`, set `page: 1` and `per_page` (default 25,
   max 100). The response carries `data.results[]` and `data.meta`.

5. **Page with the meta block, not a guess.** Loop while `data.meta.last_page` is
   false, using `data.meta.next_page` as the next `page`. Stop as soon as you have
   what you need — every additional page is more credits.

6. **Track spend per page.** `data.userInfo.availableCredits` is returned on every
   page. If it stops moving, the rows you are getting back are `unknown` or
   `invalid` and are free.

## Using the Pro endpoint

`prospectorProSearch` takes the same shape but a much wider filter set and
returns each contact with the **full company object nested** under `company`,
which saves a separate company lookup. Extra filters worth knowing:

- `domains`, `industries`, `industries_ids`, `sic_codes`, `naics_codes`
- `employees` (`1-10` … `10001+`), `revenues` (`0-1m` … `1b+`)
- `business_types` (`education`, `government`, `non_profit`, `public`, `private`, `subsidiary`)
- `regions` (`AMER`, `APAC`, `EMEA`, `LATAM`) and `location_target` (`contact` or `company`)
- `exclude_eu` to drop EU contacts entirely

`industries_ids` takes UpLead's own numeric industry ids — resolve them first
with `searchIndustriesGet` (`GET /industries?text=...`). The `industry` string on
a record is a display name and is not the same key.

## Failure handling

- `400` — `domain` missing on `prospectorSearch`, or an out-of-range enum value.
- `403` — plan gate, not a bad key. Fall back to `prospectorSearch`.
- `429` — 500 req/min exceeded; sleep `Retry-After` seconds.
- `200` with an empty `results[]` — the segment is genuinely empty. Free.

## Do not

- Do not fetch a page just to find out how big the segment is. That is what
  `count_only` is for, and it is free.
- Do not invent `industries_ids` values. Resolve them through the Industries API.
