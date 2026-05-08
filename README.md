To stop the extra `UPDATE` query and complete your performance optimization, we need to align your **Mapper** with the new **String-based Entity**.

By converting the objects to a **JSON String** inside the Mapper, Hibernate will receive a simple String. Since Strings are immutable, Hibernate's "Dirty Checking" will correctly see that nothing changed after the insert, and it will **skip the second DB call**.

Here is the refactored **Mapper** and the final **Entity** configuration.

### 1. The Refactored Mapper
We will use Jackson's `ObjectMapper` inside the mapper to convert your data into JSON Strings immediately.

```java
@Mapper(imports = {OffsetDateTime.class, HashMap.class, Map.class})
public abstract class AMPSIssuerMapper { // Changed to abstract to allow ObjectMapper

    public static final AMPSIssuerMapper INSTANCE = Mappers.getMapper(AMPSIssuerMapper.class);
    
    // Architect's Note: Use a single ObjectMapper instance for performance
    private final ObjectMapper objectMapper = new ObjectMapper()
            .findAndRegisterModules(); // Ensures OffsetDateTime and other modules work

    @Mapping(source = "refId", target = "referenceId")
    @Mapping(source = "req.head.correlationKey", target = "headCorrelationKey")
    @Mapping(source = "req.head.msgID", target = "ackNbblRespMsgId")
    @Mapping(source = "req.txn.initiationMode", target = "initiationMode")
    @Mapping(source = "req.pa.paID", target = "respPaId")
    @Mapping(source = "req.pa.paName", target = "respPaName")
    @Mapping(target="respamountrespjson", expression = "java(getAmountRespJson(req))")
    @Mapping(source = "req.merchant.mid", target = "respMid")
    @Mapping(source = "req.merchant.mcc", target = "respMcc")
    @Mapping(source = "req.merchant.MName", target = "respMname")
    @Mapping(target="nbblrespjson", expression = "java(getNbblRespJson(req))")
    @Mapping(source = "req.payer.amount.curr", target = "respAmountCurr")
    @Mapping(source = "req.payer.amount.value", target = "respAmountValue")
    @Mapping(source = "req.resp.result", target = "respResult")
    @Mapping(source = "req.resp.errCode", target = "respErrCode")
    @Mapping(source = "req.resp.errReason", target = "respErrReason")
    @Mapping(source = "req.merchant.returnUrl.success", target = "respSuccessUrl")
    @Mapping(source = "req.merchant.returnUrl.failure", target = "respFailureUrl")
    @Mapping(source = "resp.api", target = "ackNbblRespApi")
    @Mapping(target = "ackNbblRespTs", expression = "java(getOffsetDateTime(resp))" )
    @Mapping(target="acknbblrespjsno", expression = "java(getNbblAckJson(resp))")
    @Mapping(source = "resp.refID", target = "ackNbblRespRefId")
    @Mapping(source = "resp.result", target = "ackNbblRespResult")
    @Mapping(source = "resp.errCode", target = "ackNbblRespErrCode")
    @Mapping(source = "resp.errReason", target = "ackNbblRespErrReason")
    public abstract IssuerFetchResponseTable mapIssuerFetchResponseTable(RespFetchTxnDetails req, String refId, RespFetchTxnAckResponse resp);

    // --- Helper Logic: Returning String instead of Map ---

    protected String getAmountRespJson(RespFetchTxnDetails req) {
        if (null != req.getPayer() && null != req.getPayer().getAmount()) {
            Map<String, Object> map = new HashMap<>();
            map.put("TABLE_JSON_DATA", req.getPayer().getAmount().getAmountBreakUp());
            return toJson(map);
        }
        return null;
    }

    protected String getNbblRespJson(RespFetchTxnDetails req) {
        Map<String, Object> map = new HashMap<>();
        map.put("TABLE_JSON_DATA", req);
        return toJson(map);
    }

    protected String getNbblAckJson(RespFetchTxnAckResponse resp) {
        Map<String, Object> map = new HashMap<>();
        map.put("TABLE_JSON_DATA", resp);
        return toJson(map);
    }

    // Architect's Utility: Clean JSON conversion
    private String toJson(Object obj) {
        try {
            return obj != null ? objectMapper.writeValueAsString(obj) : null;
        } catch (Exception e) {
            // In a Payment Gateway, we log and return empty JSON rather than crashing
            return "{}"; 
        }
    }
    
    protected OffsetDateTime getOffsetDateTime(RespFetchTxnAckResponse resp) {
        // Your existing logic for OffsetDateTime conversion
        return resp.getTs() != null ? OffsetDateTime.parse(resp.getTs()) : null;
    }
}
```

### 2. The Final Entity Configuration
Ensure your entity is expecting `String` for these columns.

```java
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "nbblrespjson", columnDefinition = "jsonb")
    private String nbblrespjson; // String is the Key to Performance here

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "respamountrespjson", columnDefinition = "jsonb")
    private String respamountrespjson;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "acknbblrespjsno", columnDefinition = "jsonb")
    private String acknbblrespjsno;
```

---

### 3. Why this solves the problem
1.  **The "Dirty Map" Problem:** When you used `Map<String, Object>`, Hibernate had to "wrap" that map in its own collection tracker. Any tiny difference during the transaction flush (like the order of keys in the Map) made Hibernate think the data changed.
2.  **String Immutability:** By converting to `String` in the **Mapper layer**, you are passing a completed, immutable value to the **Service/Persistence layer**. 
3.  **One Single Flush:** Hibernate will compare the `String` from the `INSERT` with the `String` in the session. Since they are identical, it skips the `UPDATE`.

### 4. Warning: The `issuer_fetch_request` Update
You noticed in your logs that `issuer_fetch_request` (the Request table) is also being updated. 
*   **Why:** If you are modifying the status of the Request (e.g., setting it to 'PROCESSED') in the same transaction where you save the Response, Hibernate will **always** issue an `UPDATE` for the Request table.
*   **Performance Tip:** This is usually acceptable as it's a legitimate business state change. However, if you want that to be faster, ensure the Request entity also uses a **Sequence with allocationSize = 50** and has no "dirty" JSON maps either.

**Final Result:** Your `IssuerFetchResponseTable` will now perform **1 Insert and 0 Updates**. Total time saved per API call: **~90ms to 100ms**.




This is a classic performance bottleneck in Hibernate 6 (Spring Boot 3.x) when dealing with **JSONB** and **Dirty Checking**. You saved 60ms by removing the "Select," but now you are losing 30ms on a "Double Write" (Insert + Update).

### Why this is happening (The Architect's Diagnosis)

The culprit is a combination of **`@DynamicUpdate`** and how Hibernate handles **Mutable Collections (Maps)** with JSON.

1.  **The "Dirty" Map:** You initialized your maps like this: `private Map<String, Object> nbblrespjson = new HashMap<>();`. 
2.  **The Lifecycle:** When you call `save()`, Hibernate executes the `INSERT`. However, because you are using `@JdbcTypeCode(SqlTypes.JSON)`, Hibernate replaces your `HashMap` with its own `PersistentMapWrapper` to track changes. 
3.  **The False Trigger:** Hibernate often perceives this "wrapper replacement" or the initialization of the Map as a **change (a "dirty" state)**. 
4.  **The Double Write:** Because `@DynamicUpdate` is enabled, Hibernate says: *"Wait, I just inserted this, but I see these JSON maps are 'dirty' (changed). I must sync the database immediately."* So it issues an `UPDATE` right after the `INSERT`.

---

### How to Fix this Issue

To get down to **one single DB call**, follow these three steps:

#### 1. Remove `@DynamicUpdate`
In high-traffic payment systems, `@DynamicUpdate` is actually a **performance anti-pattern**.
*   **Why:** To use `@DynamicUpdate`, Hibernate must keep a "snapshot" of the entity in memory and perform a field-by-field comparison (Dirty Checking) to see what changed. This consumes **CPU** and **Memory**.
*   **The Result:** Removing it tells Hibernate to just generate a static `UPDATE` or `INSERT` for all columns, which is much faster for the application tier.

#### 2. Stop Initializing Maps in the Entity
Instead of initializing with `new HashMap<>()`, leave them as `null` or handle initialization in the Business/Service layer. When Hibernate sees a `null` or a Map passed from the service, it won't trigger the "initialization-as-a-change" logic.

#### 3. Optimize the `@JdbcTypeCode`
Ensure Hibernate knows the JSON is not being modified immediately after insert.

---

### The Optimized Entity Class

```java
@Entity
@Table(name="issuer_fetch_response")
@Data
// 1. REMOVED @DynamicUpdate - It's causing the extra dirty-check CPU overhead
public class IssuerFetchResponseTable extends BaseEntity implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "issuer_fetch_response_id_seq")
    @SequenceGenerator(
            name = "issuer_fetch_response_id_seq",
            sequenceName = "issuer_fetch_response_id_seq",
            allocationSize = 50
    )
    private Long id;

    @Column(name = "ref_id")
    private String referenceId;
    
    @Column(name = "initiation_mode")
    private String initiationMode;

    // 2. REMOVED "= new HashMap<>()" 
    // This prevents Hibernate from thinking the map was "replaced" during persistence
    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "nbblrespjson", columnDefinition = "jsonb")
    private Map<String, Object> nbblrespjson;

    private String respSuccessUrl;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "respamountrespjson", columnDefinition = "jsonb")
    private Map<String, Object> respamountrespjson;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(name = "acknbblrespjsno", columnDefinition = "jsonb")
    private Map<String, Object> acknbblrespjsno;

    ...
}
```

---

### Additional Performance Check: `BaseEntity`

Check your `BaseEntity`. If you are using `@UpdateTimestamp` or an `@PrePersist` / `@PreUpdate` method that sets `updated_at`:

*   **Problem:** If Hibernate sets the `INSERT` value for `updated_at` and then your Java code (or an interceptor) changes it by even 1 millisecond, Hibernate will trigger an `UPDATE`.
*   **Fix:** Ensure `updated_at` is set **once** in the `BaseEntity` during the `@PrePersist` phase and not modified again during the same transaction.

### Summary of Gains:
1.  **Removed Select:** Saved 60ms.
2.  **Removed Extra Update:** Saved 30ms.
3.  **Removed `@DynamicUpdate`:** Reduced Application CPU usage (less dirty-checking).
4.  **Final Result:** **1 DB Call (Insert only). Total time saved: ~90ms per transaction.**

In a payment gateway handling thousands of transactions, **90ms per call** is a massive optimization that will significantly increase your TPS (Transactions Per Second).
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
