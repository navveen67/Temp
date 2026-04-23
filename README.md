To modify an existing sequence to work with your high-performance allocationSize
= 50 configuration, use the following script.

1. The Alter Script (Standard SQL / PostgreSQL / Oracle)

This script changes the increment logic without deleting your existing sequence
data.

-- 1. Change the increment to 50 to match Java's allocationSize
-- 2. Set CACHE to 1 to ensure the DB and Hibernate stay perfectly in sync
ALTER SEQUENCE sdk_key_ID_SEQ 
INCREMENT BY 50 
CACHE 1;

2. Senior Architect's "Alignment" Step (Very Important)

When you change an INCREMENT BY from 1 to 50, there is a risk that the "Current
Value" in the database is not a "round number." Hibernate prefers the sequence
to be aligned to the allocationSize.

I recommend you "Restart" the sequence at the next logical 50-increment to avoid
any ID collisions with existing data.

Example: If your current max ID in the table is 124, restart the sequence at 150
or 200.

-- Replace '200' with a value higher than your current MAX(id)
ALTER SEQUENCE sdk_key_ID_SEQ 
RESTART WITH 200 
INCREMENT BY 50 
CACHE 1;

3. How to Verify the Change

Run this query to confirm the database engine has updated the metadata
correctly:

For PostgreSQL:

SELECT sequence_name, increment_by, last_value 
FROM information_schema.sequences 
WHERE sequence_name = 'sdk_key_ID_SEQ';

For Oracle:

SELECT sequence_name, increment_by, cache_size 
FROM all_sequences 
WHERE sequence_name = 'SDK_KEY_ID_SEQ';

4. Why this script is "Highly Effective" for your team:

  - Zero Downtime: This script can be run while the application is live (though
    a restart of the App Server is recommended so Hibernate picks up the new
    allocationSize config).
  - Synchronicity: By setting CACHE 1, you prevent the "Double Caching" problem
    we discussed. The database acts as the source of truth, and Hibernate acts
    as the high-speed buffer.
  - Scalability: This change prepares your API to handle 50x more inserts with
    the same database load.

⚠️ One Final Warning for the Team:

If you have multiple instances of your application running (e.g., in a
Kubernetes cluster), once you apply this SQL script, all instances must be
updated to use allocationSize = 50 in the Java code.

If one instance still has allocationSize = 1, it will jump by 50 every time it
saves a single record, causing your IDs to exhaust very quickly!
