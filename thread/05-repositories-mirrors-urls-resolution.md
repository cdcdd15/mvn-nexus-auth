# 05 — How `settings.xml` controls Maven resolution and authentication

**Original question:** How does `settings.xml` work for artifact resolution/authentication? Why are there mirrors and URLs?

## Three concepts

- **Repository:** source of artifacts (a URL plus ID) from which Maven may resolve dependencies and plugins.
- **Mirror:** directs requests originally targeting selected repositories toward another endpoint.
- **Server:** authentication/connection configuration selected by repository or mirror **ID**; it does not define the repository URL.

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
      <password>TOKEN_PASSWORD</password>
    </server>
  </servers>
</settings>
```

## Why a Nexus mirror?

Normally Maven can resolve public artifacts from Maven Central. An enterprise often redirects matching repository requests through a Nexus **group** repository such as `maven-public`, which can combine internal hosted repositories and proxies to external upstream repositories. It centralizes caching, permissions and policy controls.

`<mirrorOf>*</mirrorOf>` requests that Maven mirror all eligible remote artifact repositories. Exact behavior can be refined with mirror patterns (including exclusions). It does not make the `settings.xml` URL appear magically from the POM: Maven first identifies a source repository, then applies matching mirror rules.

## Effective resolution flow

1. Maven evaluates dependency/plugin coordinates and available local artifacts.
2. It determines configured remote repositories from effective POM and active settings profiles.
3. It selects a matching mirror (if any).
4. It reads the URL from the selected repository/mirror.
5. It selects credentials using that selected endpoint's **ID**.
6. It establishes HTTPS, makes the artifact request and caches a successful response locally.

## Repository profile example (adds a repository; not the same as a mirror)

```xml
<profiles>
  <profile>
    <id>corporate</id>
    <repositories>
      <repository>
        <id>nexus-releases</id>
        <url>https://nexus.company.com/repository/maven-releases/</url>
      </repository>
    </repositories>
  </profile>
</profiles>
<activeProfiles><activeProfile>corporate</activeProfile></activeProfiles>
```

Unlike a mirror, this declares an additional remote repository. Maven plugins may also need `<pluginRepositories>` configuration.

## Common 401 misconfiguration

If mirror ID is `nexus-public` but server ID is `nexus-releases`, Maven may request artifacts through `nexus-public` without finding the intended credential. Matching **URLs** is insufficient; matching **IDs** matters.

```bash
mvn help:effective-settings
mvn -X clean install
```

The `help:effective-settings` goal may itself need to resolve the Maven Help Plugin. Be careful sharing output.
