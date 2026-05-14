As a Senior Architect, I agree with your logic. In a high-traffic environment, particularly with **GET** requests (which have no body) or **Large File Uploads**, forcing the URL and Body into the same log line causes issues.

By splitting these into **three granular methods**, you achieve "Progressive Logging." If the connection fails *after* the URL is logged but *before* the body is sent, you still have the trace of where the request was going.

### The Standardized "Three-Point" Network Logger

We will maintain the `[NETWORK-OUTBOUND]` anchor tag to keep the logs searchable.

```java
@Slf4j
public class OutboundLoggerUtility {

    private static final String TAG = "[NETWORK-OUTBOUND]";

    /**
     * 1. Log Request Metadata (URL, Method, Client)
     * Level: INFO (Always logged for traceability)
     */
    public static void logRequestInitiation(String refId, String reqId, String clientType, String method, String url) {
        log.info("{} [REF: {}][ID: {}] START | {} {} | Client: {}", 
            TAG, refId, reqId, method, CommonUtils.validateForLogForging(url), clientType);
    }

    /**
     * 2. Log Request Payload
     * Level: DEBUG (Masked for security)
     */
    public static void logRequestBody(String refId, String reqId, String body) {
        if (log.isDebugEnabled() && body != null && !body.isEmpty()) {
            log.debug("{} [REF: {}][ID: {}] REQ-BODY: {}", 
                TAG, refId, reqId, 
                CommonUtils.validateForLogForging(SensitiveDataMasker.maskSensitiveJson(body)));
        }
    }

    /**
     * 3. Log Response (Status, Duration, and Body)
     * Level: INFO (Status/Time) | DEBUG (Body)
     */
    public static void logResponse(String refId, String reqId, int statusCode, String responseBody, long duration) {
        // Log the summary at INFO level
        log.info("{} [REF: {}][ID: {}] END | Status: {} | Time: {}ms", 
            TAG, refId, reqId, statusCode, duration);

        // Log the body at DEBUG level
        if (log.isDebugEnabled() && responseBody != null && !responseBody.isEmpty()) {
            log.debug("{} [REF: {}][ID: {}] RESP-BODY: {}", 
                TAG, refId, reqId, 
                CommonUtils.validateForLogForging(SensitiveDataMasker.maskSensitiveJson(responseBody)));
        }
    }
}
```

---

### Why this "Three-Point" Strategy is Superior:

1.  **Handles GET vs POST naturally:** 
    *   For a **GET** request, you simply call `logRequestInitiation` and `logResponse`. You skip the body method entirely. No "null" or "empty" junk in your logs.
2.  **Failure Forensics:** 
    *   If your external bank server hangs, your logs will show the `START` line, but won't show the `END` line. This tells you exactly where the "Black Hole" is.
3.  **Log Forging Protection:** 
    *   Every single method passes data through `validateForLogForging`, ensuring a malicious 3rd party cannot inject a newline `\n` into their response to write fake logs into your system.
4.  **Performance & Cost:**
    *   In a high-volume payment system, logging Request/Response bodies in **INFO** level is a major cost (Storage/Disk I/O). 
    *   By putting the bodies in **DEBUG**, you can keep them OFF in production to save costs, but turn them ON for a specific `referenceId` during a production issue.

---

### Example: Usage in Feign Client

```java
// Method: Request
OutboundLoggerUtility.logRequestInitiation(refId, reqId, "FEIGN", "POST", request.url());

if(request.body() != null) {
    OutboundLoggerUtility.logRequestBody(refId, reqId, request.body().toString());
}

// Method: Response
OutboundLoggerUtility.logResponse(refId, reqId, response.status(), bodyText, duration);
```

### Example: Usage in WebClient

```java
// Initiation
OutboundLoggerUtility.logRequestInitiation(refId, reqId, "WEBCLIENT", method, url);

// Payload (Only if exists)
OutboundLoggerUtility.logRequestBody(refId, reqId, jsonBody);

// Response
OutboundLoggerUtility.logResponse(refId, reqId, statusCode, responseBody, duration);
```

### Master's Architectural Verdict:
This is the **"Final Form"** of network logging. It provides a chronological story of the network call. Any developer can search Splunk for `[NETWORK-OUTBOUND]` and immediately see the **Start -> Body -> End** lifecycle of every packet leaving your system. This is what we call **"High-Resolution Observability."**
