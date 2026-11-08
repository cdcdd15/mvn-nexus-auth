# 03 — Curl succeeds with credentials, Maven still fails

**Original question:** Curl authenticates to a Nexus URL using a plaintext username/password and returns an HTML page.

## What curl success proves — and does not prove

If curl returns `200 OK` for the **same protected artifact URL** as Maven, with the same token credentials:

- Credentials work **for that resource**.
- Network route and HTTP service are available to curl.
- TLS validation worked for curl **if `-k/--insecure` was not used**.

But an HTML UI/login page is **not** a reliable artifact-authentication test. A `200` from a public page doesn't demonstrate permission to fetch a Maven artifact. Curl and Java/Maven may also trust different CA stores.

## Better test: actual artifact URL

Use the *actual failing `.pom` or `.jar` URL* reported by Maven:

```bash
curl -i -u 'TOKEN_USERNAME'   'https://nexus.company.com/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom'
```

The `-u 'TOKEN_USERNAME'` form prompts for the token password. Replace repository/coordinates with the ones in the failing Maven request.

## Maven-specific checks

1. Temporarily use the **correct plaintext token password** in `settings.xml` to isolate legacy decryption mistakes (restore secure configuration afterward).
2. Check effective mirror/repository ID against `<servers><server><id>`.
3. Confirm which settings files are loaded by Maven, including any `-s`/`-gs` overrides.
4. Check whether a custom mirror/proxy redirects Maven to a different endpoint.
5. Check `mvn -X` output for the actual artifact URL; never post sensitive log contents unredacted.

Plaintext credentials in a temporary local test are a troubleshooting technique, not a recommended permanent enterprise setup.
