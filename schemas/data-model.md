# Data Model — Financial Planner App

## Where things live in Firestore

Transactions are a top-level collection; every transaction carries a `userId` field and the
security rules (`budgetly_backend_nodejs/firestore.rules`) check it (SCRUM-109). Everything else
sits under the user's own document, so the rules check the `{uid}` in the path.

```
transactions/{transactionId}         Transaction (owner in the userId field)
users/{uid}                          User: profile and settings
users/{uid}/imports/{importId}       Statement (import record)
users/{uid}/budgets/{categoryId}     Budget: one per category
users/{uid}/merchantCache/{docId}    Merchant cache (categories.md)
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

### Transaction — `transactions/{transactionId}`
| Field | Type | Notes |
|---|---|---|
| id | string | The document ID (below), not a field |
| userId | string | **Required.** The owner's Firebase Auth uid. Set on create from the verified auth token, never from client input. Never changes |
| date | timestamp | Transaction date |
| merchant | string | Raw merchant name from statement |
| amount | number | Negative = spending, positive = income |
| category | string | A category id, e.g. "dining", "groceries" (list in categories.md) |
| categoryConfidence | number | null | 0–1; null until something categorizes it (categories.md) |
| categorySource | string | "none", "rule", "ai", "cache" or "user" (categories.md) |
| isUserCorrected | boolean | True if user manually changed the category |
| accountId | string | null | Which of the user's accounts the statement was for |
| sourceStatementId | string | null | The importId of the statement it came from; null for manual ones |
| source | string | "import" (written by the import function) or "manual" (added by the user) |
| deletedAt | timestamp | null | **Required.** Created as null; set when the user deletes it (soft delete) |
| deletedBy | string | null | uid of whoever deleted it, or null |
| purgeAfter | timestamp | null | When the server may permanently remove it, or null |

**Document IDs**
- Imported: `{uid}_{importId}_{rowNumber}` (row number zero-padded to 5 digits), so re-running an
  import overwrites the same documents instead of duplicating them.
- Manual: a Firestore auto-generated ID.

**Queries the app runs.** Every client query starts with `where("userId", "==", uid)`, or the
rules reject it.
| Screen | Query |
|---|---|
| Transactions list | `userId == uid`, `deletedAt == null`, `orderBy("date", "desc")` |
| One import's transactions | `userId == uid`, `sourceStatementId == importId` |
| Recently deleted | `userId == uid`, `deletedAt != null` |

Server only (Admin SDK, which skips the rules): the purge job (`purgeAfter <= now`) and the
admin fraud scan across all users.

**Rules this needs**
- Read: `resource.data.userId == request.auth.uid`.
- Create: `request.resource.data.userId == request.auth.uid`.
- Update: `userId` can't change.
- Delete: never from the client (soft delete with `deletedAt` instead; the server purges).

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
- Access to transactions is controlled by their `userId` field; everything else is under
  `users/{uid}`, and the rules check `request.auth.uid == uid`.
- Categories are a fixed list (categories.md). Users can add their own `custom-...` categories;
  those are kept in the browser for now and would move to `users/{uid}/categories/{id}`.