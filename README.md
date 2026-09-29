# Budgetly — Contracts

The interfaces every Budgetly app and service builds against: data schemas, API definitions, and the AI interface.

If two repos need to agree on something (the shape of a transaction, what a function returns, what the AI model receives), it's defined here once.

## Budgetly Repositories

| Repo | Contents |
|---|---|
| [budgetly_web_customer](https://github.com/zsheppard78/budgetly_web_customer) | Customer-facing React web app |
| [budgetly_web_dash](https://github.com/zsheppard78/budgetly_web_dash) | Dashboard web app |
| [budgetly_backend_nodejs](https://github.com/zsheppard78/budgetly_backend_nodejs) | Firebase Cloud Functions (Node.js), Firestore rules |
| **budgetly_contracts** (this repo) | Interfaces: data schemas, API definitions, AI interface |
| [budgetly_assets](https://github.com/zsheppard78/budgetly_assets) | Docs, images, reports |

## Structure

```
schemas/
  data-model.md           Firestore data structure: users, transactions, budgets, uploads
ai/
  ai-integration-plan.md  AI categorization input/output, confidence, fallback behavior
api/                      Cloud Function interfaces: inputs, outputs, error codes (planned)
```

## Rules

1. **Contract first.** Any change to a data shape, function signature, or AI input/output is made here before the code changes.
2. **Both sides review.** Contract PRs need approval from someone working on each affected side (frontend and backend).
3. **Note breaking changes.** If a change would break existing code, say so in the PR title and list which repos need updates.

## Workflow

All changes go through a branch and pull request with at least one reviewer approval. No direct pushes to `main`.

## Team

- Zach Sheppard
- Tyler Petersen

CSCI 475 / CIS 675: Software Engineering, UW-Parkside, Fall 2026