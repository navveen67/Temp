package com.hdfc.ampsissuer.helper;

import com.hdfc.ampsissuer.config.YamlConfig; // Adjust package as per your project
import com.hdfc.ampsissuer.constants.AMPSIssuerConstants;
import com.hdfc.ampsissuer.crypto.AESGCMEncDecAlgorithm;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

@Slf4j
@Component
public class MobileNumberHelper {

    private final AESGCMEncDecAlgorithm aesGCMEncDecAlgorithm;
    private final YamlConfig yamlConfig;

    // Constructor injection
    public MobileNumberHelper(AESGCMEncDecAlgorithm aesGCMEncDecAlgorithm, YamlConfig yamlConfig) {
        this.aesGCMEncDecAlgorithm = aesGCMEncDecAlgorithm;
        this.yamlConfig = yamlConfig;
    }

    /**
     * Resolves the mobile number by attempting decryption. 
     * Handles plain text and malformed encrypted data gracefully.
     */
    public String resolveMobNumber(String mobNoEnc) {
        // 1. Basic Null/Empty Check
        if (mobNoEnc == null || mobNoEnc.trim().isEmpty()) {
            log.warn("Mobile number is null or empty. Using default from config.");
            return yamlConfig.getMobileno();
        }

        // 2. Length Check (Logic: Encrypted strings are significantly longer than 15 chars)
        if (mobNoEnc.length() < 15) {
            log.info("Mobile number [{}...] appears to be plain text. Cleaning and returning.", 
                     mobNoEnc.substring(0, Math.min(mobNoEnc.length(), 5)));
            return cleanString(mobNoEnc);
        }

        // 3. Decryption Attempt
        try {
            String decryptedMob = aesGCMEncDecAlgorithm.decrypt(mobNoEnc, yamlConfig.getAesKey());

            if (decryptedMob != null && !decryptedMob.trim().isEmpty()) {
                return cleanString(decryptedMob);
            } else {
                log.warn("Decryption returned empty string. Using default from config.");
                return yamlConfig.getMobileno();
            }

        } catch (Exception e) {
            // Catches Base64 (IllegalArgumentException) and AES (BufferUnderflowException)
            log.error("Decryption failed for value: [{}]. Reason: {}. Falling back to default mobile number.",
                    mobNoEnc, e.getMessage());
            return yamlConfig.getMobileno();
        }
    }

    /**
     * Internal utility to remove quotes and trim spaces
     */
    private String cleanString(String input) {
        if (input == null) return null;
        return input.replace(AMPSIssuerConstants.QUOTE, AMPSIssuerConstants.EMPTY).trim();
    }
}
