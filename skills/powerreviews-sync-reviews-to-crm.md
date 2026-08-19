---
name: Sync PowerReviews review content into a downstream system
description: Incrementally pull all reviews for a merchant that changed since a timestamp, paging safely inside PowerReviews' published bounds, for CRM, analytics or search indexing.
api: openapi/powerreviews-readservices-openapi.yml
operations: [getAllReviewsUsingGET, getAllQuestionsUsingGET, getAnswersUsingGET]
generated: '2026-08-13'
method: generated
source: https://developers.powerreviews.com/Content/Read%20API/Use%20Cases.htm
---

# Sync PowerReviews review content into a downstream system

Use this when you need PowerReviews review content in something that is not a
product page — a CRM, a warehouse, a search index, a sentiment model.

## Before you start

- You need a **ReadServices** API key and the **merchant id**. Keys are not
  self-service: email `support@powerreviews.com` with the brand you support and
  what the integration needs. The ReadServices key will not work against the
  Write API.
- Base URL: `https://readservices-b2c.powerreviews.com`
- Auth: append `apikey=<key>` as a **query parameter** on every request. Nothing
  goes in a header.

## Steps

1. **Pick your watermark.** Keep the highest `updated` timestamp you have already
   ingested, as **epoch milliseconds**. On a cold start use the beginning of the
   window you care about.

2. **Pull the first page** with `getAllReviewsUsingGET`:

   ```
   GET /m/{merchantId}/reviews
       ?apikey={key}
       &date={epoch_millis}
       &updated_date_query=true
       &paging.size=25
   ```

   `date` is **required** on this operation. `updated_date_query=true` makes
   `date` apply to the last-updated timestamp rather than the created timestamp —
   that is what makes this a change feed rather than a backfill. Drop it if you
   want created-date semantics instead.

3. **Page with `paging.from`.** Increment the offset by your page size:
   `&paging.size=25&paging.from=25`, then `50`, and so on. Read
   `paging.total_results` and `paging.pages_total` off the `PagingResponse` to
   know when to stop.

   **Hard bounds, both published:** `paging.size` must be between **1 and 25**,
   and `paging.size + paging.from` must not exceed **10,000**. Going over either
   returns an error. If a merchant has more than 10,000 changed reviews, do not
   try to page past the window — advance the `date` watermark to the newest
   record you received and start a fresh scan.

4. **Do the same for Q&A** if you need it: `getAllQuestionsUsingGET` on
   `/m/{merchantId}/questions` (also requires `date`), then
   `getAnswersUsingGET` on
   `/m/{merchantId}/l/{locale}/question/{questionId}/answers` per question.

5. **Advance the watermark** to the newest `updated` value you actually ingested,
   not to "now" — a record updated during your scan can otherwise be missed.

## Rate limits — read this before you parallelise

PowerReviews blocks an IP that sends more than **1,800 requests in 5 minutes**,
for 5 minutes, and keeps extending the block while the traffic continues. There
is **no** `RateLimit-*` header, **no** `Retry-After` and **no** 429. You will get
no warning and no signal — the requests simply stop working. Budget your own
client-side rate limiting, and cache.

## Handling the response

`QueryResponse.results` is an **untyped array** in the Swagger document — the
review shape is not in the contract. Read one live response and pin your mapping
to what you observe; do not assume fields are guaranteed. Product `UPC` and
`GTIN` appear on `review.details` where available.

## Errors

Errors are `{"url":..., "message":..., "status_code":...}` — not RFC 9457, and
there is no stable error code to switch on.

- `401 api key is required for authentication` — you omitted `apikey`.
- `401 api key (...) is not authenticated for this request` — wrong key for this
  merchant, or a WriteServices key against the Read API.
- `403` / `404` — check `merchantId`, `locale`, and `pageId`.

See `errors/powerreviews-problem-types.yml` and
`conventions/powerreviews-conventions.yml`.
