This is a classic "Race Condition" and "State Synchronization" issue in Payment Gateways. Based on your code and logs, there are **three critical reasons** why this is happening, ranging from a major architectural bug to a logical flow error.

### 1. The Architectural Bug: `MessageDigest` Thread Safety
This is likely your "Silent Killer."
*   **The Issue:** You have declared `private MessageDigest messageDigest` as a class-level field in a `@Component`.
*   **The Danger:** In Spring, `@Component` is a **Singleton**. `MessageDigest` is **NOT thread-safe**. 
*   **The Consequence:** In a high-traffic "Payment Factory," if two customers (Customer A and Customer B) generate or verify an OTP at the same millisecond, they will share the same `messageDigest` instance. Thread A will call `.update()`, and Thread B will call `.digest()`. 
*   **The Result:** The `messageHash` becomes **corrupted**. When you send a corrupted hash to the 3rd party (HDFC/OBP), their system cannot find a matching record for that hash and returns **"No password in the system."**

**Fix:** Initialize `MessageDigest` inside the method or use a `ThreadLocal`.
```java
// Instead of a class field, do this inside the digestIt method:
public byte[] digestIt(byte[] dataIn) {
    try {
        return MessageDigest.getInstance(SHA_VERSION).digest(dataIn);
    } catch (NoSuchAlgorithmException e) { ... }
}
```

---

### 2. The "Resend OTP" Logical Invalidation
Looking at your scenario: *1st Gen -> Invalid Verify -> Wait 2 mins -> Resend -> Enter Valid.*

*   **The Issue:** When the user clicks **Resend**, the 3rd-party OBP system generates a **new** OTP and **deletes/invalidates the old OTP** immediately to prevent fraud.
*   **The Wait Time Problem:** If the user clicked "Resend" and then entered the **OTP from the first SMS** (because the second SMS was delayed), the OBP system will return "No password in the system" because that first OTP was purged the moment "Resend" was triggered.
*   **Wait... check the logs:** Your logs show two `Verify` calls but **only one `Generate` call.**
    *   `16:03:53` -> Generate
    *   `16:04:03` -> Verify (Attempt 1)
    *   `16:06:14` -> Verify (Attempt 2)
*   **Observation:** There is no log for the "Resend" (Generate API) between the two Verify calls. If the user *thought* they clicked Resend but the request never reached your server, they are trying to verify against a stale session.

---

### 3. Verification of your "3-Minute" Theory
> *If i generate otp and wait for more than 3 mins then for 1st verify otp it will fail... correct?*

**No, not necessarily.** If your config says `expiryMin: 5`, it should last 5 minutes. **HOWEVER**, there is a hidden HDFC OBP behavior:

*   **OBP Session vs. OTP Expiry:** While the OTP might be valid for 5 minutes, the **Internal OBP Session** that tracks the `refNo` often has a shorter timeout (typically 180 seconds / 3 minutes). 
*   **If the session expires before the OTP,** the system "forgets" the hash associated with that `refNo`. When you call `Verify`, the system returns "No password in the system" because the session record was purged.

---

### 4. Step-by-Step Task for your Team to Fix This:

| Task | Action |
| :--- | :--- |
| **Fix Thread Safety** | Move `MessageDigest` initialization inside the `hash()` method. Stop sharing it across threads. |
| **Verify Hash Logic** | Ensure `otpProperties.getHashKey()` is not being modified or reloaded mid-transaction. |
| **Check OBP Session** | Ask the HDFC/OBP team what the **"Idle Session Timeout"** is. It is likely 180 seconds. If so, your 5-minute OTP expiry is useless. |
| **Log the Hash** | In your `log.debug`, print the **first 5 characters** of the `messageHash`. If the hash for the same user changes between two identical calls, it confirms the `MessageDigest` thread-safety bug. |

### Summary for your Confluence:
*   **Symptom:** "No password in the system" after ~3 minutes.
*   **Root Cause A:** `MessageDigest` corruption due to Singleton component (High Probability).
*   **Root Cause B:** OBP Server-side session timeout (3 mins) is shorter than your app's OTP expiry (5 mins).
*   **Root Cause C:** User using an invalidated OTP after a Resend attempt.

**Master's Advice:** Fix the `MessageDigest` thread-safety issue immediately today. It is a critical architectural flaw for a Payment Gateway.
