To make your logs perfectly searchable using the **ReferenceId** and **URL** while keeping your "sentence-style" structure, we will prioritize the **ReferenceId** at the beginning of each sentence and move the **RequestId** to a secondary position.

Here are the optimized versions of your loggers:

### 1. Request Body Logger
**Goal:** Connect the ReferenceId and the content immediately.

```java
log.debug("Request for ReferenceId {} with ID {} Body: {}", 
    referenceId, 
    requestId, 
    SensitiveDataMasker.maskSensitiveJson(CommonUtils.convertObjectToJsonString(payload))
);
```

### 2. Response Logger
**Goal:** Track the status and body of a specific transaction instantly.
*Note: I simplified the Status Code part slightly to keep the sentence clean.*

```java
log.debug("Response for ReferenceId {} with ID {} - Status Code = {}, Body = {}", 
    referenceId, 
    requestId, 
    response.getStatusCode(), 
    SensitiveDataMasker.maskSensitiveJson(CommonUtils.convertObjectToJsonString(response.getBody()))
);
```

### 3. API Timing Logger
**Goal:** Link the ReferenceId directly to the URL and the Time Taken for fast performance auditing.

```java
log.debug("Total time taken for ReferenceId {} and API {} with ID {}: Time = {}ms", 
    referenceId, 
    CommonUtils.validateForLogForging(url), 
    requestId, 
    (System.currentTimeMillis() - startTime)
);
```

---

### Why this solves your search problem:

1.  **"Anchored Search" Pattern:** In production logs (Splunk/ELK), you can now search for:
    `"ReferenceId PHT02611" "URL: http://nbbl/pay"`
    Because **ReferenceId** is now the first variable in almost every sentence, it acts as a "Primary Key" for your log files.

2.  **No Variable Interference:** Previously, your `requestId` was in the middle, acting like "noise." By moving it after the ReferenceId, you can search for the transaction flow without needing to know the internal Request ID.

3.  **Human Context:** These still read like natural sentences, which fits your project's coding standards, but they are architecturally structured for **Indexing**.

---

### Architect's Performance Tip for the Timing Logger:

Since you are using Java 21 and Spring Boot 3.4, `System.currentTimeMillis() - startTime` is fine, but for **High Precision** in a Payment Factory, consider using `System.nanoTime()` or the Spring **`StopWatch`** class.

**Optional "Master" timing code:**
```java
StopWatch watch = new StopWatch();
watch.start();
// ... API CALL ...
watch.stop();

log.debug("Total time taken for ReferenceId {} and API {} with ID {}: Time = {}ms", 
    referenceId, url, requestId, watch.getTotalTimeMillis());
```
*The `StopWatch` is cleaner and easier for your developers to use consistently across the 4,500-line file.*

### Summary for your Team Today:
Give these patterns to your two developers as the **"Standard Logging Template."** Tell them:
> *"Every log must start its variable sequence with ReferenceId, then URL (if applicable), then RequestID. This ensures we can search the entire lifecycle of a payment in one click."*
