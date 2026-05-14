As a Senior Architect, I understand the need for **Granular Forensic Logging**. By decoupling the **Response Metadata**, the **Response Payload**, and the **Performance Metrics**, we allow the system to log exactly what is available at each stage of the lifecycle.

This is especially useful if a connection is severed midway—you might get the performance timing but not a body, or a status code but the body is too large to log.

### The Granular "Five-Point" Network Logger

Here is the refined, production-ready utility with dedicated methods for maximum granularity.

```java
@Slf4j
public class OutboundLoggerUtility {

    private static final String TAG = "[NETWORK-OUTBOUND]";

    /**
     * 1. Log Request Initiation (Metadata)
     */
    public static void logRequestInitiation(String refId, String reqId, String clientType, String method, String url) {
        log.info("{} [REF: {}][ID: {}] START | {} {} | Client: {}", 
            TAG, refId, reqId, method, CommonUtils.validateForLogForging(url), clientType);
    }

    /**
     * 2. Log Request Payload (Separated)
     */
    public static void logRequestBody(String refId, String reqId, String body) {
        if (log.isDebugEnabled() && body != null && !body.isEmpty()) {
            log.debug("{} [REF: {}][ID: {}] REQ-BODY: {}", 
                TAG, refId, reqId, 
                CommonUtils.validateForLogForging(SensitiveDataMasker.maskSensitiveJson(body)));
        }
    }

    /**
     * 3. Log Response Status (Separated)
     */
    public static void logResponseStatus(String refId, String reqId, int statusCode) {
        log.info("{} [REF: {}][ID: {}] END | Status Code: {}", 
            TAG, refId, reqId, statusCode);
    }

    /**
     * 4. Log Response Payload (Separated)
     */
    public static void logResponseBody(String refId, String reqId, String responseBody) {
        if (log.isDebugEnabled() && responseBody != null && !responseBody.isEmpty()) {
            log.debug("{} [REF: {}][ID: {}] RESP-BODY: {}", 
                TAG, refId, reqId, 
                CommonUtils.validateForLogForging(SensitiveDataMasker.maskSensitiveJson(responseBody)));
        }
    }

    /**
     * 5. Log Performance Metrics (Separated)
     * Dedicated to: Performance Tuning & Profiling
     */
    public static void logPerformanceMetrics(String refId, String reqId, String url, long duration) {
        log.info("{} Total time taken for ReferenceId {} and URL {} with ID {}: Time = {}ms",
            TAG, refId, CommonUtils.validateForLogForging(url), reqId, duration);
    }
}
```

---

### Why this "Five-Point" Granularity is Elite:

1.  **Debugging vs. Profiling:** 
    *   If you only care about **latency**, you look for the `logPerformanceMetrics` line. 
    *   If you are debugging a **data issue**, you look for `logResponseBody`. 
    *   In a high-traffic environment, you can keep `INFO` level for status and timing, while keeping the heavy `DEBUG` bodies completely disabled until needed.
2.  **Handling "Partial" Failures:** 
    *   In a scenario where a 3rd party bank returns a `500 Internal Server Error` with **no body**, your code will still call `logResponseStatus`. The granular `logResponseBody` simply won't execute, keeping your logs clean of "null" or "empty" markers.
3.  **Traceability:** 
    *   By having a dedicated performance method, you can easily create **Splunk Alerts**. For example: "Alert me if `logPerformanceMetrics` for URL 'X' shows `Time > 5000ms`". This is much easier to parse than trying to extract a number from a long combined string.
4.  **Async Friendly:** 
    *   If you decide to log bodies asynchronously (to save main-thread time), having separate methods makes it easy to wrap only the "Body" methods in a background task while keeping the "Status" and "Timing" on the main thread for immediate visibility.

---

### Final Implementation Flow for your Service:

```java
long startTime = System.currentTimeMillis();

// 1. Start Trace
OutboundLoggerUtility.logRequestInitiation(refId, reqId, "FEIGN", "POST", url);

// 2. Log Body (If exists)
OutboundLoggerUtility.logRequestBody(refId, reqId, requestJson);

try {
    // ... Actual Network Execution ...
    int status = response.status();
    String body = response.body();

    // 3. Log Status
    OutboundLoggerUtility.logResponseStatus(refId, reqId, status);

    // 4. Log Body
    OutboundLoggerUtility.logResponseBody(refId, reqId, body);

} finally {
    // 5. Always log performance, even if an exception occurs
    long duration = System.currentTimeMillis() - startTime;
    OutboundLoggerUtility.logPerformanceMetrics(refId, reqId, url, duration);
}
```

**Architect's Verdict:** This is the most **Atomic** approach to logging. Each method has a **Single Responsibility**. This structure is highly robust and perfectly optimized for the **Performance Tuning** purpose you mentioned at the beginning of our session. You now have the ultimate observability tool for your payment system.
