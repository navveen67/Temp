Since you are using **Approach 1** with a `Long id` and a `SequenceGenerator` (with `allocationSize = 50`), it is critical to sync the sequence correctly to avoid "Duplicate Key" exceptions.

Here is the SQL script to synchronize your sequence with the existing data.

### 1. The Sync Script
This script finds the maximum ID currently in your table and sets the sequence to that value.

```sql
-- Sync the sequence with the maximum ID in the table
SELECT setval('issuer_fetch_response_id_seq', COALESCE((SELECT MAX(id) FROM issuer_fetch_response), 1), true);
```

**Explanation:**
*   `MAX(id)`: Finds the highest existing ID.
*   `COALESCE(..., 1)`: If the table is brand new (empty), it starts the sequence at 1.
*   `true`: This tells the database that the value being set has already been "used," so the **next** call to `nextval()` will return `MAX(id) + 50` (since your increment is 50).

---

### 2. Critical Step: If you just added the `id` column
If you just added the `id` column to a table that already has millions of rows, the `id` values for old rows will be `NULL`. You **must** populate them before running the sequence sync.

**Run this ONLY if your old rows have NULL IDs:**
```sql
-- 1. Create a temporary counter to update old rows
UPDATE issuer_fetch_response 
SET id = nextval('issuer_fetch_response_id_seq') 
WHERE id IS NULL;

-- 2. Then sync the sequence to the new MAX
SELECT setval('issuer_fetch_response_id_seq', (SELECT MAX(id) FROM issuer_fetch_response), true);
```

---

### 3. Verification Query
After running the sync, run this to make sure everything is correct:

```sql
-- Check the current max ID vs the next value the sequence will give
SELECT 
    (SELECT MAX(id) FROM issuer_fetch_response) AS current_max_id,
    nextval('issuer_fetch_response_id_seq') AS next_sequence_value;
```

### ⚠️ A Final Warning for High Traffic:
Because you are using **Hibernate Allocation Size = 50**, ensure your DB sequence was created with `INCREMENT BY 50`. 

If the DB sequence is `INCREMENT BY 1` but Java says `allocationSize = 50`, Hibernate will calculate IDs in memory that **will** eventually clash with the database.

**To be 100% safe, run this to check the increment:**
```sql
SELECT increment_by 
FROM information_schema.sequences 
WHERE sequence_name = 'issuer_fetch_response_id_seq';
```
*If it returns 1, you must run:* `ALTER SEQUENCE issuer_fetch_response_id_seq INCREMENT BY 50;`
