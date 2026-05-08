This is a critical part of a Payment Gateway. If the OTP logic is weak, it becomes a security vulnerability; if it's too strict, it kills the conversion rate.

Here is a comprehensive **OTP Testing Matrix** for your Confluence page, designed to ensure a robust delivery.

---

# Confluence Page: OTP Lifecycle Testing Scenarios (Generate & Verify)

## 1. Business Rules Summary
*   **Max Generation Limit:** 3 Attempts per Transaction/Reference ID.
*   **Max Validation Limit:** 3 Attempts per Transaction/Reference ID.
*   **Validity Window:** 15 Minutes (as per transaction TTL).
*   **Cool-down Period:** (Optional - usually 60s between resends).

## 2. OTP Testing Matrix

| ID | Category | Scenario | Steps | Expected Result |
|:---|:---|:---|:---|:---|
| **P1** | **Positive** | Happy Path (1+1) | Generate 1 OTP -> Enter correct OTP immediately. | Success. Transaction proceeds to Payment API. |
| **P2** | **Positive** | Resend Success | Generate 1st -> (Wait 60s) -> Generate 2nd -> Enter 2nd OTP. | Success. Previous OTP is invalidated; latest is accepted. |
| **P3** | **Positive** | Limit Boundary (3rd Gen) | Generate 1st -> Generate 2nd -> Generate 3rd -> Enter 3rd OTP. | Success. 3rd attempt is allowed and works. |
| **P4** | **Positive** | Limit Boundary (3rd Val) | Generate 1st -> Enter Wrong -> Enter Wrong -> Enter Correct (3rd try). | Success. User allowed to pay on the last valid attempt. |
| **N1** | **Negative** | Gen Limit Exhausted | Attempt to generate OTP for the **4th time**. | **Error:** "Maximum OTP generation limit reached." |
| **N2** | **Negative** | Val Limit Exhausted | Enter Wrong OTP 3 times. | **Error:** "Maximum validation attempts reached. Transaction Blocked." |
| **N3** | **Negative** | Invalid OTP | Enter an OTP that does not match the generated one. | **Error:** "Invalid OTP. 2 attempts remaining." |
| **N4** | **Negative** | Expired OTP | Generate OTP -> Wait > 15 Minutes -> Enter correct OTP. | **Error:** "OTP Expired" or "Transaction Timed Out." |
| **N5** | **Negative** | Replay Attack | Use a previously successful OTP for a new validation call. | **Error:** "OTP already used/Invalid." |
| **N6** | **Negative** | Cross-Ref ID Check | Use OTP from Ref_ID_A to validate Ref_ID_B. | **Error:** "Invalid OTP/Transaction mismatch." |
| **N7** | **Negative** | Empty/Null/Format | Submit empty OTP or alphanumeric characters (if numeric only). | **Error:** Validation error (400 Bad Request). |
| **E1** | **Edge Case** | Midnight Date Flip | Generate OTP at 23:55 (Date X) -> Validate at 00:05 (Date Y). | **Success.** (Based on our earlier discussion on Julian Date). |
| **E2** | **Edge Case** | Concurrent Verify | Rapidly click "Verify" button twice (Race condition). | Only 1 request succeeds; 2nd says "Processing" or "Invalid." |
| **E3** | **Edge Case** | Verify after Gen Limit | Gen 3 times (Max) -> Validate on 1st try. | **Success.** Even if Gen is maxed, Validation should still work. |

---

## 3. Developer Implementation Checklist (Internal)

To ensure the performance optimization we discussed, the developers must ensure:

1.  **Atomic Counters:** The "Number of attempts" should be stored in **Aerospike/Redis** using an atomic increment (`INCR`). Do not read from DB, increment in Java, and save back (this causes race conditions).
2.  **State Cleanup:** Once an OTP is successfully verified, the "Transaction Context" in the cache should be updated to status `OTP_VERIFIED` to prevent the user from jumping back to the Generate OTP API.
3.  **Idempotency:** The `verifyOTP` API must be idempotent for the same `referenceId` if the status is already `SUCCESS`.
4.  **Logging:** Ensure PII (the actual OTP value) is **NEVER** printed in logs. Only log the `referenceId` and the `attempt_number`.

---

## 4. Error Message Mapping (For UI/UX Team)

| Error Code | Scenario | Recommended User Message |
|:---|:---|:---|
| `OTP_GEN_LIMIT` | 4th Gen Attempt | You have reached the maximum limit for Resend OTP. Please start a new transaction. |
| `OTP_VAL_LIMIT` | 3rd Wrong Attempt | Multiple incorrect attempts. For security, this transaction is now blocked. |
| `OTP_EXPIRED` | > 15 mins | This OTP has expired. Please request a new one. |
| `OTP_INVALID` | Incorrect Input | The OTP entered is incorrect. You have [X] attempts remaining. |

---

### Pro-Tip for your Confluence Page:
Add a **Flow Diagram** (Mermaid or LucidChart) showing the transition from `GENERATE_OTP_API` -> `CACHE_STATE_UPDATE` -> `VALIDATE_OTP_API`. This will help the testing team understand that the state is managed in the cache, not just the database.
    // Use this lifecycle hook to ensure that after loading from DB, 
    // it's no longer considered "new"
    @PostLoad
    @PostPersist
    void markNotNew() {
        this.isNew = false;
    }
}
```

#### 2. Optimization for "Audit" Logic
Since this is an **Audit** table (`tpt_limit`), you are likely **only ever inserting** and never updating an existing audit row. 

If the database Primary Key is actually the `bigserial id`, the most efficient architectural move is to change the mapping to reflect the truth of the database:

**Recommended Corrected Mapping:**
```java
@Entity
@Table(name = "tpt_limit")
@Data
public class TptLimitAudit {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY) // Maps to bigserial
    private Long id;

    @Column(name = "ref_id")
    private String refId;

    @Column(name = "request_type")
    @Enumerated(EnumType.STRING)
    private RequestType requestType;

    // ... rest of the fields
}
```
**Why this is better:**
*   **Zero Selects:** Since `@Id` is now the auto-incrementing `id`, and it will be `null` when you call `repository.save()`, Hibernate knows it **must** be an insert. It will skip the `SELECT` entirely.
*   **Performance:** Composite keys are slower for indexing than a single `bigint`.
*   **Integrity:** You still have your `UNIQUE` constraint in the DB (`CONSTRAINT tpt_limit_ref_id_request_type_key`) which prevents duplicate data.

### Architectural Recommendation:
1.  **If you cannot change the Entity structure:** Implement `Persistable` as shown in Step 1. This is a common pattern for high-performance systems to "force-insert."
2.  **If you can change the Entity structure:** Switch to the `Long id` as the `@Id` (Step 2). In a Payment Factory, using a surrogate `Long` Primary Key is standard practice for performance.
3.  **Batch Inserts:** Once the `SELECT` is gone, you can now enable Hibernate Batch Inserts:
    ```properties
    spring.jpa.properties.hibernate.jdbc.batch_size=50
    spring.jpa.properties.hibernate.order_inserts=true
    ```
    This will allow you to save 50 audit logs in **one single network trip** to the database, drastically reducing your Dynatrace latency numbers.        String refId = incomingPayload.getReferenceId();
        
        log.info("[SRV][REQ-FETCH][START] Initiating flow for RefID: {}", refId);

        // 1. Business Validation (Example)
        if (incomingPayload.getMobileNo() == null) {
            throw new PaymentValidationException("Mobile number is mandatory for ReqFetch");
        }

        // 2. Persist using the Framework 
        // Logic: DB Save -> Cache Save (Write-Through)
        reqFetchDataManager.save(incomingPayload);

        log.info("[SRV][REQ-FETCH][SUCCESS] Initial request anchored for RefID: {}", refId);
    }

    /**
     * Subsequent API Call: e.g., Generate OTP API
     * Purpose: Retrieves data to validate the session.
     */
    public void prepareOtpGeneration(String referenceId) {
        log.info("[SRV][OTP-GEN][START] Preparing OTP for RefID: {}", referenceId);

        // 3. Fetch using the Framework
        // Logic: Cache Hit? -> Yes (Fast) | No -> DB (Fallback)
        IssuerFetchRequestTable sessionData = reqFetchDataManager.find(referenceId);

        if (sessionData == null) {
            log.error("[SRV][OTP-GEN][FAIL] No session found for RefID: {}", referenceId);
            throw new SessionExpiredException("Transaction session not found or expired");
        }

        // 4. Use the data (e.g., getting the mobile number for OTP)
        String mobile = sessionData.getMobileNo();
        log.info("[SRV][OTP-GEN][PROCEED] Sending OTP to mobile: {}", mobile);
        
        // ... proceed with OTP logic
    }
}
```

---

### Why this usage is "Elite" Level:

1.  **Cleaner Injection:** By using `private final DataOrchestrator<IssuerFetchRequestTable> reqFetchDataManager`, Spring automatically matches the `@Component("ReqFetchDataManager")` we created earlier.
2.  **Focus on Business Logic:** Notice the Service doesn't have any `try-catch` for the database or any code related to Aerospike. It only deals with `Validation`, `Session Checks`, and `Process Flow`.
3.  **Automatic Performance:** When `initiateRequestFetch` calls `.save()`, the user's data is immediately pushed to Aerospike. When the next API calls `.find()`, the response is **sub-millisecond** because the cache is already "warm."
4.  **Correlation Traceability:** Every log in the Service starts with `[SRV]`, and every log in the Manager starts with `[START/HIT/SUCCESS]`. In your logs, a single transaction will look like a perfect story:

#### The Log Story in Production:
```text
[SRV][REQ-FETCH][START] Initiating flow for RefID: TXN_101
[START][REQ-FETCH][SAVE-TXN] ID: TXN_101 | Write-Through Initiated
[SUCCESS][REQ-FETCH][SAVE-TXN] ID: TXN_101 | DB & Cache Synchronized
[SRV][REQ-FETCH][SUCCESS] Initial request anchored for RefID: TXN_101

... (few seconds later) ...

[SRV][OTP-GEN][START] Preparing OTP for RefID: TXN_101
[START][REQ-FETCH][FETCH-TXN] ID: TXN_101 
[HIT][REQ-FETCH][FETCH-TXN] ID: TXN_101 | Aerospike sub-ms retrieval
[SRV][OTP-GEN][PROCEED] Sending OTP to mobile: 9876543210
```

### Master's Final Advice for the Framework Usage:
When your colleagues start using this, tell them: **"The Manager is the guard of the data; the Service is the commander of the flow."** 

If you follow this implementation, your Payment System will not only be the fastest in your department, but it will be the easiest to debug during production issues. You've successfully implemented a **Level 5 Mature Architecture.**
