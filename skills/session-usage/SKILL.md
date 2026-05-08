---
name: session-usage
description: "Shows estimated token usage across Copilot CLI sessions for a given time period. Load when: the user wants to see token usage, session stats, how much they've used Copilot, or usage metrics."
---

# Session Usage

## KNOW

- Session data is in `~/.copilot/session-store.db` (SQLite), accessible read-only via the SQL tool with `database: "session_store"`.
- The `turns` table has: session_id, turn_index, user_message, assistant_response, timestamp.
- The `sessions` table has: id, summary, created_at, updated_at.
- Token counts are not stored directly. Estimate using character lengths: ~1 token per 4 characters (English text average).
- Input tokens = user_message characters / 4. Output tokens = assistant_response characters / 4.
- Some assistant_response values may be NULL (e.g., tool-only responses). Treat NULL as 0.

## DO

### Step 1 — Ask for time period

Ask the user: "Show usage for what time period?" Offer choices: Last 24 hours, Last 7 days, Last 30 days, All time.

### Step 2 — Query usage data

Use the SQL tool with `database: "session_store"`:

```sql
SELECT
    s.id,
    COALESCE(s.summary, '(unnamed)') as session_name,
    COUNT(t.turn_index) as turns,
    SUM(COALESCE(length(t.user_message), 0)) as total_input_chars,
    SUM(COALESCE(length(t.assistant_response), 0)) as total_output_chars,
    MIN(t.timestamp) as first_activity,
    MAX(t.timestamp) as last_activity
FROM sessions s
JOIN turns t ON t.session_id = s.id
WHERE t.timestamp >= datetime('now', '<TIME_OFFSET>')
GROUP BY s.id
ORDER BY total_output_chars DESC
```

Replace `<TIME_OFFSET>` with:
- Last 24 hours: `-1 day`
- Last 7 days: `-7 days`
- Last 30 days: `-30 days`
- All time: remove the WHERE clause

### Step 3 — Present results

Show a table with:
- Session name
- Turns
- Est. input tokens (input_chars / 4, rounded)
- Est. output tokens (output_chars / 4, rounded)
- Est. total tokens
- Last active time

Then show totals at the bottom:
- Total sessions active in period
- Total turns across all sessions
- Total estimated input tokens
- Total estimated output tokens
- Grand total estimated tokens

Format large numbers with commas for readability (e.g., 12,450 tokens).

### Step 4 — Offer breakdown (optional)

Ask the user if they want a per-day breakdown or details on a specific session. If yes, query accordingly and present.

## CHECK

- [ ] The user was asked for a time period before querying.
- [ ] Token estimates clearly state they are estimates (~4 chars per token).
- [ ] Results are sorted by usage (heaviest sessions first).
- [ ] Totals are shown at the bottom.
- [ ] NULL assistant_response values were handled (treated as 0, not skipped).
- [ ] Session names are shown (not just IDs).
