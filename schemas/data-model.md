# Data Model — Financial Planner App

## Where things live in Firestore

Every user's data sits under their own user document, so the security rules
(`budgetly_backend_nodejs/firestore.rules`) can give each user access to only their own data.

```
users/{uid}                          User: profile and settings
users/{uid}/transactions/{id}        Transaction
users/{uid}/imports/{importId}       Statement (import record)
users/{uid}/budgets/{categoryId}     Budget: one per category
```

Raw statement files are in Firebase Storage at `users/{uid}/imports/{timestamp}_{fileName}`.

## Entities

### User (profile and settings) — `users/{uid}`
| Field | Type | Who writes it | Notes |
|---|---|---|---|
| uid | string | | Firebase Auth user ID (the document ID, not a field) |
| email | string | signup | Read-only after signup |
| displayName | string | signup, Settings | 1–60 characters |
| createdAt | timestamp | signup | |
| consentAcceptedAt | timestamp | consent screen | Set once the user accepts the data consent screen |
| monthlyIncome | number | null | Settings | Dollars per month, 0 to 10,000,000, or null/missing = not set. Used as the budget for a month where no money came in (for example when only a credit card statement is imported). A month with real money in always uses that instead |

Rules: the user can read only their own document, and can change only `displayName`,
`consentAcceptedAt` and `monthlyIncome`, with the limits above (SCRUM-88).

### Transaction
| Field | Type | Notes |
|---|---|---|
| id | string | Auto-generated document ID |
| userId | string | Owner reference |
| date | timestamp | Transaction date |
| merchant | string | Raw merchant name from statement |
| amount | number | Negative = spending, positive = income |
| category | string | A category id, e.g. "dining", "groceries" (list in categories.md) |
| categoryConfidence | number | null | 0–1; null until something categorizes it (categories.md) |
| categorySource | string | "none", "rule", "ai", "cache" or "user" (categories.md) |
| isUserCorrected | boolean | True if user manually changed the category |
| accountId | string | null | Which of the user's accounts the statement was for |
| sourceStatementId | string | Reference to the imported statement |

### Statement
| Field | Type | Notes |
|---|---|---|
| id | string | Auto-generated document ID |
| userId | string | Owner reference |
| uploadedAt | timestamp | |
| fileType | string | "csv" or "pdf" |
| status | string | "processing", "completed", "failed" |
| transactionCount | number | How many transactions were extracted |

### Category
| Field | Type | Notes |
|---|---|---|
| id | string | e.g. "dining", "groceries" |
| label | string | Display name |
| icon | string | Icon identifier for UI |
| color | string | Hex color for charts/progress bars |

### Budget — `users/{uid}/budgets/{categoryId}`
| Field | Type | Notes |
|---|---|---|
| categoryId | string | Same as the document ID, e.g. "dining" (one budget per category) |
| limit | number | Monthly limit in dollars, 0 or more |
| updatedAt | timestamp | Set on every save |

- A budget is a **monthly** limit that applies to every month: the app compares it with
  whichever month is being shown. There is no per-month copy and no stored "spent" total;
  spending is always added up from the transactions, so it can't get out of sync.
- `income`, `transfers` and `uncategorized` can't have a budget (categories.md).
- Rules: only the owner can read, create, change or delete their budgets, with exactly
  these fields (SCRUM-81, SCRUM-88).

## Relationships
- One **User** → many **Transactions**, **Statements**, **Budgets**
- One **Statement** → many **Transactions** (the ones extracted from it)
- One **Category** → many **Transactions** and **Budgets**

## Notes
- Access is controlled by the path: everything is under `users/{uid}`, and the rules check
  `request.auth.uid == uid`. Transactions also carry a `userId` field.
- Categories are a fixed list (categories.md). Users can add their own `custom-...` categories;
  those are kept in the browser for now and would move to `users/{uid}/categories/{id}`.