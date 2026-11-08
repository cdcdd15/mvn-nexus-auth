# Check 7 — Verify the effective repository/mirror-to-server authentication mapping

**Problem:** Your token succeeds in curl, but Maven returns `401`. Maven sends credentials chosen by the *effective remote repository ID*, which can differ from the `<repository><id>` you saw in the POM once a mirror is applied.

## Summary

| Configuration | Role | Matching rule |
|---|---|---|
| `<repositories><repository>` | Defines an artifact repository | Has own `<id>` and `<url>` |
| `<pluginRepositories><pluginRepository>` | Defines plugin repository | Also subject to mirror selection |
| `<mirrors><mirror>` | Redirects eligible repository requests | `mirrorOf` selects original repo IDs |
| `<servers><server>` | Defines auth material | Its `<id>` must match the **selected endpoint ID** |

## 7.1 The minimal correct mirror example

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

Both IDs are `company-nexus`. The URL does not itself select the password: **Maven matches IDs, not endpoint URL strings**. `mirrorOf=*` generally directs dependency and plugin repository requests through this mirror, with certain Maven-internal exceptions/behaviors depending on configuration. This example does **not** configure deployments.

## 7.2 The common error

```xml
<mirror>
  <id>nexus-public</id>
  <mirrorOf>*</mirrorOf>
  <url>https://nexus.company.com/repository/maven-public/</url>
</mirror>
<server>
  <id>nexus-releases</id>
  <username>TOKEN_USERNAME</username>
  <password>TOKEN_PASSWORD</password>
</server>
```

Maven downloads through **`nexus-public`** but has credentials only for **`nexus-releases`**. Even if the token is valid and the URL looks correct, requests may go unauthenticated and produce HTTP 401.

## 7.3 Observe the effective mirror Maven actually selects

```bash
mvn -X clean install > /tmp/mvn-check7.log 2>&1

grep -E 'Using mirror|Downloading from|from/to|401|Unauthorized' \
  /tmp/mvn-check7.log
```

Example debug line (illustrative):

```text
Using mirror company-nexus (https://nexus.company.com/repository/maven-public/)
for central (https://repo.maven.apache.org/maven2/).
```

Key interpretation: even though the original repository is `central`, the endpoint ID for this download is **`company-nexus`**. Verify that a `<server id="company-nexus">` entry is in the effective settings. Output differs between Maven versions, and not every log line reveals the authentication lookup directly.

## 7.4 Inspect the effective settings and effective POM

```bash
mvn help:effective-settings
mvn help:effective-pom
```

The first shows the merged settings, including mirrors and server entries; the second helps identify inherited repositories and plugin repositories. Either command might fail if Maven needs to download an uncached Help plugin from the misconfigured Nexus endpoint. If so, start from `mvn -X` and static files.

Review the effective sources:

- Maven Central defaults and the super POM.
- Parent POMs and active POM profiles.
- User/global `settings.xml` profiles (`<repositories>` and `<pluginRepositories>`).
- Mirrors from effective settings and their order/matching.
- Requested operation: dependency download vs plugin download vs `mvn deploy`.

## 7.5 Understand mirrorOf patterns

| Pattern | Meaning |
|---|---|
| `*` | Match all eligible repositories |
| `central` | Match repository whose ID is `central` |
| `external:*` | External repositories (excludes localhost and file-based) |
| `external:http:*` | External HTTP repositories (Maven 3.8.0+) |
| `*,!internal-repo` | Match all except repository ID `internal-repo` |

Pattern whitespace and declaration order can matter. Maven does not aggregate several mirror URLs for one repository; it chooses one matching mirror. A `mirrorOf` pattern is a *selector*, not a Nexus user or credential selector.

## 7.6 Remember plugin versus dependency versus deployment IDs

**Artifact/plugin downloads:** repositories and pluginRepositories can be mapped via a mirror; the mirror ID selects credentials.

**Deployment:** `mvn deploy` typically uses the `<distributionManagement>` repository ID (or an explicitly selected alternate deployment repository). Credentials for a group mirror used for downloading do not automatically authorize a separate hosted deployment URL.

Example deployment POM:

```xml
<distributionManagement>
  <repository>
    <id>nexus-releases</id>
    <url>https://nexus.company.com/repository/maven-releases/</url>
  </repository>
</distributionManagement>
```

That upload generally needs:

```xml
<server>
  <id>nexus-releases</id>
  <username>TOKEN_USERNAME</username>
  <password>TOKEN_PASSWORD</password>
</server>
```

## 7.7 Diagnostic matrix

| Observation | Meaning | Action |
|---|---|---|
| `Using mirror A for central`, only `<server id=B>` | Wrong auth lookup | Add/correct server ID `A` |
| Correct ID, no username/token | Incomplete server entry | Supply verified Nexus user token |
| Correct ID, encrypted credential | Might have decryption issue | Check 3 and the plaintext isolation test (#6) |
| Curl tested URL B, Maven downloads URL A | Not equivalent tests | Curl exact Maven artifact URL (#4) |
| Different ID for plugin repository | Plugin download is failing | Inspect plugin repository and mirror rules |
| All match, Maven still 401 | Proxy/realm/redirect possibility | Check 9 and server-side auth logs |

**References:** [Maven Mirror Guide](https://maven.apache.org/guides/mini/guide-mirror-settings.html), [Maven Settings Reference](https://maven.apache.org/settings.html).
