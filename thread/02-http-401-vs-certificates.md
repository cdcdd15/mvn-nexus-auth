# 02 — Maven HTTP 401: authentication or certificates?

**Original question:** Received `401` on `mvn clean install`. Can certificate problems be ruled out?

## Key distinction

An actual **HTTP 401 Unauthorized** means Maven received an HTTP response. With HTTPS, that normally means the TLS handshake succeeded for that specific connection. A certificate trust failure normally prevents HTTP communication and produces a Java SSL exception instead.

| Symptom | Typical interpretation |
|---|---|
| `HTTP 401 Unauthorized` | Missing, invalid, rejected, or misapplied credentials |
| `HTTP 403 Forbidden` | Authorization/policy restriction (implementations vary) |
| `PKIX path building failed` | JVM cannot build a trusted certificate chain |
| `SSLHandshakeException` | TLS handshake/certificate/protocol issue |

## Caveats

- The 401 could come from a reverse proxy, SSO gateway or other intermediary, not Nexus itself.
- Maven may contact several hosts (Nexus, proxy, redirects); success on one TLS connection does not validate all others.
- Mutual TLS/corporate gateway rules can complicate diagnosis.

## Practical diagnostic order

1. Read the failing repository **URL and HTTP status** in Maven error output.
2. Confirm `<server><id>` corresponds to the actual repository/mirror ID.
3. Confirm `settings.xml` and any encryption/security file Maven is loading.
4. Test **the exact failing artifact URL** using `curl` and the Nexus token credentials.
5. Investigate Java certificates if there are TLS/PKIX errors, different endpoints, or unresolved truststore questions.

```bash
mvn -version
mvn clean install -U -e
mvn -X clean install
```

Debug logs may contain sensitive details; sanitize before sharing.

**Conclusion:** A 401 is principally an HTTP authentication investigation, not first evidence of a certificate failure.
