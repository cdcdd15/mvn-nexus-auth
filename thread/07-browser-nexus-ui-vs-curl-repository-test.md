# 07 — Nexus UI vs curl: which URL should be tested?

**Original question:** Browser can access Nexus UI without credentials, while generated credentials came from Nexus UI/account. Which URL matters for curl?

## Different authentication contexts

A web browser may access the Nexus UI via company SSO or an existing session cookie. Maven/curl may authenticate to repository endpoints using **Nexus user-token username/password**, which can differ from normal SSO credentials.

| URL type | Purpose | Validates Maven-style access? |
|---|---|---|
| `https://nexus.company.com/` | Nexus UI | No, not by itself |
| `https://nexus.company.com/#browse` | UI navigation | No |
| `https://nexus.company.com/repository/maven-public/` | Repository base URL | Somewhat; may yield directory page or error |
| `https://nexus.company.com/repository/maven-public/<group-path>/<artifact>/<version>/<file>.pom` | Exact artifact request | **Yes — best test** |

The `maven-public` name and all example coordinates are placeholders; use the **actual Maven error URL**.

## Recommended curl test

```bash
curl -i -u 'TOKEN_USERNAME'   'https://nexus.company.com/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom'
```

Curl prompts for the token password. Look at **HTTP status** and ensure the body is a `.pom` XML file or the expected artifact, rather than a login page.

Compare with unauthenticated access:

```bash
curl -sS -o /dev/null -w '%{http_code}
'   'https://nexus.company.com/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom'
```

| No auth | With token | Interpretation |
|---|---|---|
| 401 | 200 | Token is accepted and needed for that artifact |
| 200 | 200 | Artifact may be anonymous; token validity unproven |
| 401 | 401 | Token/configuration/permissions problem |
| 404 | 404 | Path/artifact may be wrong; auth still inconclusive |

Redirects and proxy/SSO pages can also confuse testing. Inspect `-i` response headers and content type; compare the **same URL** as Maven.

## Mapping result back to Maven

If the exact protected artifact URL succeeds through curl but fails with Maven 401, focus on token values in `<server>`, `<server><id>` matching mirror ID, settings file selection and password decryption.
