# Recommended diagnostic order for Maven → Nexus HTTP 401

## Context and objective

**Observed:** `mvn clean install` returns HTTP **401 Unauthorized** while `curl` appears to connect successfully to Nexus with credentials generated in the Nexus UI.

**Scope:** This guide orders five previously expanded checks by diagnostic value for this particular symptom. It is not a general priority ranking for every Maven failure. A real HTTP 401 means an HTTP server or intermediary responded, so the TLS handshake for that request normally succeeded. That makes Maven credential selection and endpoint routing more promising initial investigations than importing certificates.

## Recommended sequence at a glance

| Priority | Original check | Guide | Why at this point | Evidence of success / next move |
|---|---|---|---|---|
| 1 | **7 — Effective mirror/repository/server ID** | `07-effective-mirror-repository-server-id.md` | The most common direct explanation is Maven choosing a mirror whose ID has no matching `<server>` credentials. | Confirm the actual mirror/repository ID equals the relevant `<server><id>`. If correct, continue to #8. |
| 2 | **8 — Settings overrides (CLI/IDE/CI/wrapper)** | `08-settings-overrides-ide-ci-wrapper.md` | Even a correct `settings.xml` is useless if this Maven invocation loads a different file or settings profile. | Confirm the exact Maven invocation and effective settings; then continue to #3. |
| 3 | **3 — Maven/JVM/settings-security locations** | `03-maven-jvm-settings-security-locations.md` | Once the active settings are known, verify credentials are actually available and any encrypted value can be decrypted. JVM identification also prepares for TLS testing. | Confirm runtime, settings paths, and encrypted-password setup; continue to #9 if 401 persists. |
| 4 | **9 — Proxy/mirror/redirect routing** | `09-proxy-mirror-redirect-routing.md` | The 401 might come from a proxy, SSO gateway, redirected host, or a different Nexus endpoint than the one curl tested. | Confirm the exact artifact URL, response origin, redirects, and proxy settings; if still uncertain, continue to #5. |
| 5 | **5 — Java TLS/corporate CA truststore** | `05-java-tls-corporate-ca-truststore.md` | Important for corporate networks, but an actual 401 generally means TLS already worked on that connection. Investigate earlier if there are SSL/PKIX errors instead. | Verify the JVM's actual truststore and the trusted corporate CA chain, if TLS evidence warrants it. |

> **Companion check outside this five-item sequence:** Before making configuration changes, compare the exact artifact URL and HTTP status from `mvn -X` with an authenticated curl request for that **same artifact**, not the Nexus UI homepage. An HTML login page or an HTTP 200 from a public UI endpoint does not prove repository-token authentication.

## 1. Check 7 — Effective mirror, repository, and server ID

**Why first:** Maven selects a repository (possibly replacing it with a mirror), then looks up credentials using the **selected endpoint's ID**. The URL is not the key for matching credentials. For example, if the selected mirror is `nexus-public`, the credentials under `<server><id>nexus-releases</id>` will not automatically apply.

**Check:**

```bash
mvn -X clean install 2>&1 | grep -E 'Using mirror|Downloading from|401|Unauthorized'
```

Review the effective settings (avoid sharing potentially sensitive values):

```bash
mvn help:effective-settings
```

**Expected:** The repository/mirror actually named in the download is configured with the appropriate server ID in the effective settings. Fix mismatched IDs before proceeding.

**Caution:** Depending on active profiles and Maven configuration, different repositories may be used for dependencies, plugins, and deployments; validate the ID involved in the failing request.

## 2. Check 8 — Settings file overrides and execution context

**Why second:** The configuration you edited may not be the one your terminal, IDE, wrapper, or GitLab runner uses. Maven can be invoked with `-s` (user settings) and `-gs` (global settings); IDEs and CI jobs may use different files and environment values.

**Check:**

```bash
mvn -version
mvn help:effective-settings
```

Inspect the actual command line for `-s` / `--settings` and `-gs` / `--global-settings`. Compare execution from the same terminal and directory where the failure occurs. If the failure occurs in an IDE or CI, inspect that environment separately.

**Expected:** You can identify which user/global settings and active profiles control this execution, and they contain the intended Nexus mirror and server definition.

## 3. Check 3 — Maven/JVM and settings-security locations

**Why third:** After confirming which settings are loaded, verify that any encrypted Nexus token/password can be decrypted using the Maven version and security configuration actually in use. Maven 3 uses the legacy `settings-security.xml` mechanism; Maven 4 has a different encryption mechanism, so don't assume the same setup works unchanged.

**Check:**

```bash
mvn -version
printf 'JAVA_HOME=%s\n' "$JAVA_HOME"
```

For Maven 3, look for the user security configuration under `~/.m2/settings-security.xml`, or any explicitly configured alternate location. Confirm that `settings.xml` contains the correct **Nexus token username and token password**, if tokens were generated in the UI. Do not publish their values.

**Controlled diagnostic:** Temporarily test a known-good plaintext token password in a secure, local `settings.xml`. If that works while the encrypted value fails, focus on encryption/master-password configuration. Remove the plaintext after testing, and avoid committing secrets.

**Expected:** Maven uses the expected JVM and can read/decrypt the credentials configured for the selected server ID.

## 4. Check 9 — Proxy, mirror, redirect, and endpoint routing

**Why fourth:** A 401 can be returned by Nexus **or by an intermediary**. Curl and Maven may also use different proxy settings or different target URLs. Testing the Nexus UI is not sufficient.

**Check:** Use the *exact artifact URL* that Maven logged, and compare no-credential versus authenticated responses:

```bash
# Replace the URL and token username. Curl prompts for the token password.
URL='https://nexus.company.com/repository/maven-public/path/to/artifact.pom'

curl -sS -o /dev/null -w '%{http_code}\n' "$URL"
curl -sS -o /dev/null -w '%{http_code}\n' -u 'TOKEN_USERNAME' "$URL"
```

For troubleshooting redirects or response headers (without revealing tokens), examine `curl -I`/`curl -v` cautiously, and compare Maven's debug output. Check Maven `<proxies>` in effective settings and corporate gateway rules. Do not assume that following a redirect preserves credentials across hosts.

**Expected:** Maven and curl reach the same intended repository endpoint, and you can identify whether the 401 originates at Nexus or a proxy/SSO gateway.

**Interpretation:** `401` without credentials and `200` with token credentials on a protected artifact is strong evidence the token works for that endpoint. `200` both ways may indicate anonymous access, not proof of token validity. `404` is inconclusive about authentication.

## 5. Check 5 — Java truststore and corporate CA

**Why last for this particular case:** Certificate-chain validation usually fails **before** an HTTP response is available and normally appears as `PKIX path building failed` or `SSLHandshakeException`, not HTTP 401. Nonetheless, curl and Java can trust different certificate authorities, so a successful curl TLS connection alone does not validate Maven's truststore.

**Check:**

```bash
mvn -version
# Confirm JAVA_HOME reflects Maven's reported Java home before using this path.
ls -l "$JAVA_HOME/lib/security/cacerts"
"$JAVA_HOME/bin/keytool" -list -keystore "$JAVA_HOME/lib/security/cacerts"
```

Also inspect `MAVEN_OPTS`, `.mvn/jvm.config` (if present), and Java properties such as `-Djavax.net.ssl.trustStore=...`. Java may use `jssecacerts` or an explicitly configured truststore instead of `cacerts`. When necessary, run Maven with `-Djavax.net.debug=ssl,trustmanager` to see which trust material was loaded; restrict access to the logs.

Compare the corporate CA certificate's **SHA-256 fingerprint** with the actual trusted certificate entry; names/aliases alone are insufficient. Import a verified certificate only into the approved truststore and only when needed.

**Expected:** The exact Maven JVM trusts the certificate chain used on the HTTPS connection.

## Decision rules and reordering triggers

| Observed evidence | Next priority |
|---|---|
| HTTP `401`, no TLS exception | Start with **7 → 8 → 3 → 9 → 5** |
| `PKIX path building failed` / `SSLHandshakeException` | Move **5** to the front |
| Plaintext token works in Maven, encrypted token fails | Focus on **3** |
| IDE/CI fails but terminal Maven works | Focus on **8** |
| Curl works for a different URL than Maven | Focus on **9**, then **7** |
| Curl gets `200` with and without credentials | Verify a protected artifact; token acceptance remains unproven |
| Exact artifact returns `200` to authenticated curl but `401` to Maven | Focus on **7 → 8 → 3**, then request routing in **9** |

## Suggested end state

Document four observations before escalating to your Nexus or platform team: **(1)** the exact failing artifact URL and HTTP status; **(2)** the effective mirror/repository ID and matching server ID; **(3)** the Maven Java version and settings source; and **(4)** the authenticated curl status for that same URL. Never include raw credentials, HTTP Authorization headers, or decrypted secrets in tickets.

---

**Related guides:** `03-maven-jvm-settings-security-locations.md`, `05-java-tls-corporate-ca-truststore.md`, `07-effective-mirror-repository-server-id.md`, `08-settings-overrides-ide-ci-wrapper.md`, `09-proxy-mirror-redirect-routing.md`.
