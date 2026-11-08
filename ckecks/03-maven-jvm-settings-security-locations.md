# Check 3 — Confirm Maven, JVM, settings.xml and settings-security.xml locations

**Scenario:** `curl` can download a Nexus artifact, but `mvn clean install` receives HTTP 401. Before changing credentials, establish which Maven installation and configuration files the *failing command* is really using.

## Scope and success criterion

| Question | How to verify | Expected result |
|---|---|---|
| Which Maven runs? | `mvn -version` | Expected Maven version and Maven home |
| Which JVM runs Maven? | `mvn -version` | Expected `Java home` |
| Which user owns the Maven run? | `whoami`, `echo "$HOME"` | Intended account/home directory |
| Which user/global settings are loaded? | `mvn -X ...`, explicit `-s` / `-gs` | Intended files |
| Which encryption mechanism applies? | Maven version and encryption config | Compatible master-password configuration |
| Does the expected server entry exist? | Inspect user/global `settings.xml` | Matching ID and intended token username |

## 3.1 Identify the actual Maven executable and Java

```bash
mvn -version
command -v mvn
printf 'HOME=%s\nJAVA_HOME=%s\n' "$HOME" "$JAVA_HOME"
```

Typical output:

```text
Apache Maven 3.9.x
Maven home: /opt/apache-maven-3.9.x
Java version: 17.x
Java home: /opt/jdk-17
```

Record **Maven home**, **Java home**, and the operating-system account. `java -version` by itself is not sufficient: `mvn` may select a different JVM. On Windows, use `where mvn`, `echo %USERPROFILE%`, and `mvn -version` in Command Prompt; use `$env:USERPROFILE` in PowerShell.

## 3.2 Understand Maven's two settings layers

By default Maven combines:

- **Global settings:** `${maven.home}/conf/settings.xml` (Maven installation).
- **User settings:** `${user.home}/.m2/settings.xml` (for the OS user running Maven).

User settings override or augment corresponding global settings according to Maven's merge rules. In practice, a company mirror might be in global settings while the secret `<server>` entry is in user settings. Do not assume the literal `$HOME` value matches Java's `user.home` in every environment.

```bash
ls -l "$HOME/.m2/settings.xml" \
      "$HOME/.m2/settings-security.xml" 2>/dev/null
```

A missing file is not automatically an error: repositories may be public, credentials may come from alternate settings, and newer Maven versions use different encryption mechanisms.

## 3.3 See what the invocation loads

```bash
mvn -X -e clean install > /tmp/mvn-check3.log 2>&1
```

Search the log locally for settings-related lines:

```bash
grep -Ei 'reading (global|user) settings|settings.xml|maven home|java home' \
  /tmp/mvn-check3.log
```

Typical debug messages can include `Reading global settings from ...` and `Reading user settings from ...`. Wording varies across Maven versions. Do not publish unredacted debug logs.

If the Help plugin is already usable, inspect effective settings:

```bash
mvn help:effective-settings
```

This combines settings but may require downloading the Help plugin—the same Nexus problem can prevent the command from running. Sensitive data can appear in logs or outputs, even if the plugin masks some passwords.

## 3.4 Maven 3 encryption: where settings-security.xml belongs

The legacy Maven 3 encrypted-password configuration normally lives at:

```text
~/.m2/settings-security.xml
```

Example structure (illustrative values):

```xml
<settingsSecurity>
  <master>{ENCRYPTED_MASTER_VALUE}</master>
</settingsSecurity>
```

The file contains an encoded master secret that Maven uses to decrypt an encrypted `<server><password>` in `settings.xml`. Maven does *not* send the master password to Nexus. The master-password mechanism is not strong protection against an attacker who can read both files.

A Maven 3 security file may also use relocation to point to another master-password file. If it does, verify the referenced file is accessible from the current account. Maven 4 has a newer credential encryption approach and uses different tooling/file conventions (`mvnenc`, `settings-security4.xml`); confirm the Maven major version before following Maven 3 instructions.

## 3.5 Explicitly select the settings file to eliminate ambiguity

```bash
mvn -s "$HOME/.m2/settings.xml" -e clean install
```

For a known global settings path:

```bash
mvn -s "$HOME/.m2/settings.xml" \
    -gs '/opt/apache-maven/conf/settings.xml' \
    -e clean install
```

`-s` chooses the user-settings file; `-gs` chooses global settings. It does **not** by itself select or relocate the Maven 3 `settings-security.xml` master file. If an IDE or CI runner uses a separate home directory, settings selection and decryption must both be valid in that process.

## 3.6 Interpret the findings

| Observation | Meaning | Next step |
|---|---|---|
| `mvn -version` shows unexpected Java home | Wrong JVM could affect TLS or credential tooling | Follow check 5; fix runtime choice |
| Debug log reads an unexpected `settings.xml` | Maven might lack the expected `<server>` | Follow check 8 |
| Expected settings are loaded, server ID absent | Auth entry is unavailable | Follow check 7 |
| Encrypted password + wrong/missing master file | Decryption may fail | Follow check 6 from main guide |
| Everything matches, HTTP 401 persists | Need endpoint/routing comparison | Follow checks 7 and 9 |

## Safety and notes

Do not commit actual `settings.xml` containing secrets or `settings-security.xml`. Check file permissions (e.g., `chmod 600 ~/.m2/settings.xml ~/.m2/settings-security.xml` on a private Unix account, where appropriate). Redact hostnames and usernames when sharing logs if company policy requires it.

**References:** [Maven Settings](https://maven.apache.org/settings.html), [Maven 3 password encryption](https://maven.apache.org/guides/mini/guide-encryption.html).
