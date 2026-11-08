# Check 5 — Verify the corporate CA in the JVM truststore Maven actually uses

**Goal:** Prove that Maven's Java runtime can establish a trusted TLS connection to the Nexus HTTPS endpoint. This is mainly relevant for `PKIX path building failed` / `SSLHandshakeException`, **not** usually for a genuine HTTP 401.

## Quick decision table

| Evidence | What it means | Where to focus |
|---|---|---|
| Maven gets HTTP `401` from Nexus URL | TLS succeeded for that HTTP exchange | Auth, mirrors and proxies (checks 7–9) |
| Maven shows `PKIX path building failed` | JVM could not build a trusted certificate path | Truststore / CA chain |
| Curl works, Maven fails with PKIX | Curl and Java probably use different trust material | JVM truststore |
| TLS error only on one hostname | Different TLS termination/chain or proxy | Actual contacted host, SNI, corporate interception |

## 5.1 Identify Java used by Maven (not merely shell Java)

```bash
mvn -version
```

Record the **Java home** line. For example:

```text
Java home: /opt/jdk-17
```

Set an explicit variable based on that output (replace the example):

```bash
MVN_JAVA_HOME='/opt/jdk-17'
```

## 5.2 Determine the truststore selection

For standard Java JSSE trust management, the usual order is:

1. `-Djavax.net.ssl.trustStore=/path/to/custom-store` when configured.
2. `${java.home}/lib/security/jssecacerts` if it exists.
3. `${java.home}/lib/security/cacerts` otherwise.

An explicit but nonexistent truststore path can lead to an empty or unusable trust configuration; it is **not** safe to assume it falls back to `cacerts`. A JVM may also use a PKCS#11-backed store, OS-integrated provider, or custom trust manager. Corporate tools/IDEs can inject JVM properties.

Check the normal locations:

```bash
ls -l "$MVN_JAVA_HOME/lib/security/jssecacerts" \
      "$MVN_JAVA_HOME/lib/security/cacerts" 2>/dev/null
```

Check common Maven JVM option sources:

```bash
printf 'MAVEN_OPTS=%s\nJAVA_TOOL_OPTIONS=%s\nJDK_JAVA_OPTIONS=%s\n' \
  "$MAVEN_OPTS" "$JAVA_TOOL_OPTIONS" "$JDK_JAVA_OPTIONS"
cat .mvn/jvm.config 2>/dev/null
```

Search for `javax.net.ssl.trustStore`, `trustStoreType`, `trustStorePassword`, and any corporate Java-agent settings. **Never paste secrets from environment output into tickets.**

## 5.3 Inspect the CA certificate using keytool

For default `cacerts`:

```bash
"$MVN_JAVA_HOME/bin/keytool" -list -v \
  -keystore "$MVN_JAVA_HOME/lib/security/cacerts" \
  -alias company-root-ca
```

The alias `company-root-ca` is a placeholder: use the actual alias. If the alias is unknown, list entries and search for the corporate issuer/company name:

```bash
"$MVN_JAVA_HOME/bin/keytool" -list -v \
  -keystore "$MVN_JAVA_HOME/lib/security/cacerts"
```

The common `cacerts` store password is `changeit`, but it may differ; `keytool` may prompt. You need not know a server's *leaf certificate* alias: trust usually comes from a trusted CA root (sometimes an intermediate), not a leaf certificate explicitly stored in `cacerts`.

## 5.4 Validate the *exact* CA certificate by SHA-256 fingerprint

Given a verified `company-ca.crt` from IT:

```bash
"$MVN_JAVA_HOME/bin/keytool" -printcert -file company-ca.crt
```

Compare the `SHA256` fingerprint with the entry in the actual truststore. A matching alias *alone* is insufficient: entries can be overwritten or differ between JDK installations.

For macOS JDKs, `cacerts` may be a symlink. Resolve the real file:

```bash
realpath "$MVN_JAVA_HOME/lib/security/cacerts"
```

## 5.5 Inspect what Java actually loads during Maven execution

```bash
MAVEN_OPTS="$MAVEN_OPTS -Djavax.net.debug=ssl,trustmanager" \
  mvn -e clean install > /tmp/mvn-tls-check5.log 2>&1

grep -Ei 'trustStore is|trustStore type|trustmanager|PKIX|handshake' \
  /tmp/mvn-tls-check5.log | head -80
```

Debug wording depends on JDK version; the log may be large and sensitive. The debug setting traces what the Java TLS stack does; if a plugin uses a custom HTTP/TLS client, its logging behavior may differ.

## 5.6 Compare JVM TLS and curl carefully

Curl may trust the OS certificate store or a curl-specific CA bundle. `curl -k` disables validation and therefore is **not** evidence of a trusted certificate. A successful `curl` also does not tell you which JVM/Java truststore Maven uses.

An HTTP `401` on Maven's **actual artifact endpoint** normally demonstrates successful TLS *on that connection*. It does not guarantee a separate deployment endpoint, proxy, redirect target, or plugin repository uses the same certificate chain.

## 5.7 Only if missing: coordinate a trusted CA import

Prefer your company's approved JDK/truststore deployment procedure. After verifying the fingerprint with IT and backing up the relevant truststore, an authorized administrator might use:

```bash
sudo "$MVN_JAVA_HOME/bin/keytool" -importcert \
  -trustcacerts \
  -alias company-root-ca \
  -file company-ca.crt \
  -keystore "$MVN_JAVA_HOME/lib/security/cacerts"
```

Do **not** import an unverified certificate from a failed TLS connection. Updating the JDK may reset `cacerts`; a managed separate truststore is often preferable. Never solve production TLS failures by disabling validation.

## Interpretation and next action

- **CA missing / PKIX failure:** coordinate truststore correction and retry Maven.
- **Correct CA present / TLS still fails:** inspect intermediate chain, proxy interception, hostname verification, wrong truststore, and actual hostname.
- **Maven HTTP 401 remains:** return to checks 7–9; truststore changes ordinarily won't repair HTTP Basic authentication.

**References:** [Oracle keytool](https://docs.oracle.com/en/java/javase/17/docs/specs/man/keytool.html), [JSSE Reference Guide](https://docs.oracle.com/en/java/javase/17/security/java-secure-socket-extension-jsse-reference-guide.html).
