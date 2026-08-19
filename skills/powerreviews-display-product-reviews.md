---
name: Render reviews and rating snippets for product pages
description: Fetch the review list for one product and aggregate star-rating snippets for many products, with sorting and paging, to render user-generated content in a custom storefront or mobile app.
api: openapi/powerreviews-readservices-openapi.yml
operations: [getProductReviewsUsingGET, getProductSnippetsUsingGET, getQuestionsUsingGET, getAnswersUsingGET, getConfigurationByMerchantUsingGET]
generated: '2026-08-13'
method: generated
source: https://developers.powerreviews.com/Content/Read%20API/Use%20Cases.htm
---

# Render reviews and rating snippets for product pages

Use this when you are building the product experience yourself instead of
dropping in the PowerReviews display widget from `ui.powerreviews.com`.

## Before you start

- ReadServices API key + merchant id, requested from `support@powerreviews.com`.
- Base URL: `https://readservices-b2c.powerreviews.com`
- Every path is `/m/{merchantId}` and most also carry `/l/{locale}`, e.g. `en_US`.

## Listing pages: one call for many products

`getProductSnippetsUsingGET` takes a **comma-joined list** of page ids, so a
category or search page is one request, not one per tile:

```
GET /m/{merchantId}/l/{locale}/product/{pageIds}/snippet?apikey={key}
```

Use this for star ratings and review counts on grids. Do not call the full
review endpoint per tile.

## Product detail page: the review list

```
GET /m/{merchantId}/l/{locale}/product/{pageId}/reviews
    ?apikey={key}
    &paging.size=10
    &sort=Newest
```

`sort` accepts `HighestRating`, `LowestRating`, `MostHelpful`, `Oldest`,
`Newest`. A "top reviews" module is just `sort=HighestRating&paging.size=5`.
`image_only=true` restricts to reviews carrying visual content.

Paging is `paging.size` (**1–25**) plus `paging.from`, and
`paging.size + paging.from` must stay under **10,000**.

## Questions and answers

```
GET /m/{merchantId}/l/{locale}/product/{pageId}/questions?apikey={key}&paging.size=10
GET /m/{merchantId}/l/{locale}/question/{questionId}/answers?apikey={key}
```

Answers are a separate call per question — fetch lazily on expand rather than
eagerly for every question on the page.

## Merchant display configuration

```
GET /m/{merchant_id}/l/{locale}/configuration?apikey={key}
```

Returns `ConfigurationResponse` with the merchant's `features`, `localizations`
and `properties`. All three are untyped objects in the spec; read a live response
to learn the shape. Fetch this once per locale and cache it — it changes rarely.

## Cache, do not call per page load

PowerReviews explicitly recommends caching responses rather than calling the API
on every page render, both for page speed and because the Read API blocks an IP
that exceeds **1,800 requests per 5 minutes**. There is no rate-limit header and
no 429 to warn you — a server-side cache in front of these calls is not optional
at storefront traffic.

## Errors

`{"url":..., "message":..., "status_code":...}`. 401 for a missing or wrong key,
403/404 for a bad merchant, locale, or page id — note that a page id with no
PowerReviews content yet also lands on 404, so treat it as "no reviews", not as a
failure.
