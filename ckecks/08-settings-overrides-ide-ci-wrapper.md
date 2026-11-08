# Check 8 — Detect settings overrides in terminal, IDE, Maven Wrapper and CI

**Goal:** Verify that the *exact invocation producing HTTP 401* reads the expected settings files and uses the expected JVM, user home, and credentials. The same repository may behave differently across terminal, IntelliJ, an IDE plugin, Docker, GitLab CI, or a shell script.

## Quick comparison

| Context | Common source of difference | What to check |
|---|---|---|
| Terminal `mvn` | Shell alias, Maven installation, `$HOME` | `command -v`, `alias`, `mvn -version` |
| `./mvnw` | Wrapper Maven version/distribution | `./mvnw -version`, `.mvn/` |
| IDE | Bundled Maven/JDK, user settings path | IDE Maven settings panel/run logs |
| GitLab Runner | Different OS user/container home, injected settings | Job `mvn -version`, job `-s`, protected CI variables |
| Docker | Files not mounted into container | Inside-container `$HOME`, `/root/.m2` |

## 8.1 Inventory the terminal invocation

```bash
command -v mvn
mvn -version
alias mvn 2>/dev/null || true
printf 'HOME=%s\nJAVA_HOME=%s\nMAVEN_OPTS=%s\n' \
  "$HOME" "$JAVA_HOME" "$MAVEN_OPTS"
```

`command -v` identifies shell resolution; if `mvn` is an alias/function, inspect it. Run `mvn -X ...` for the failing command and look for `Reading user settings from ...` and `Reading global settings from ...` where present.

## 8.2 Check project-level `.mvn/` files

```bash
ls -la .mvn 2>/dev/null || true
cat .mvn/maven.config 2>/dev/null || true
cat .mvn/jvm.config 2>/dev/null || true
```

- `.mvn/maven.config` may supply Maven CLI options, including a settings file.
- `.mvn/jvm.config` can supply JVM options, such as a custom TLS truststore.
- `.mvn/wrapper/maven-wrapper.properties` selects a wrapper-managed Maven distribution.

Check a project Maven wrapper separately:

```bash
./mvnw -version
./mvnw -X clean install
```

The wrapper can run a different Maven version from `mvn`, although it commonly uses the same user settings by default.

## 8.3 User vs global settings precedence

Default settings locations are:

```text
${maven.home}/conf/settings.xml       # global
${user.home}/.m2/settings.xml         # user
```

Maven merges settings; user settings generally take precedence for corresponding elements. Explicit flags override file locations:

```bash
mvn -s '/path/to/user-settings.xml' \
    -gs '/path/to/global-settings.xml' \
    -X clean install
```

Use your real file paths. Never assume `~/.m2/settings.xml` is being read if Maven is invoked by CI or an IDE under another account.

## 8.4 Force a reproducible settings test

From the same project directory as the failing command:

```bash
mvn -s "$HOME/.m2/settings.xml" -e clean install
```

If this succeeds and bare `mvn clean install` fails, investigate injected CLI flags, `.mvn/maven.config`, shell wrappers, or process environment. If both fail, examine effective server/mirror mapping (#7) and auth/token format (#6).

## 8.5 IntelliJ IDEA and other IDEs

Inspect the IDE's Maven configuration for: **Maven home path**, **user settings file**, **local repository**, **JDK for importer/runner**, and any command-line runner VM options. Compare the IDE's actual Maven command with the terminal command. Some IDE settings override user settings and can use different JDK installations. Prefer testing in a terminal first, then reproducing in the IDE.

## 8.6 GitLab CI / enterprise runners

The runner can have a different home directory, Maven installation, network proxy, and injected settings. A common supported pattern is to create an ephemeral settings file from CI secrets and pass it via `-s`. Never commit a live secret or echo it into logs.

Example conceptual invocation:

```bash
mvn -B -s "$CI_PROJECT_DIR/.ci/settings.xml" clean verify
```

Only use this if the file is securely created by the pipeline and unavailable to untrusted jobs. Inspect GitLab variable protection, masking, scope, and whether the job runs on a protected branch; these details can explain why variables exist locally but not in CI. Build output must not expose secrets.

If using a container, inspect *inside the container* rather than on the host:

```bash
mvn -version
printf '%s\n' "$HOME"
ls -l "$HOME/.m2/settings.xml" 2>/dev/null
```

## 8.7 Diagnostic outcomes

| Result | Likely explanation | Next step |
|---|---|---|
| Terminal works, IDE fails | Different settings/JDK/command | Align IDE runtime and settings |
| Host works, Docker fails | Container lacks settings/credentials/CA | Inject approved settings/trust config |
| `mvn` works, `./mvnw` fails | Wrapper version/options mismatch | Compare versions and debug logs |
| Local works, CI returns 401 | CI token absent, scope restrictions, wrong ID | Check CI secret injection and #7 |
| `-s` fixes issue | Wrong default user settings or override | Fix invocation or profile |

## Safety

Do not include real tokens in CI YAML, committed settings files, shell history, bug reports, or terminal screenshots. If debugging needs credentials, use temporary files with restricted permissions and remove them after use.

**References:** [Maven Configuration](https://maven.apache.org/configure.html), [Maven Settings Reference](https://maven.apache.org/settings.html).
