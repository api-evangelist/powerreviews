---
name: Post merchant responses and Q&A content
description: Reply publicly to a review as the merchant, and submit questions and answers through the PowerReviews Write API.
api: openapi/powerreviews-writeservices-openapi.yml
operations: [submitMerchantResponseUsingPOST, submitQuestionUsingPOST, submitAnswerUsingPOST, getAllReviewsUsingGET]
generated: '2026-08-13'
method: generated
source: https://developers.powerreviews.com/Content/reference/write.html
---

# Post merchant responses and Q&A content

Use this to close the loop on review content — reply to a review as the brand,
or seed and answer product questions.

## Before you start

- WriteServices API key, base URL `https://writeservices.powerreviews.com`,
  `apikey={key}` on the query string.
- To reply to a review you need its `review_ugc_id`. Find it by pulling reviews
  through the Read API (`getAllReviewsUsingGET`) — see the CRM sync skill.

## Reply to a review

```
POST /api/b2b/merchant-response
    ?apikey={key}&merchant_id={merchant_id}&page_id={page_id}&locale=en_US
Content-Type: application/json

{
  "review_ugc_id": 0,
  "text": "...",
  "author_name": "...",
  "author_email": "...",
  "author_location": "..."
}
```

`MerchantResponseData` is the body. Returns `B2BResponse`. Merchant-response
endpoints were added to the API in January 2019 and are the newest capability
PowerReviews has shipped on this surface.

## Submit a question

```
POST /api/b2b/question
    ?apikey={key}&merchant_id={merchant_id}&page_id={page_id}&locale=en_US
```

`QuestionData` carries `question_text`, `question_type`,
`merchant_question_category`, the author fields, `merchant_user_id`, your own
`merchant_question_id`, and `iovation_black_box`. Set `is_seeded: true` when the
question is brand-authored rather than shopper-submitted — be accurate here, it
is a content-authenticity signal.

Returns `QuestionResponse`, which includes `merchant_information` and
`product_information` alongside the question.

## Submit an answer

```
POST /api/b2b/answer
    ?apikey={key}&merchant_id={merchant_id}&page_id={page_id}&locale=en_US
```

`AnswerData` carries `question_id`, `answer_text`, `answer_source`, the author
fields, and the flags `is_expert`, `is_verified_buyer`, `is_seeded`, `is_import`
and `is_eligible_for_notification`. Returns `AnswerResponse`.

Note `question_id` is an **integer** here, while the Read API takes `questionId`
as a **string** path segment. Convert deliberately rather than passing values
through untouched.

## Operating rules

- **Carry the iOvation BlackBox** on question and answer submissions —
  `iovation_black_box` is on both `QuestionData` and `AnswerData` and feeds
  PowerReviews' fraud screening.
- **No idempotency.** None of these POSTs accept an idempotency key. Track your
  own `merchant_question_id` and check for an existing record before retrying, or
  you will post duplicate public content under the brand's name.
- **Everything is moderated.** Content lands pending and is reviewed before it
  appears.
- **Set the flags honestly.** `is_seeded`, `is_expert` and `is_verified_buyer`
  are disclosure signals, not formatting options.
