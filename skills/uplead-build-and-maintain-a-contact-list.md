---
name: Build and maintain a contact list
description: >-
  Create an UpLead contact list, add contacts to it from a prospector search,
  page through its membership, and remove or delete safely without an idempotency
  key to fall back on.
api: openapi/uplead-lists-api-openapi.yml
operations:
  - getLists
  - createList
  - getList
  - getListContacts
  - addContactsToList
  - deleteContactsFromList
  - deleteList
generated: '2026-08-13'
method: generated
source: openapi/uplead-lists-api-openapi.yml + https://docs.uplead.com/
---

# Build and maintain a contact list

Lists are the only writable objects in the UpLead API. They hold contact ids and
nothing else — there is no rename, no PATCH, and no metadata.

## Before you start

- `Authorization: <api key>` header, bare, on every call.
- The whole Lists API is **Professional/Elite/Enterprise only**. A `403` here
  means the plan, not the key.
- List operations do not spend credits. The searches that *produce* the contact
  ids do.

## Steps

1. **Look for an existing list before creating one.** Call `getLists`
   (`GET /lists?text=<name>&page=1&per_page=15`). `per_page` defaults to 15 and
   caps at 100. Match on `name` in `data.results[]`. This matters because there
   is no idempotency key on create — running create twice gives you two lists
   with the same name and different ids.

2. **Create if it is genuinely absent.** `createList` (`POST /lists`) with
   `{"name": "New List"}`. The response is
   `{"data": {"id": "<uuid>", "name": "...", "contacts_count": 0}}`. Keep the id.

3. **Get contact ids.** Run a search (`prospectorSearch`, `prospectorProSearch`,
   `quickSearch`, `personSearchGet`) and collect `data.results[].id`. These are
   the person UUIDs the list operations take. See
   `skills/uplead-size-then-run-a-prospector-search.md` for doing that without
   overspending.

4. **Add them.** `addContactsToList`
   (`POST /lists/{list_id}/contacts/add`) with
   `{"contact_ids": ["<uuid>", ...]}`. `contact_ids` is required and is an array.

5. **Verify by reading back.** `getListContacts`
   (`GET /lists/{list_id}/contacts?page=1&per_page=100`) returns
   `data.results[]` plus `data.meta`. Page with `data.meta.next_page` while
   `data.meta.last_page` is false. `text` filters by contact name, company name or
   company URL. Reading back is the substitute for the idempotency guarantee this
   API does not offer — after an ambiguous write, read, then decide.

6. **Remove selectively.** `deleteContactsFromList`
   (`DELETE /lists/{list_id}/contacts/delete`) with the same
   `{"contact_ids": [...]}` body. Note the body on a DELETE — some HTTP clients
   drop it silently.

7. **Delete the list only when you mean it.** `deleteList`
   (`DELETE /lists/{list_id}`) returns `{"data": null}`. There is no soft delete,
   no undo, and no confirmation step. Call `getList` (`GET /lists/{list_id}`)
   first and check `name` and `contacts_count` against what you expect.

## Failure handling

- `400` — `contact_ids` missing or not an array; or `name` missing on create.
- `401` — bad or missing key.
- `403` — the account's plan does not include the Lists API.
- `404` — unknown `list_id`, or a contact id that is not in the list on delete.
- `429` — back off for `Retry-After` seconds.

## Do not

- Do not retry `createList` on a timeout. Call `getLists` with a `text` filter
  first and check whether the list already landed.
- Do not treat `deleteList` as reversible.
- Do not use suppression/exclusion semantics here — `exclusion_list_names` on the
  prospector endpoints refers to lists uploaded in the application, not to lists
  built through this API.
