---
name: Submit a review server-to-server
description: Retrieve the locale-aware write-a-review template for a product, then submit a review through the PowerReviews Write API with the required iOvation device fingerprint.
api: openapi/powerreviews-writeservices-openapi.yml
operations: [startReviewUsingGET, submitReviewUsingPOST]
generated: '2026-08-13'
method: generated
source: https://developers.powerreviews.com/Content/Write%20API/Use%20Cases.htm
---

# Submit a review server-to-server

Use this when you collect review content in your own experience — a sampling
campaign, a post-purchase survey, an app form — and need it to land in
PowerReviews.

## Before you start

- You need a **WriteServices** API key. It is a *different* credential from the
  ReadServices key. Email `support@powerreviews.com`, or
  `sampling@powerreviews.com` if you are an agency running a campaign for a
  brand — the campaign path also gets you the page ids and account credentials.
- Base URL: `https://writeservices.powerreviews.com`
- Auth: `apikey={key}` as a query parameter.

## Step 1 — always fetch the template first

```
GET /api/b2b/writereview/review_template
    ?apikey={key}
    &merchant_id={merchant_id}
    &page_id={page_id}
    &locale=en_US
    &merchant_user_id={your_user_id}
```

`startReviewUsingGET` returns `B2BReviewData`: `context_information`,
`started_timestamp`, and a `fields[]` array of review-form fields with `id`,
`key`, `label`, `group`, `helper_text`, `required` and `hidden`.

PowerReviews documents calling this **on every submission**, not caching it. The
brand controls the field set and may require fields beyond the mandatory ones;
fetching the template each time is how you stay in sync with what that brand
currently requires.

Note the concrete field subtypes (`SimpleReviewField`, `CompositeReviewField`,
`CollectionReviewField`) are declared with no properties in the Swagger document,
so read the live response rather than the contract.

## Step 2 — collect the mandatory fields

PowerReviews publishes five fields your form or survey must always carry:

- **Headline** — the review title
- **Rating** — the star rating
- **Comments** — the review body
- **Location** — reviewer location
- **Nickname** — reviewer display name

Plus anything the template marks `required: true` for that brand.

## Step 3 — get an iOvation BlackBox

The device fingerprint is a **required field** for API review submission. Load
the iOvation snare script in the browser where the review is collected:

- Production: `https://mpsnare.iesnare.com/snare.js`
- Test accounts: `https://ci-mpsnare.iovation.com/snare.js`

Read the value from `ioGetBlackbox().blackbox` once the page has loaded and carry
it through to your server as `iovation_black_box`.

This is a hard constraint on automation: a headless agent with no browser cannot
produce a BlackBox, so review submission cannot be fully unattended. It also
matters downstream — content without iOvation data is not suitable for Open
Syndication Network syndication.

## Step 4 — submit

```
POST /api/b2b/writereview/submit_review
    ?apikey={key}&merchant_id={merchant_id}&page_id={page_id}&locale=en_US
Content-Type: application/json
```

Body is `WriteAReviewB2BPostRequest`: `fields[]` filled from the template, plus
`iovation_black_box`, `ip_address`, `started_timestamp`, `submitted_timestamp`,
`order_id`, `campaign_id`, `source`, `reviewer_type` and `disclosure_code` where
they apply. Use `disclosure_code` when the review came from a sampling or
incentivised program.

## Retries — there is no idempotency key

PowerReviews publishes **no** idempotency header, no request-id echo and no
replay semantics. A retry after a timeout can create a duplicate review. Track
your own `unique_review_id` per submission and check before you retry. Do not
blind-retry on a network error.

## After submission

Content arrives in **pending** status and is moderated before it appears
publicly. Reviews submitted via the API are flagged as such in PowerReviews'
audit trail. Send a single test review and get confirmation from PowerReviews
before pushing a campaign's full volume — that confirmation step is part of their
documented integration process.

## Errors

The Write API returns `B2BResponse` with `error_code`, `message`, `status_code`
and a `details[]` array of `ErrorMessage`. 401 means a missing key or a
ReadServices key used here; 403/404 mean the merchant, site or page id is not in
scope for your key.
