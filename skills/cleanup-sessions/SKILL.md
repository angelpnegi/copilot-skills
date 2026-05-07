---
name: cleanup-sessions
description: "Finds and deletes empty or low-activity Copilot CLI sessions. Load when: the user wants to clean up sessions, remove empty sessions, or prune their session list."
---

# Cleanup Sessions

## KNOW

- Sessions are stored in `~/.copilot/session-store.db` (SQLite).
- The `sessions` table holds session metadata; the `turns` table holds conversation turns linked by `session_id`.
- A session with 0 turns is one that was opened but never used.
- The user may want to delete sessions below a certain turn threshold (default: sessions with fewer than 1 turn).
- The current active session must NEVER be deleted.
- Use Python's `sqlite3` module for database operations (the `sqlite3` CLI may not be available).
- The SQL `session_store` database (via the SQL tool) is read-only — use it only for querying. Use Python via bash for writes.

## DO

### Step 1 — Ask for threshold

Ask the user: "Delete sessions with fewer than how many turns?" Offer choices: 1 (empty only), 2, 3, 5.

Default to 1 if the user doesn't specify.

### Step 2 — Query sessions below threshold

Use the SQL tool with `database: "session_store"` to find sessions:

```sql
SELECT s.id, COALESCE(s.summary, '(unnamed)') as name, s.created_at,
       (SELECT COUNT(*) FROM turns t WHERE t.session_id = s.id) as turn_count
FROM sessions s
WHERE s.id != '<CURRENT_SESSION_ID>'
  AND (SELECT COUNT(*) FROM turns t WHERE t.session_id = s.id) < <THRESHOLD>
ORDER BY s.created_at DESC
```

Replace `<CURRENT_SESSION_ID>` with the active session ID from session context. Replace `<THRESHOLD>` with the user's chosen value.

### Step 3 — Present findings and confirm

Show the user a table of sessions that will be deleted:
- ID (first 8 chars for readability)
- Name
- Turns
- Created date

Show the total count. Ask the user to confirm: "Delete all X sessions?" with choices Yes / No / Let me pick which ones.

If the user wants to pick, let them specify which to keep.

### Step 4 — Backup and delete

Use Python via bash to:
1. Create a backup of the session store
2. Delete confirmed sessions by explicit IDs
3. Report what was deleted

```python
import sqlite3, shutil

db_path = '/home/<user>/.copilot/session-store.db'
backup_path = db_path + '.cleanup-bak'

# Backup
shutil.copy2(db_path, backup_path)

# Delete
conn = sqlite3.connect(db_path)
ids = ['<id1>', '<id2>', ...]
placeholders = ','.join(['?' for _ in ids])
cur = conn.execute(f'DELETE FROM sessions WHERE id IN ({placeholders})', ids)
conn.commit()
print(f'Deleted {cur.rowcount} sessions')
conn.close()
```

Replace `<user>` with the actual username from the environment. Use the actual session IDs from Step 2.

### Step 5 — Report results

After deletion, show a summary:
- Total sessions deleted (with their names and turn counts from Step 2)
- Total sessions remaining (query session_store again)

### Step 6 — Remove backup

After confirming deletion was successful (deleted count matches expected), remove the backup file:

```python
import os
os.remove('/home/<user>/.copilot/session-store.db.cleanup-bak')
print('Backup removed')
```

If deletion count does NOT match expected, keep the backup and warn the user.

## CHECK

- [ ] The current active session was excluded from the delete list.
- [ ] The user was shown the full list of sessions to be deleted before any action.
- [ ] The user explicitly confirmed before deletion was performed.
- [ ] Only sessions below the specified turn threshold were targeted.
- [ ] A backup was created before deletion.
- [ ] The backup was removed after successful deletion.
- [ ] A post-deletion summary showing deleted sessions (with turn counts) and remaining session count was provided.
- [ ] Python `sqlite3` module was used (not the sqlite3 CLI binary).
- [ ] Deletion used explicit IDs — never a broad WHERE clause without IDs.
