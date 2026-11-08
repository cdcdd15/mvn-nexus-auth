# 06 — Verify corporate certificate in the JVM Maven uses

**Original question:** How can I verify the company's certificate is installed in the actual JVM truststore used by Maven?

## Step 1 — Identify Maven's JVM

```bash
mvn -version
```

Note **Java home**. It may not be the same JVM as `java -version`, especially when multiple JDKs are installed.

On macOS/Linux:

```bash
echo "$JAVA_HOME"
which java
which mvn
```

The commands below assume `$JAVA_HOME` is the same directory as Maven's reported **Java home**; if not, use that reported directory explicitly.

## Step 2 — Locate potential truststores

Common location:

```text
<JAVA_HOME>/lib/security/cacerts
```

```bash
ls -l "$JAVA_HOME/lib/security/cacerts"
realpath "$JAVA_HOME/lib/security/cacerts"
```

The file may be a symlink. Java's truststore selection also considers an explicit `-Djavax.net.ssl.trustStore`, and (by default) `jssecacerts` ahead of `cacerts`.

## Step 3 — Inspect the certificate

List all certificates (default password is commonly `changeit`):

```bash
"$JAVA_HOME/bin/keytool" -list -v   -keystore "$JAVA_HOME/lib/security/cacerts"   -storepass changeit
```

By alias:

```bash
"$JAVA_HOME/bin/keytool" -list -v   -keystore "$JAVA_HOME/lib/security/cacerts"   -storepass changeit   -alias company-root-ca
```

Compare the SHA-256 fingerprint with the *IT-approved certificate*:

```bash
"$JAVA_HOME/bin/keytool" -printcert -file company-ca.crt
```

Matching the fingerprint is more reliable than matching a human-readable alias. A trusted **CA/root or intermediate**, not necessarily the server leaf certificate, is usually needed to build the chain.

## Step 4 — Look for an overridden truststore

```bash
echo "$MAVEN_OPTS"
cat .mvn/jvm.config  # if it exists
```

Check command-line options, `MAVEN_OPTS`, project `.mvn/jvm.config`, toolchains/CI launch options for `javax.net.ssl.trustStore` and truststore passwords. A custom store can override the default.

For a runtime check:

```bash
MAVEN_OPTS="$MAVEN_OPTS -Djavax.net.debug=ssl,trustmanager"   mvn clean install > /tmp/maven-tls.log 2>&1

grep -Ei 'trustStore is|trustStore type|Reloaded.*trust certs' /tmp/maven-tls.log
```

This is verbose and may expose details. Review/redact before sharing.

## Step 5 — If missing, coordinate an import

After confirming the correct trusted CA with your IT team:

```bash
sudo "$JAVA_HOME/bin/keytool" -importcert   -trustcacerts   -alias company-root-ca   -file company-ca.crt   -keystore "$JAVA_HOME/lib/security/cacerts"
```

Prefer organization-approved JDK truststore management or a dedicated corporate truststore where provided. JDK upgrades can replace `cacerts`.

**Relevance to the 401:** a verified HTTP 401 is usually *after* TLS, so truststore verification is useful but typically not the fix for that particular error.
