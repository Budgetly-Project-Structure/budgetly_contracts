# AI categorization interface (SCRUM-69)

What the categorization step sends to the AI model and what it must get back.
The Cloud Function (SCRUM-71) builds the request, calls the model, validates the reply and falls
back to Uncategorized when anything is wrong (SCRUM-72). The model provider is a detail behind
this interface: any model that can return this JSON works.

## When it runs

During a statement import, after parsing and before the Firestore write (SCRUM-73). Only
transactions with `categorySource == "none"` are sent; rule matches (card payments) and anything
the user already categorized are skipped.

## Request

One request per batch of **at most 50 transactions** (SCRUM-74 splits larger imports).

```json
{
  "categories": [
    { "id": "dining", "label": "Dining Out" },
    { "id": "groceries", "label": "Groceries" },
    { "id": "transportation", "label": "Transportation" },
    { "id": "subscriptions", "label": "Subscriptions" },
    { "id": "shopping", "label": "Shopping" },
    { "id": "utilities", "label": "Utilities" },
    { "id": "entertainment", "label": "Entertainment" },
    { "id": "health", "label": "Health" },
    { "id": "income", "label": "Income" },
    { "id": "transfers", "label": "Transfer" }
  ],
  "transactions": [
    { "id": "t1", "merchant": "STARBUCKS STORE 08812", "amount": -6.72, "date": "2026-08-29" },
    { "id": "t2", "merchant": "ACME CORP PAYROLL", "amount": 1850.00, "date": "2026-09-01" }
  ]
}
```

| Field | Notes |
|---|---|
| `categories` | The "AI can assign" categories from `schemas/categories.md`. Never `uncategorized`, never custom categories |
| `transactions[].id` | A short id **for this request only** (`t1`, `t2`, ...), not the Firestore id, so no real ids leave the backend |
| `transactions[].merchant` | Raw description from the statement, already masked (no account numbers) |
| `transactions[].amount` | Dollars. Negative = money out, positive = money in |
| `transactions[].date` | `YYYY-MM-DD` |

Nothing else is sent: no user id, name, email, account numbers or balances.

## Response

```json
{
  "results": [
    { "id": "t1", "category": "dining", "confidence": 0.96 },
    { "id": "t2", "category": "income", "confidence": 0.99 }
  ]
}
```

| Field | Rule |
|---|---|
| `results[].id` | Must match an id from the request |
| `results[].category` | Must be one of the request's category ids |
| `results[].confidence` | Number from 0 to 1. Below 0.7 the web app flags the transaction for review |

## Validation and fallback (SCRUM-72)

The function checks every result. A transaction falls back to
`{ category: "uncategorized", categoryConfidence: null, categorySource: "none" }` when:

- the model call fails, is rate limited, or takes longer than **20 seconds** for the batch
- the reply isn't valid JSON or doesn't have a `results` array → the whole batch falls back
- a result has an unknown id, an unknown category, or a confidence that isn't a number from 0 to 1 → just that transaction falls back
- a transaction has no result → that transaction falls back
- an id appears twice → the first result is used

Categorization never fails an import. The worst case is that some or all transactions stay
Uncategorized, exactly as before this feature.

Valid results are saved as `{ category, categoryConfidence: confidence, categorySource: "ai" }`.

## Sign check

The model sees the amount, but the function also checks one thing itself: `income` is only
accepted for money in (`amount > 0`). An `income` answer for money out falls back to
Uncategorized. Spending categories are accepted either way, because a positive amount in a
spending category is a refund.
