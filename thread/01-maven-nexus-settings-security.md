# 01 — Maven/Nexus authentication: `settings.xml` and `settings-security.xml`

**Original question:** Explain how Maven/Nexus authentication works with `settings.xml` and `settings-se.xml` (interpreted as `settings-security.xml`).

## Responsibilities

- **`~/.m2/settings.xml`** provides repositories/mirrors, server IDs and credentials.
- **`~/.m2/settings-security.xml`** (Maven 3 legacy encryption) stores an encrypted master-password value used to decrypt encrypted passwords in `settings.xml`.
- **Nexus** checks the credential sent by Maven. Nexus does not decrypt the Maven-format encrypted password.

```xml
<!-- ~/.m2/settings.xml -->
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
      <password>{MAVEN_ENCRYPTED_TOKEN_PASSWORD}</password>
    </server>
  </servers>
</settings>
```

```xml
<!-- ~/.m2/settings-security.xml (Maven 3) -->
<settingsSecurity>
  <master>{MAVEN_ENCRYPTED_MASTER_PASSWORD}</master>
</settingsSecurity>
```

The `{...}` strings are **placeholders**, not valid encrypted passwords. Server ID matching is central: Maven associates credentials with a repository/mirror by **ID**, not URL.

## Maven 3 legacy encryption setup

```bash
mvn --encrypt-master-password
# Put returned value in settings-security.xml
mvn --encrypt-password
# Put returned value in settings.xml's <password>
```

The legacy encryption mechanism does not provide strong security against someone who can read both files. Prefer centrally managed secrets/tokens in CI. Maven 4 uses a newer encryption mechanism (`settings-security4.xml` and `mvnenc`); verify Maven version before using Maven 3 commands.

## Download vs upload

- `mvn clean install` installs the built project to the **local** Maven repository; it may **download** dependencies/plugins from Nexus.
- `mvn deploy` publishes the built artifact to a **remote** deployment repository when configured; its deployment repository ID needs matching `<server>` credentials.

**Security:** Never commit real credentials or a populated `settings-security.xml` to Git.
