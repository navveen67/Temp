For a high-stakes payment gateway, naming is everything. You want a name that screams **"Validation Only"** so no developer ever confuses it with actual transaction data.

The best naming convention here is **"Request Trace"** or **"Validation Context."** It implies we are leaving a "trace" behind to verify the callback later.

### 1. The DTO: `NbblRequestTraceContext`
This name clearly indicates that the object is a "Context" for "Tracing" a request.

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class NbblRequestTraceContext implements Serializable {
    private static final long serialVersionUID = 1L;

    private String referenceId;
    
    // Clearly named for Idempotency/Duplicate Check
    private boolean isProcessed; 

    private long requestTimestamp;
}
```

### 2. The Data Manager: `NbblRequestTraceDataManager`
By naming it "Trace Data Manager," you distinguish it from your "Response Data Manager."

```java
@Component
@RequiredArgsConstructor
public class NbblRequestTraceDataManager extends AbstractCacheOnlyManager<NbblRequestTraceContext> {

    private final AerospikeUtil aerospikeUtil;
    private final CacheTtlConfig ttlConfig;

    @Override
    protected String getDomain() {
        // Understandable Logging Tag for Production Logs
        return "NBBL-CALLBACK-TRACE";
    }

    @Override
    protected String getIdentifier(NbblRequestTraceContext e) {
        return e.getReferenceId();
    }

    @Override
    protected void cacheWrite(String id, NbblRequestTraceContext d) {
        // Set this to 900 (15 minutes) in your config
        aerospikeUtil.addUpdateCache(
            AerospikeConstants.NBBL_CALLBACK_TRACE_CACHE, 
            id, 
            ttlConfig.getNbblTraceTtl(), 
            d
        );
    }

    @Override
    protected NbblRequestTraceContext cacheRead(String id) {
        return aerospikeUtil.getRecord(
            AerospikeConstants.NBBL_CALLBACK_TRACE_CACHE, 
            id, 
            NbblRequestTraceContext.class
        );
    }
}
```

---

### 3. Usage Pattern: Validation Flow

This logic ensures your "futureStep1" and "futureStep2" are handled in one go with very clear logging.

#### When sending the Request (The "Trace" creation):
```java
public void initiateRequest(String refId) {
    NbblRequestTraceContext trace = NbblRequestTraceContext.builder()
            .referenceId(refId)
            .isProcessed(false)
            .requestTimestamp(System.currentTimeMillis())
            .build();
            
    traceDataManager.save(trace);
    log.info("[NBBL-CALLBACK-TRACE][INIT] Trace created for RefId: {}", refId);
}
```

#### When receiving the Callback (The "Validation" check):
```java
public void validateCallback(String refId) {
    // 1. Fetch the Trace Context
    NbblRequestTraceContext trace = traceDataManager.find(refId);

    // STEP 1: VALIDATE OWNERSHIP (Does it belong to us?)
    if (trace == null) {
        log.error("[NBBL-CALLBACK-TRACE][REJECT] RefId {} not found. This request did not originate from this system.", refId);
        throw new SecurityException("Unauthorized Reference ID");
    }

    // STEP 2: VALIDATE DUPLICATION (Has it been processed?)
    if (trace.isProcessed()) {
        log.warn("[NBBL-CALLBACK-TRACE][DUPLICATE] RefId {} has already been processed. Ignoring duplicate callback.", refId);
        return; // Exit to avoid double processing
    }

    // SUCCESS: Proceed and Lock
    trace.setProcessed(true);
    traceDataManager.save(trace);
    
    log.info("[NBBL-CALLBACK-TRACE][SUCCESS] RefId {} validated successfully. Proceeding to business logic.", refId);
}
```

---

### Why these names work:

1.  **Domain Tag (`NBBL-CALLBACK-TRACE`):** When you search your logs in Splunk or ELK, this tag will filter out all the noise and show you exactly the validation lifecycle.
2.  **DTO Name (`NbblRequestTraceContext`):** No one will confuse "TraceContext" with "ResponseData." It sounds temporary, which fits the 15-minute TTL.
3.  **Boolean Flag (`isProcessed`):** It is a standard industry term for idempotency checks. It clearly tells the developer: "The callback has already been handled."

### Task for your developers:
*   **Dev 1:** Update `AerospikeConstants` to include `NBBL_CALLBACK_TRACE_CACHE`.
*   **Dev 2:** Update `CacheTtlConfig` with `nbblTraceTtl = 900`.
*   **Both:** Implement the `NbblRequestTraceDataManager` to replace the manual `CompletableFuture` validation steps.

This architecture is clean, highly performant, and easy for your production support team to monitor!
