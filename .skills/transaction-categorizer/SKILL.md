---
name: transaction-categorizer
description: Categorize YNAB transactions using payee-based rules with progressive automation. Start in suggest-and-confirm mode, graduate to auto-apply as confidence builds.
---

# Transaction Categorizer

Categorize uncategorized YNAB transactions using learned payee patterns. Operates in three modes with increasing automation.

## Modes

### Mode 1 — Suggest & Confirm (default)

Use this mode when starting out or when the user asks to "categorize transactions" or "review uncategorized."

**Workflow:**

1. Fetch transactions needing attention: use `getTransactions` with `type=unapproved` for the active budget (this matches the YNAB dashboard "new" count). Filter to those with `category_id=null` and no `transfer_account_id` — these are the ones that actually need categorization. Do NOT use `type=uncategorized` — it returns a superset including old approved transfers that don't need action.
2. Load rules from `references/payee-rules.md`
3. For each transaction needing a category:
   - Match payee name against known rules (exact match first, then prefix/pattern)
   - Present the transaction with your suggested category and confidence level
   - Format: `$AMOUNT | PAYEE | → SUGGESTED_CATEGORY (reason)`
4. Wait for user confirmation or correction
5. Apply confirmed categories via `updateTransaction`
6. **If the user corrects a suggestion**, add or update the rule in `references/payee-rules.md`

**Batching:** Present 5-10 transactions at a time. Group by suggested category when possible to speed up review.

### Mode 2 — Auto + Flag

Use when the user says "auto-categorize" or "categorize automatically."

1. Auto-apply categories for payees with established rules (exact match or high-confidence pattern)
2. Flag ambiguous transactions for manual review
3. Present a summary: "Auto-categorized X transactions, Y flagged for review"
4. Show flagged transactions for user decision

### Mode 3 — Full Auto

Use when the user says "categorize everything" or explicitly requests no review.

1. Apply all rules automatically
2. For truly unknown payees, skip (leave uncategorized) rather than guess
3. Present a completion summary with counts by category

## Rule Matching Priority

1. **Exact payee match** — highest confidence (e.g., "Whole Foods Market" → Groceries)
2. **Payee prefix/pattern** — high confidence (e.g., payee starts with "AMZN" → Shopping)
3. **Amount-based hint** — medium confidence (e.g., recurring $15.99 → Subscriptions)
4. **Account-based default** — low confidence, only suggest (e.g., dining credit card → Dining Out)

## Learning New Rules

When a user confirms or corrects a categorization:
1. Check if a rule already exists for that payee in `references/payee-rules.md`
2. If yes, update it (corrections override previous rules)
3. If no, add a new rule entry
4. Use the most specific match type possible (prefer exact over pattern)

## Important Notes

- Always fetch the budget's category list first (`getCategories`) so suggestions use actual category names
- The YNAB API uses `category_id` for updates — resolve category names to IDs before calling `updateTransaction`
- Amounts in YNAB are in milliunits (divide by 1000 for display)
- Present outflows as positive numbers for readability (YNAB stores them as negative milliunits)
- When showing transactions, include the date and account name for context
- Use `plan_id=last-used` for all API calls (avoids needing to look up the budget ID)
- **API filter gotcha:** `type=unapproved` matches the YNAB dashboard's "new transactions" count. `type=uncategorized` returns a much larger set including old approved transfers that are intentionally category-free. Always prefer `unapproved` and filter client-side for `category_id=null`.
- Transfers between accounts (`transfer_account_id` is set) don't need categories — exclude them from categorization batches
