# 04 — Credentials vs TLS vs Maven authentication

**Original question:** Explain the three distinctions: credentials, TLS and Maven authentication.

## 1. Credentials: identity and permissions

A Nexus account, user token or token username/password pair identifies a client. The server validates the supplied credential and checks whether that identity can access the resource.

Basic authentication typically carries an HTTP header containing base64-encoded `username:password`; **base64 is not encryption**. HTTPS protects the header in transit.

## 2. TLS: secure transport and server identity

Before sending an HTTPS request, the client negotiates TLS and validates the server's certificate chain against trusted CAs.

- Curl uses its configured TLS backend and CA trust configuration.
- Maven runs on the JVM; Java typically uses the configured Java truststore, often `cacerts`.

Curl trusting a certificate does **not** establish Java/Maven trust.

## 3. Maven authentication: credential selection and transmission

Maven identifies which repository or mirror it is contacting, looks up the `<server>` entry with the same ID, decrypts an encrypted password locally when configured, and provides credentials for HTTP authentication.

```xml
<mirror>
  <id>company-nexus</id>
  <mirrorOf>*</mirrorOf>
  <url>https://nexus.company.com/repository/maven-public/</url>
</mirror>
<server>
  <id>company-nexus</id>
  <username>TOKEN_USERNAME</username>
  <password>TOKEN_PASSWORD</password>
</server>
```

## Summary

| Layer | Question | Main failure example |
|---|---|---|
| Credentials/Nexus | Is this identity valid/allowed? | HTTP 401/403 |
| TLS/HTTPS | Is the connection secure and the server trusted? | `PKIX` / `SSLHandshakeException` |
| Maven auth configuration | Did Maven select and send the right credential? | HTTP 401 despite successful direct curl |

A real Maven 401 normally implies TLS completed for that HTTP request; start with the Maven server ID and credentials.
