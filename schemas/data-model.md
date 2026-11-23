# Data Model — Financial Planner App

## Where things live in Firestore

Transactions and accounts are top-level collections; every document in them carries a `userId`
field and the security rules (`budgetly_backend_nodejs/firestore.rules`) check it (SCRUM-109,
Sprint 5 #17). Everything else sits under the user's own document, so the rules check the
`{uid}` in the path.

```
transactions/{transactionId}         Transaction (owner in the userId field)
accounts/{accountId}                 Account: a checking account or credit card (owner in the userId field)
users/{uid}                          User: profile and settings
users/{uid}/imports/{importId}       Statement (import record)
users/{uid}/budgets/{categoryId}     Budget: one per category
users/{uid}/merchantCache/{docId}    Merchant cache (categories.md)
```

Raw statement files are in Firebase Storage at `users/{uid}/imports/{timestamp}_{fileName}`.

## Money is stored in whole cents (Sprint 5)

Every amount of money is saved as an **integer number of cents**, never as a dollar
number with a decimal point:

| Where | Field | Meaning |
|---|---|---|
| Transaction | `amountCents` | Integer. Negative = money out (spending), positive = money in. $12.34 spent = `-1234` |
| Budget | `limitCents` | Integer, 0 or more. A $250 monthly limit = `25000` |

**Converting.** On the way in, `cents = Math.round(dollars * 100)`: a number a user types,
or a statement amount when it can't be read straight from the text. The import reads
statement text like `"1,234.56"` or `"(45.00)"` directly into cents (`123456`, `-4500`),
without going through a dollar number at all. Everything is added up, compared and stored in
cents. Dollars exist **only for display**: `(cents / 100)` formatted as currency right before
it's shown.

**Why.** Computers store numbers with decimals in binary, and most decimal fractions,
like 0.1, can't be stored exactly. In JavaScript `0.1 + 0.2` is `0.30000000000000004`, not
`0.3`. Adding up hundreds of transactions in dollars piles up these tiny errors, so totals
can be off by a cent, `===` comparisons fail (the duplicate check compares amounts), and a
budget can look over or under by $0.01. Integers are exact: `10 + 20 === 30` always, so
sums and comparisons in cents never drift.

**Deprecated, removed after the migration.** The old dollar fields, transaction `amount`
and budget `limit`, are **deprecated**. Until the web app reads cents (Sprint 5 #5/#7), the
import keeps writing `amount` (= `amountCents / 100`) next to `amountCents`, so the live
app doesn't break. Nothing new should read them. Sprint 5 #8 migrates existing documents to
cents and then removes both fields (and the security rules for them).

`users/{uid}.monthlyIncome` is still dollars; it isn't part of this change.

## Entities

### User (profile and settings) — `users/{uid}`
| Field | Type | Who writes it | Notes |
|---|---|---|---|
| uid | string | | Firebase Auth user ID (the document ID, not a field) |
| displayName | string | signup, Settings | 1–60 characters. Saved only here, not in the Firebase Auth profile |
| createdAt | timestamp | signup | |
| consentAcceptedAt | timestamp | consent screen | Set once the user accepts the data consent screen |
| monthlyIncome | number | null | Settings | Dollars per month, 0 to 10,000,000, or null/missing = not set. Used as the budget for a month where no money came in (for example when only a credit card statement is imported). A month with real money in always uses that instead |

The email isn't saved here: it lives in Firebase Auth, which is where it's read from (SCRUM-146).
Older documents may still have an `email` field; nothing reads it.

Rules: the user can read only their own document, and can change only `displayName`,
`consentAcceptedAt` and `monthlyIncome`, with the limits above (SCRUM-88).

### Transaction — `transactions/{transactionId}`
| Field | Type | Notes |
|---|---|---|
| id | string | The document ID (below), not a field |
| userId | string | **Required.** The owner's Firebase Auth uid. Set on create from the verified auth token, never from client input. Never changes |
| date | timestamp | Transaction date |
| merchant | string | Raw merchant name from statement |
| amountCents | integer | **Money in cents.** Negative = spending, positive = income (see "Money is stored in whole cents") |
| amount | number | **Deprecated.** Dollars, `amountCents / 100`. Still written until the web app reads cents; removed by the migration (#8) |
| category | string | A category id, e.g. "dining", "groceries" (list in categories.md) |
| categoryConfidence | number | null | 0–1; null until something categorizes it (categories.md) |
| categorySource | string | "none", "rule", "ai", "cache" or "user" (categories.md) |
| isUserCorrected | boolean | **Removed (SCRUM-146).** Same as `categorySource == "user"`, which is the source of truth. Older documents may still have it; nothing reads it |
| accountId | string | null | The account it belongs to (`accounts/{accountId}`), or null. Only the id: type, institution and last 4 digits are read from the account |
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

### Statement (import record) — `users/{uid}/imports/{importId}`
Written only by the server (the `onStatementUploaded` Cloud Function and the delete/restore
functions); the app can only read it.

| Field | Type | Notes |
|---|---|---|
| id | string | Hash of the upload (the document ID, not a field); the owner is the `{uid}` in the path |
| storagePath | string | `users/{uid}/imports/{timestamp}_{fileName}`. The file name and type (csv/pdf) are read from this |
| accountId | string | null | The account the statement is for (`accounts/{accountId}`): the one picked, or the one detected. Only the id |
| detectedAccountType | string | null | "checking" or "credit": what the file looked like (null = couldn't tell) |
| sizeBytes | number | File size |
| status | string | "uploaded", "processing", "completed", "failed" |
| skippedRowCount | number | Rows that couldn't be read |
| duplicateRowCount | number | Rows an earlier statement already saved, so they were skipped |
| rowErrors | array | `[{ row, reason }]` for skipped rows, at most 100 |
| error | string | null | Message for the user when status is "failed" |
| uploadedAt | timestamp | |
| deletedAt / deletedBy / purgeAfter | | Soft delete, same as transactions |

**Not stored (Sprint 5, data-duplication-audit.md):** the record doesn't copy what is stored elsewhere.
- How many transactions it saved: count the transactions with `userId == uid` and
  `sourceStatementId == importId` (the Import page uses a `count()` query).
- File name and type: from `storagePath`.
- Account type: from the account (`accountId`).
Older records may still have `transactionCount`, `fileName`, `fileType` and `accountType` until
`functions/scripts/removeImportCopies.js` is run; nothing reads them.

### Account — `accounts/{accountId}` (Sprint 5 #17)

One document per bank account or credit card a user has. Today the app has two accounts built
into the code (`acct-checking`, `acct-credit` in the web app's `mockData.js` and the backend's
`accountType.js`); this is where real, per-user accounts will live instead.

| Field | Type | Notes |
|---|---|---|
| id | string | Firestore auto-generated ID (the document ID, not a field). Never contains the account number |
| userId | string | **Required.** The owner's Firebase Auth uid. Set on create from the verified auth token, never from client input. Never changes |
| type | string | `"checking"` or `"credit"` |
| institution | string | The bank or card company as the user knows it, e.g. "Chase", 1–60 characters |
| last4 | string | null | Last 4 digits of the account number, exactly 4 digits, shown as •••• 4521. **Never the full number** |
| nickname | string | null | The user's own name for it, e.g. "Joint checking", up to 40 characters |
| createdAt | timestamp | Set on create |
| updatedAt | timestamp | Set on every change |
| deletedAt | timestamp | null | **Required.** Created as null; set when the user deletes it (soft delete) |
| deletedBy | string | null | uid of whoever deleted it, or null |
| purgeAfter | timestamp | null | When the daily job may move it to the archive, or null |

**Only the account document describes the account.** Transactions and statements store just its
`accountId`; the type, institution, last 4 digits and nickname are read from here, never copied
(data-duplication-audit.md, D4). The import's `detectedAccountType` is what the file *looked*
like, not a copy of `type`.

**Queries the app runs.**
| Screen | Query |
|---|---|
| Account list (Settings, Import picker, filters) | `userId == uid`, `deletedAt == null` |

**Rules this needs** (same pattern as transactions)
- Read: `resource.data.userId == request.auth.uid`; a list query must include
  `where("userId", "==", <own uid>)`.
- Create: `request.resource.data.userId == request.auth.uid`, only the fields above, `type` is
  `"checking"` or `"credit"`, `last4` is null or exactly 4 digits, `deletedAt == null`.
- Update: the owner may change `institution`, `last4`, `nickname` and `updatedAt`. `userId`
  and `type` never change (a different kind of account is a new account).
- Delete: never from the client. Deleting is a soft delete (set or clear only `deletedAt`,
  `deletedBy`, `purgeAfter`, the same as budgets); the daily job archives it after 30 days
  (`archive_accounts`, admin-only like the other archive collections).
- A soft-deleted account's transactions and statements stay as they are and keep the
  `accountId`; the app shows them as "Deleted account".

**Moving from the built-in accounts.** Existing transactions and statements use the ids
`acct-checking` and `acct-credit`. When per-user accounts ship, a one-time script creates each
user's two accounts as `accounts/{uid}_checking` and `accounts/{uid}_credit` (the only
non-random ids, so the script can be run again safely) and rewrites `accountId` on that user's
transactions and statements to match.

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
| categoryId | string | **Removed (SCRUM-146).** The document ID is the category, e.g. "dining" (one budget per category). Older documents may still have it and, if so, it matches the ID |
| limitCents | integer | **Monthly limit in cents**, 0 or more (`25000` = $250) |
| limit | number | **Deprecated.** Monthly limit in dollars. Removed by the migration (#8) |
| updatedAt | timestamp | Set on every save |

- A budget is a **monthly** limit that applies to every month: the app compares it with
  whichever month is being shown. There is no per-month copy and no stored "spent" total;
  spending is always added up from the transactions, so it can't get out of sync.
- `income`, `transfers` and `uncategorized` can't have a budget (categories.md).
- The document ID is the only place the category is saved. Likewise, a monthly budget record
  `users/{uid}/budgetMonths/{YYYY-MM}` has no `month` field: its ID is the month (SCRUM-146).
- Rules: only the owner can read, create, change or delete their budgets, with exactly
  these fields (SCRUM-81, SCRUM-88).

## Relationships
- One **User** → many **Accounts**, **Transactions**, **Statements**, **Budgets**
- One **Account** → many **Transactions** and **Statements** (by `accountId`)
- One **Statement** → many **Transactions** (the ones extracted from it)
- One **Category** → many **Transactions** and **Budgets**

## Notes
- Access to transactions and accounts is controlled by their `userId` field; everything else is
  under `users/{uid}`, and the rules check `request.auth.uid == uid`.
- Categories are a fixed list (categories.md). Users can add their own `custom-...` categories;
  those are kept in the browser for now and would move to `users/{uid}/categories/{id}`.