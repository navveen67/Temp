I need a Java utility class called `SSLUtils` that builds an `SSLContext` from two PEM files:

1. `newcert.pem` — contains both the client certificate and CA certificate chain.
2. `key-dev-client.pem` — contains the client private key in PKCS#8 format.

Requirements:
- Method signature: `public static SSLContext buildSSLContextFromStrings(String combinedCertData, String clientKeyData, String clientKeyPassword)`
- Parse multiple certificates from `combinedCertData` using regex.
- First certificate is the client cert; the rest are CA certs.
- Load the private key from `clientKeyData` (unencrypted PKCS#8).
- Create a `KeyStore` with the client cert and private key.
- Create a `TrustStore` with the CA certs.
- Initialize `KeyManagerFactory` and `TrustManagerFactory`.
- Return a fully initialized `SSLContext`.

Also include helper methods:
- `parseCertificates(String pemData)` — returns a list of `X509Certificate`.
- `parsePrivateKey(String pemKeyData)` — returns a `PrivateKey`.

Use standard Java libraries only (no BouncyCastle). Make sure to handle whitespace and PEM headers correctly.
