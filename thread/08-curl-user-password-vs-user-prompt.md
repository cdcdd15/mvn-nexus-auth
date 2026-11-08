# 08 — Why the two curl `-u` examples differ

**Original question:** Why does one example specify `TOKEN_USERNAME:TOKEN_PASSWORD`, while another only specifies `TOKEN_USERNAME`?

Both use curl's `-u` / `--user` option for HTTP authentication, ordinarily Basic authentication for a Nexus repository.

## Example A — username and password inline

```bash
curl -i -u 'TOKEN_USERNAME:TOKEN_PASSWORD'   'https://nexus.company.com/repository/maven-public/...'
```

The colon separates username and password. This can expose the password in shell history and process information. The URL ending in `...` is **illustrative**, not a runnable artifact URL.

## Example B — username only, password prompted

```bash
curl -i -u 'TOKEN_USERNAME'   'https://nexus.company.com/repository/maven-public/...'
```

Curl prompts interactively for a password, usually displaying something like:

```text
Enter host password for user 'TOKEN_USERNAME':
```

This avoids putting the token password in shell history/command arguments. With identical credentials, the resulting HTTP authentication is equivalent.

## Preferred real test

Replace the demonstration URL with the **exact artifact URL from the Maven 401**, for example:

```bash
curl -i -u 'TOKEN_USERNAME'   'https://nexus.company.com/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom'
```

Do not share your token password or paste it into project documentation.
