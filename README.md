String mobNoEnc = fetchOriginatorInformation.getMobNoEnc();
String mobNum;

// 1. Check if the string is null or too short to be valid AES-GCM data
// (Encrypted + Base64 strings are usually much longer than 15 characters)
if (mobNoEnc == null || mobNoEnc.length() < 15) {
    log.warn("Mobile number is plain text or invalid length: [{}]. Using default mobile number from config.", mobNoEnc);
    mobNum = yamlConfig.getMobileno();
} else {
    try {
        // 2. Attempt decryption
        mobNum = aesGCMEncDecAlgorithm.decrypt(mobNoEnc, yamlConfig.getAesKey());
        
        // Optional: If decryption returns null/empty, fallback to default
        if (mobNum == null || mobNum.isEmpty()) {
            mobNum = yamlConfig.getMobileno();
        }
    } catch (Exception e) {
        // 3. Catch Base64 (IllegalArgumentException) or Decryption (BufferUnderflow) errors
        log.error("Decryption failed for value: [{}]. Error: {}. Falling back to default mobile number.", 
                  mobNoEnc, e.getMessage());
        mobNum = yamlConfig.getMobileno();
    }
}

// Now use mobNum for the rest of your process
