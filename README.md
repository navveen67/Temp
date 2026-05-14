As a Senior Architect, I have refactored your Feign Logging logic to match the "Correlation Standard." In a high-traffic payment system, the **ReferenceId** must be the first identifier in every log line to ensure that SRE and Support teams can trace a transaction's lifecycle across different microservices using a single Splunk/ELK query.

Here is the production-ready, standardized Feign Logger implementation.

### 1. Refactored Feign Logging Logic

I have integrated the `ReferenceId` as the primary anchor for each log entry, maintaining your security standards for Log Forging and Data Masking.

```java
// 1. Initial Request Logging
log.info("Making Feign request for ReferenceId {} to URL: {} with Method: {} and ID: {}",
    referenceId, 
    CommonUtils.validateForLogForging(request.url()), 
    request.httpMethod(), 
    reqId);

// 2. Request Body Logging (Debug level)
log.debug("Request for ReferenceId {} with ID {} Body: {}", 
    referenceId, 
    reqId,
    CommonUtils.validateForLogForging(SensitiveDataMasker.maskSensitiveJson(bodyText)));

// 3. Detailed Response Logging (Status + Body)
log.debug("Response for ReferenceId {} with ID {} - Status Code = {}, Body = {}",
    referenceId,
    reqId,
    response.status(),
    CommonUtils.validateForLogForging(maskedAndTruncatedBody));

// 4. Summary Response Logging (Status only - useful for large payloads or errors)
log.debug("Response for ReferenceId {} with ID {} - Status Code = {}", 
    referenceId, 
    reqId, 
    response.status());

// 5. Performance/Timing Logging
log.debug("Total time taken for ReferenceId {} and Feign API {} with ID {}: Time = {}ms",
    referenceId,
    CommonUtils.validateForLogForging(response.request().url()),
    reqId,
    duration);
```

---

### 2. Architect's Implementation Strategy:

Since Feign Loggers usually run in a generic context, you must ensure the `referenceId` is available within the logger scope.

**Recommendation A: Using MDC (Standard Approach)**
If you are using **MDC (Mapped Diagnostic Context)** to store your `referenceId` at the start of the transaction, you should fetch it like this inside your custom Feign Logger:

```java
String referenceId = org.slf4j.MDC.get("referenceId"); 
// Default to "N/A" if missing to prevent null in logs
referenceId = (referenceId != null) ? referenceId : "UNKNOWN";
```

**Recommendation B: ThreadLocal / Header**
If you are passing the `referenceId` as a custom header in Feign, you can extract it from the `request.headers()` collection before logging.

---

### 3. Why this is the "Elite" Standard:

1.  **Uniform Traceability:** By putting `ReferenceId` at the start of the message, you make the logs "Greppable." You can run `grep "REF12345"` and see both your internal logic and your Feign network calls in exact chronological order.
2.  **Security Compliance:** Using `CommonUtils.validateForLogForging` is critical in banking to prevent **Log Injection** attacks where a malicious payload attempts to write fake log entries.
3.  **Data Privacy:** The `SensitiveDataMasker` ensures that PII (Personally Identifiable Information) like card numbers or OTPs are never written to the disk in plain text.
4.  **Performance Visibility:** The "Total time taken" log is vital for detecting **latency creep** in 3rd party bank APIs (like NPCI or HDFC core).

**Architect's Note:** Ensure that your `log.info` and `log.debug` levels are correctly configured in your `logback-spring.xml` or OpenShift ConfigMap, so you don't overwhelm the disk with debug bodies in the production environment unless active troubleshooting is required.
