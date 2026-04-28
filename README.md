This Confluence page is designed to onboard any developer in 5 minutes. It focuses on **intent** and **usage** rather than complex theory.

---

# 🏗️ Core Persistence Framework (SPF) Guide

## 1. Overview
The **Standard Persistence Framework (SPF)** is our centralized engine for Database and Cache operations. It ensures **High Performance**, **Automatic DB-Fallback**, and **Unified Logging** across all 40+ tables in our Payment System.

### Why use it?
*   **Zero Junk Logs:** Every log follows a searchable `[STATUS][DOMAIN][ACTION]` pattern.
*   **Performance:** Automatically handles Aerospike caching for speed.
*   **Robustness:** If the Cache is down, it silently falls back to the DB without breaking the payment.

---

## 2. Which Manager do I use?

| Scenario | Use This Abstract Class | Features |
| :--- | :--- | :--- |
| **High Frequency** (ReqFetch, OTP, Session) | `AbstractCacheAsideManager` | Cache Read -> DB Fallback -> Cache Sync |
| **Audit/History** (Logs, Status History) | `AbstractDbOnlyManager` | Direct DB Persistence (No Cache) |

---

## 3. Implementation Steps (Example: `OtpDataManager`)

### Step 1: Create the Manager
Extend the appropriate base class and implement the "Hooks."

```java
@Component
public class OtpDataManager extends AbstractCacheAsideManager<OtpEntity, OtpCacheDTO> {
    
    @Override protected String getDomain() { return "OTP-VERIFY"; }
    @Override protected String getIdentifier(OtpEntity e) { return e.getRefId(); }

    // Implement the 4 basic I/O methods:
    @Override protected void dbWrite(OtpEntity e) { repo.save(e); }
    @Override protected OtpEntity dbRead(String id) { return repo.findById(id); }
    @Override protected void cacheWrite(String id, OtpCacheDTO d) { aerospike.put(..., d); }
    @Override protected OtpCacheDTO cacheRead(String id) { return aerospike.get(...); }
    
    // Wire your Mapper
    @Override protected OtpCacheDTO mapToCacheDto(OtpEntity e) { return mapper.toDto(e); }
    @Override protected OtpEntity mapToEntity(OtpCacheDTO d) { return mapper.toEntity(d); }
}
```

---

## 4. Usage in your Service
**Rule:** Never inject the Repository or AerospikeUtil directly into your Service. **Only inject the Orchestrator.**

```java
@Service
public class PaymentService {

    // Inject using the interface and the Bean Name
    @Autowired 
    private DataOrchestrator<OtpEntity> otpDataManager;

    public void verify(String refId) {
        // Fetch: Automatic Cache -> DB logic
        OtpEntity otp = otpDataManager.find(refId);

        // Save: Automatic DB -> Cache logic
        otpDataManager.save(otp);
    }
}
```

---

## 5. How to Track in Logs (Splunk/ELK)
All operations generate standardized "Breadcrumbs." Search by your Domain tag:

*   `[START][OTP-VERIFY][FETCH]` : Request initiated.
*   `[HIT][OTP-VERIFY][FETCH]` : Data served from Aerospike (Fast!).
*   `[MISS][OTP-VERIFY][FETCH]` : Data fetched from DB (Slow).
*   `[SUCCESS][OTP-VERIFY][SAVE]` : DB and Cache are in sync.
*   `[CRITICAL-FAIL][OTP-VERIFY][SAVE]` : Database error (Check RDS logs).

---

## 🛑 Rules of the Framework
1.  **Naming:** Domain names must be UPPERCASE (e.g., `REQ-FETCH`, `PAY-AUDIT`).
2.  **No Logic in Managers:** Managers are for Data Access only. Put Business Logic in the `@Service`.
3.  **Fail-Safe:** Base classes already handle `try-catch`. Do not add extra `try-catch` blocks inside your Manager implementation unless specifically required.

---
**Questions?** Contact the System Architect.
