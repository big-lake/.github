---
mode: agent
description: 'Delete all notebooks from the local SQLite DB (C:\opt\biglake\api\app.db) so you can test the first-visit welcome landing. Asks for confirmation first.'
---

# Delete local notebooks

Wipes all rows from the `notebooks` table in the local dev SQLite database so the
welcome landing (`§5.1a` in `BFF_API_REQUIREMENTS.md`) is visible on next page load.

**Before running**, confirm with the user that they want to delete all local notebooks —
they cannot be recovered.

Once confirmed, run:

```python
import sqlite3
con = sqlite3.connect(r'C:\opt\biglake\api\app.db')
con.execute('DELETE FROM notebooks')
con.commit()
print('Deleted:', con.execute('SELECT changes()').fetchone()[0], 'rows')
con.close()
```

Use the api venv Python at `C:\Users\jerem\Github\biglake\api\.venv\Scripts\python.exe`.

After running, tell the user to hard-refresh `http://localhost:5173/?noauto=1`.
