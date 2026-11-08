# Maven ↔ Nexus HTTP 401: Ten-Check Troubleshooting Guide

**Scenario:** `curl` can retrieve a protected Nexus artifact with a username/password or Nexus user token, but `mvn clean install` reports HTTP `401 Unauthorized`.

**Goal:** Find whether the discrepancy comes from the URL, credentials, mirror/server ID selection, settings files, proxy routing, or (less likely for a true HTTP 401) TLS trust.

> **Security:** Do not commit real tokens, passwords, `settings-security.xml`, unredacted effective settings, or debug logs. Prefer an interactive `curl -u 'TOKEN_USERNAME'` prompt rather than placing a password in command history. All URLs, usernames, aliases, and artifact coordinates below are illustrative; replace them with your actual values.

## Summary: ten checks

| # | Name | Description / scope | Command or evidence | Expected result | If it fails / next action |
|---|---|---|---|---|---|
| 1 | Capture the failing URL and status | Determine exactly which endpoint and operation return 401 | `mvn clean install -e`; inspect `from/to ... (https://...)` | HTTP 401 tied to an identifiable repository and artifact URL | Fix wrong endpoint or distinguish artifact download from plugin resolution |
| 2 | Match repository/mirror and server IDs | Confirm Maven has an authentication entry for the effective remote repository | Compare `<mirror><id>` / `<repository><id>` to `<server><id>` | IDs match exactly | Correct the server ID; URL equality does not match credentials |
| 3 | Confirm Maven JVM and settings/security locations | Find the actual Maven installation, Java home, user/global settings, and encryption configuration | `mvn -version`; Maven `-X`; settings under `~/.m2/` | Correct files and Maven version in use | Pass `-s` explicitly or repair settings/security path |
| 4 | Curl the exact artifact URL | Test the same protected file, not just the Nexus UI or repository homepage | `curl -sS -o /dev/null -w '%{http_code}\n' -u 'TOKEN_USERNAME' 'ARTIFACT_URL'` | `200` (for a known existing artifact), with anonymous request ideally `401` | Confirm URL, permissions, token type, and redirects |
| 5 | Check TLS/truststore only when indicated | Determine whether Java trusts the corporate CA and is using the expected truststore | `mvn -version`; `keytool -list`; optionally JSSE debug | No PKIX/handshake error; trusted CA chain | Inspect actual JVM truststore or custom truststore; 401 normally implies TLS worked |
| 6 | Isolate encrypted-password handling | Briefly test plaintext password in local Maven settings | Controlled edit to `<server><password>` | Plaintext works if decryption was the problem | Restore safe config; if still 401, inspect ID/URL/proxy/token |
| 7 | Verify **effective** ID mapping | Check IDs after profiles, mirror selection, parent POMs, and effective settings are applied | `mvn help:effective-settings`; `mvn -X` mirror lines | The ID Maven actually connects through has a matching server entry | Change the correct `<server><id>` rather than an unused one |
| 8 | Inspect settings overrides | Check `-s`, `-gs`, environment, wrapper, IDE, CI runner | Shell aliases, scripts, `.mvn/`, CI job config | The intended files are in use for the failing invocation | Make the invocation explicit and compare environments |
| 9 | Verify proxy and mirror routing | Confirm whether requests traverse Nexus mirror, corporate HTTP proxy, SSO gateway, or redirects | Maven `-X`; settings `<proxies>`; `curl -v` (redacted) | Maven and curl address equivalent endpoint with expected auth handling | Fix proxy configuration, redirect/auth behavior, or repository path |
| 10 | Analyze debug output safely | Correlate artifact, effective repo ID, response, and reason for rejection | `mvn -X clean install` with redacted output | Clear cause for 401 or a narrow set of possibilities | Escalate with sanitized URL/ID/status and evidence |

## 1. Capture the failing URL and HTTP status

Run:

```bash
mvn clean install -e
```

Look for the **first relevant** failure line, for example:

```text
Could not transfer artifact com.example:internal-lib:pom:1.2.3
from/to company-nexus (https://nexus.company.com/repository/maven-public/):
status code: 401, reason phrase: Unauthorized
```

Record separately:

- Artifact coordinates (`groupId:artifactId:type:version`) or plugin involved.
- Repository ID (`company-nexus`).
- Base repository URL and, when obtainable, the full artifact URL.
- HTTP status (`401`, not `403` or a TLS exception).
- Whether failure is dependency resolution, **plugin** resolution, or deployment (different repositories/IDs may apply).

**Expected:** a reproducible 401 for a specific endpoint. **Interpretation:** HTTP 401 means an HTTP component rejected or requested authentication; it does not by itself prove Nexus generated the response. If the actual error is `PKIX path building failed`, switch to check 5.

## 2. Match repository/mirror IDs to `<server><id>`

Example configuration:

```xml
<settings>
  <mirrors>
    <mirror>
      <id>company-nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.company.com/repository/maven-public/</url>
    </mirror>
  </mirrors>
  <servers>
    <server>
      <id>company-nexus</id>
      <username>TOKEN_USERNAME</username>
      <password>TOKEN_PASSWORD_OR_ENCRYPTED_VALUE</password>
    </server>
  </servers>
</settings>
```

**Rule:** Maven associates a `<server>` with a remote repository/mirror by **ID**, not by matching URL strings. When Maven chooses a mirror, the **mirror's ID** is the one that generally needs a matching server entry. Check exact spelling, including case.

**Expected:** `company-nexus` matches in both places. **If not:** fix the server ID, then retry. Avoid putting credentials in `pom.xml`.

## 3. Confirm which Maven, Java, and settings/security files are in use

```bash
mvn -version
printf 'HOME=%s\nJAVA_HOME=%s\nMAVEN_OPTS=%s\n' "$HOME" "$JAVA_HOME" "$MAVEN_OPTS"
ls -l ~/.m2/settings.xml ~/.m2/settings-security.xml 2>/dev/null
```

Maven normally merges global settings (`<MAVEN_HOME>/conf/settings.xml`) and user settings (`~/.m2/settings.xml`), with user settings taking precedence in conflicts. Maven 3's legacy encrypted-password mechanism uses `~/.m2/settings-security.xml` by default; Maven 4 has different encryption tooling and security-file conventions.

To inspect your effective configuration:

```bash
mvn help:effective-settings
```

**Caution:** Even when some secrets are masked, don't share raw output. If `help:effective-settings` itself fails because the help plugin cannot be downloaded, move to `mvn -X` diagnostics or use an already-cached plugin.

**Expected:** the Maven/Java installations and configuration locations correspond to the environment being tested. **If not:** ensure you edit the *same* settings files that the failing Maven command reads.

## 4. Test the exact artifact URL using curl

A browser page at `https://nexus.company.com/` often works through corporate SSO cookies; it does **not** verify Maven repository/token access. Instead, take the URL of a **known existing artifact** under the repository Maven uses.

Example:

```bash
ARTIFACT_URL='https://nexus.company.com/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom'

# Unauthenticated control
curl -sS -o /dev/null -w 'Anonymous HTTP: %{http_code}\n' "$ARTIFACT_URL"

# Authenticated: curl prompts for the token password, without echoing it
curl -sS -o /dev/null -w 'Authenticated HTTP: %{http_code}\n' \
  -u 'TOKEN_USERNAME' "$ARTIFACT_URL"
```

These are **real curl format strings**: `%{http_code}` is not a placeholder; `ARTIFACT_URL` and `TOKEN_USERNAME` are placeholders. For more details without displaying the body, add `-i` (to a request that does not discard output) or use `curl -v` while carefully protecting credentials and headers.

| Anonymous | Authenticated | Likely meaning |
|---|---|---|
| 401 | 200 | Credentials are required and accepted for that artifact |
| 200 | 200 | Public artifact: this does not prove credentials were checked |
| 401 | 401 | Wrong/missing token, permission, authentication realm, or intermediary |
| 403 | 403 | Authorization/policy issue likely; inspect response source |
| 404 | 404 | Wrong/nonexistent artifact path, or possible existence hiding |

**Expected:** HTTP 200 for a known-existing artifact that Maven needs. For redirects, inspect the redirect chain; don't blindly follow redirects while forwarding credentials to unrelated hosts.

## 5. Verify JVM TLS trust only when indicated

A genuine HTTP response means TLS succeeded *for that particular connection*. Curl's successful TLS connection is not conclusive for Java because curl and the JVM may trust different certificate stores.

Find Java used by Maven:

```bash
mvn -version
```

For a conventional JVM truststore, inspect:

```bash
ls -l "$JAVA_HOME/lib/security/cacerts"
"$JAVA_HOME/bin/keytool" -list -v \
  -keystore "$JAVA_HOME/lib/security/cacerts" \
  -alias company-root-ca
```

Use the **Java home printed by `mvn -version`** if it differs from `$JAVA_HOME`. Compare the certificate's SHA-256 fingerprint with the official corporate CA certificate, rather than trusting an alias alone. Check for a custom override via `-Djavax.net.ssl.trustStore=...`, `jssecacerts`, or environment/Maven JVM options.

To diagnose the truststore used during execution:

```bash
MAVEN_OPTS="$MAVEN_OPTS -Djavax.net.debug=ssl,trustmanager" \
  mvn clean install > /tmp/maven-tls.log 2>&1
```

Search the log for `trustStore` and certificate-handshake details. **Expected:** trusted CA chain and no `SSLHandshakeException` / `PKIX path building failed`. **If actual status is 401:** normally return focus to checks 2, 4, 6–10. Corporate TLS interception or mTLS can add complexity, so distinguish the precise failing connection.

## 6. Temporarily test plaintext credentials to isolate decryption

If the password in `settings.xml` is encrypted, make a **local, temporary** test using the exact credentials curl accepted:

```xml
<servers>
  <server>
    <id>company-nexus</id>
    <username>TOKEN_USERNAME</username>
    <password>ACTUAL_NEXUS_TOKEN_PASSWORD</password>
  </server>
</servers>
```

Never commit this file or share it; restrict file permissions and promptly restore the encrypted or secret-injected configuration after the test. If the password includes XML-sensitive characters, escape them (`&amp;`, `&lt;` etc.).

**Expected interpretation:**

- **Plaintext works, encrypted fails:** investigate Maven's master-password configuration, `settings-security.xml` (Maven 3), encryption format/version, and any custom settings path.
- **Both fail:** focus on which server ID and endpoint Maven selects, and whether the token is being sent correctly.
- **Both work:** the initial failure may have depended on environment, repository endpoint, stale settings, or a transient auth condition.

## 7. Verify the **effective** mirror-to-server mapping

A project may have multiple repository declarations, inherited parent POMs, plugin repositories, profiles, or an overriding mirror. What matters is the **ID selected at runtime**.

```bash
mvn help:effective-settings
mvn -X clean install
```

Look for messages resembling:

```text
Using mirror company-nexus (https://nexus.company.com/repository/maven-public/)
for central (https://repo.maven.apache.org/maven2/).
```

If Maven selects `company-nexus`, the corresponding server credentials should be under `<server><id>company-nexus</id>`, even though the original repository was `central`.

**Expected:** the actual mirror ID matches the server entry, not merely the repository you expected Maven to use. **If not:** fix the effective configuration, avoiding multiple conflicting mirror definitions.

## 8. Inspect user/global settings and invocation overrides

A developer terminal, IDE, Maven Wrapper, build script, Docker container, and GitLab runner can read **different settings files**.

Look for:

```bash
# Explicit settings arguments in aliases or scripts
alias mvn 2>/dev/null || true

# Project Maven JVM options and Maven configuration
cat .mvn/jvm.config 2>/dev/null || true
cat .mvn/maven.config 2>/dev/null || true

# Environment settings
printf 'MAVEN_OPTS=%s\n' "$MAVEN_OPTS"
```

Maven supports `-s /path/to/settings.xml` (user settings) and `-gs /path/to/global-settings.xml` (global settings). To remove ambiguity, you can test with an explicitly selected settings file:

```bash
mvn -s "$HOME/.m2/settings.xml" -X clean install
```

**Expected:** the failing Maven invocation uses the intended config. **If not:** align the command, CI job, or IDE settings, and make sure the security file/encryption arrangement is compatible with that environment.

## 9. Check proxy, redirect, and mirror routing

Maven may reach Nexus through a configured HTTP proxy, reverse proxy, or authentication gateway, while your curl test might take another route.

Review `settings.xml`:

```xml
<proxies>
  <proxy>
    <id>corp-proxy</id>
    <active>true</active>
    <protocol>https</protocol>
    <host>proxy.company.com</host>
    <port>8080</port>
    <nonProxyHosts>localhost|*.company.com</nonProxyHosts>
  </proxy>
</proxies>
```

This is **illustrative**, not a recommendation to add a proxy. Inspect your real company's proxy configuration and determine whether requests should bypass it.

Useful comparisons:

```bash
mvn -X clean install
curl -v -u 'TOKEN_USERNAME' 'https://nexus.company.com/repository/maven-public/PATH/TO/ARTIFACT.pom' -o /dev/null
```

Redact authorization headers, cookies, proxy details, and tokens before sharing output. Inspect HTTP response headers (`WWW-Authenticate`, `Location`, `Via`, `Server`) to see whether Nexus or an intermediary is responding. A redirect to another host may change authentication behavior.

**Expected:** Maven and curl reach the same protected artifact through equivalent routing. **If not:** investigate corporate proxy requirements, auth challenge handling, and URL redirects.

## 10. Correlate Maven debug output safely

Run:

```bash
mvn -X clean install > /tmp/maven-debug.log 2>&1
```

Identify, in order:

1. Repository/mirror selected.
2. Exact dependency or plugin that failed.
3. Effective server ID expected for that remote.
4. Target URL and whether a proxy/redirect is involved.
5. HTTP status and any server-specific error details.

**Never publish raw `-X` output without reviewing it**; debug logs may contain sensitive repository paths, user information, or credentials depending on plugins and transport implementation.

Use this decision table:

| Observation | Most useful next check |
|---|---|
| Curl authenticated = 200; Maven = 401 | #2, #7, #8, #6, then #9 |
| Curl anonymous = 200; curl authenticated = 200 | Find a genuinely protected existing artifact before concluding token works |
| Curl authenticated = 401 | Validate user token, realm, permissions, and exact URL with Nexus admin |
| Maven = `PKIX`/`SSLHandshakeException` | #5 (truststore / Java TLS) |
| Works in terminal; fails in IDE or CI | #3, #8, #9 |
| Dependency downloads succeed; deploy fails | Check deployment `distributionManagement` repository ID and corresponding server credentials |

## Suggested priority for this specific case

Given a genuine HTTP **401** from Maven, a successful authenticated curl on the **exact same protected artifact**, and no TLS errors, start with:

**#4 → #7 → #8 → #6 → #10**.

If the curl test only retrieved the **Nexus UI's HTML page**, do **#4 first** against an actual `.pom` or `.jar` file. The browser UI may use SSO session cookies, whereas Maven normally uses repository credentials or a Nexus user token.

## References

- [Apache Maven Settings Reference](https://maven.apache.org/settings.html)
- [Apache Maven Guide to Mirror Settings](https://maven.apache.org/guides/mini/guide-mirror-settings.html)
- [Apache Maven Password Encryption (Maven 3)](https://maven.apache.org/guides/mini/guide-encryption.html)
- [Oracle `keytool` documentation](https://docs.oracle.com/en/java/javase/17/docs/specs/man/keytool.html)
