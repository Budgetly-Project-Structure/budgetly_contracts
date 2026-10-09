# Categories (SCRUM-68)

The fixed list of transaction categories, and the category fields every transaction carries.
The web app (`src/data/mockData.js` → `CATEGORIES`) and the backend (`functions/lib/categories.js`)
both copy this list. Change it here first, then in both repos.

## Category list

| id | Label | Kind | AI can assign? | Can have a budget? |
|---|---|---|---|---|
| `dining` | Dining Out | spending | yes | yes |
| `groceries` | Groceries | spending | yes | yes |
| `transportation` | Transportation | spending | yes | yes |
| `subscriptions` | Subscriptions | spending | yes | yes |
| `shopping` | Shopping | spending | yes | yes |
| `utilities` | Utilities | spending | yes | yes |
| `entertainment` | Entertainment | spending | yes | yes |
| `health` | Health | spending | yes | yes |
| `income` | Income | income | yes | no |
| `transfers` | Transfer | transfer | yes | no |
| `uncategorized` | Uncategorized | none | no (it's the fallback) | no |

- **ids never change.** They're stored on every transaction and budget. A label can be renamed freely.
- **Spending** categories count toward money out and can have a monthly budget.
- **Income** counts toward money in.
- **Transfer** is money moved between the user's own accounts (for example paying off a credit card).
  It counts as neither money in nor money out.
- **Uncategorized** means "not decided yet". Transactions in it are flagged for review.
- Icons and colors are UI details and live in the web app only.

### Custom categories

Users can add their own categories (web app: Budget page → "+ New category").

- Their id is `custom-` + the name in lowercase with dashes, e.g. "Pet Care" → `custom-pet-care`,
  so they can never clash with a built-in id.
- They count as **spending** and can have a budget.
- The AI never assigns them (it only sees the fixed list above). Users assign them by hand.
- Saved per user at `users/{uid}/categories/{id}` once SCRUM-80 lands. Until then they're kept in the browser.

## Category fields on a transaction

Stored on `users/{uid}/transactions/{id}` (see `data-model.md`).

| Field | Type | Meaning |
|---|---|---|
| `category` | string | A category id from the list above, or a custom id |
| `categoryConfidence` | number \| null | 0–1. How sure the source is. `null` when nobody has categorized it yet |
| `categorySource` | string | Who set the category (below) |
| `isUserCorrected` | boolean | `true` once the user has picked the category themselves (derived: `categorySource == "user"`) |

### `categorySource` values

| Value | Set by | Confidence |
|---|---|---|
| `none` | Import, nothing has categorized it yet | `null` |
| `rule` | Import, a fixed rule matched: a card payment → `transfers`, or a known merchant from the free keyword list (e.g. STARBUCKS → `dining`) | `1` for card payments, `0.9` for merchant keywords |
| `ai` | The categorization Cloud Function (SCRUM-71) | what the model returned, 0–1 |
| `cache` | The merchant cache reused an earlier answer (SCRUM-75) | the cached confidence |
| `user` | The user picked it on the Transactions page | `1` |

Precedence: **user > rule > ai/cache > none.** Nothing automatic ever overwrites a `user` category,
and the AI step skips transactions that already have a `rule` category.

### Review flag

A transaction needs review when it is **not** user-corrected and either its category is
`uncategorized` or `categoryConfidence < 0.7`. The web app shows these with a "Review" chip.
