# Check 9 — Detailed diagnosis of Maven mirrors, HTTP proxies, reverse proxies, and redirects

## Goal
Prove that `curl` and Maven actually reach the **same protected artifact** through an equivalent network and authentication path. A successful curl request to the Nexus UI, a repository directory, or a different URL does not reproduce the request that returns Maven's HTTP 401.

## Quick differentiation

| Mechanism | Lives where? | What changes? | Typical evidence |
|---|---|---|---|
| Maven mirror | `settings.xml` | Logical artifact repository destination and ID | `Using mirror ... for ...` |
| HTTP(S) outbound proxy | Maven settings / JVM / environment-specific tooling | Network hop leaving developer machine | Proxy config, `CONNECT`, `407` |
| Nexus reverse proxy / gateway | Server/network infrastructure | Entry point, TLS termination, routing, authentication | `Server`, `Via`, auth headers, gateway logs |
| HTTP redirect | HTTP response | URL / host for next request | 301/302/307/308 + `Location` |
| Local Maven cache | `~/.m2/repository` | Whether remote request occurs | No transfer line / cached artifact |

## 9.1 Capture the exact failing Maven request

```bash
mvn -X -e clean install > /tmp/maven-routing.log 2>&1

grep -Ein 'Using mirror|Downloading from|Could not transfer|from/to|401|Unauthorized|407|redirect' \
  /tmp/maven-routing.log
```

Record these facts: failing artifact coordinates, `.pom`/`.jar` URL, repository ID, hostname, scheme, status code, and whether Maven was downloading a plugin or a normal dependency.

Example:

```text
Downloading from company-nexus: https://nexus.company.com/repository/maven-public/org/example/lib/1.0/lib-1.0.pom
... 401 Unauthorized
```

*Don't copy sample paths literally.* Use the **exact** failing URL from your own output.

## 9.2 Set a reproducible artifact URL

```bash
ARTIFACT_URL='https://nexus.company.com/repository/maven-public/org/example/lib/1.0/lib-1.0.pom'
```

The value above is an example placeholder. `%{http_code}`, `%{content_type}`, `%{redirect_url}` below are **literal curl variables**.

## 9.3 Test anonymously and with your issued Nexus token

```bash
curl -sS -o /dev/null \
  -w 'anonymous=%{http_code} type=%{content_type} redirect=%{redirect_url}\n' \
  "$ARTIFACT_URL"

curl -sS -o /dev/null \
  -w 'authenticated=%{http_code} type=%{content_type} redirect=%{redirect_url}\n' \
  -u 'TOKEN_USERNAME' "$ARTIFACT_URL"
```

`curl -u 'TOKEN_USERNAME'` prompts interactively for the password. Do not paste your token password into shared shell commands or logs. `200` on an **authenticated, protected, correct** `.pom` is useful evidence. `200` on an SSO HTML login page is not.

| Anonymous | Authenticated | Interpretation |
|---|---|---|
| 401 | 200 with actual POM/JAR | Token works for this artifact path |
| 200 | 200 | Resource may be readable anonymously; token not proven |
| 401 | 401 | Wrong token, wrong target/policy, or auth gateway issue |
| 403 | 403 | Permission/access policy likely, subject to server behavior |
| 3xx | 3xx | Inspect `Location` and authentication origin |
| 404 | 404 | Artifact/URL missing or access-obscured; choose known existing artifact |

Note that `curl` generally reports the **last** response code only if redirects are followed; the commands above intentionally **do not** follow redirects.

## 9.4 Inspect headers and redirects carefully

```bash
curl -sS -D - -o /dev/null "$ARTIFACT_URL"
```

Look at `WWW-Authenticate`, `Location`, `Server`, `Via`, and status. To inspect an authenticated response, avoid posting the raw output anywhere; even header dumps can leak metadata or cookies. If the server issues a redirect to another hostname, Maven and curl may handle credential forwarding differently.

Do not use `--location-trusted` for troubleshooting without explicit authorization: it can forward sensitive credentials to a redirected host.

To inspect response content without dumping it publicly:

```bash
curl -sS -u 'TOKEN_USERNAME' -o /tmp/nexus-test.pom "$ARTIFACT_URL"
file /tmp/nexus-test.pom
head -c 120 /tmp/nexus-test.pom
```

`file` and `head` are illustrative shell tools. Only use `head` if the retrieved content is safe to display locally. A `.pom` should be XML, not HTML for corporate SSO. A `.jar` is binary and should not be printed using `head`.

## 9.5 Verify Maven mirror matching

```xml
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
```

The `<server><id>` should match the effective mirror's ID. If the mirror's `mirrorOf` does not match an original repository, Maven may contact a different URL. Check 7.3 and 7.4 for exact commands.

## 9.6 Inspect outbound proxy behavior

Example of Maven settings (illustrative; do not add a proxy unless your IT configuration requires one):

```xml
<proxies>
  <proxy>
    <id>corp-http-proxy</id>
    <active>true</active>
    <protocol>http</protocol>
    <host>proxy.company.com</host>
    <port>8080</port>
    <nonProxyHosts>localhost|127.0.0.1|*.company.com</nonProxyHosts>
  </proxy>
</proxies>
```

For HTTP proxy authentication, Maven proxy username/password belong in the `<proxy>` entry (not in `<servers>`). Some corporate gateways return HTTP 401 rather than the more typical HTTP 407 for proxy-auth failures; identify the responding component from trace evidence.

Inspect curl environment without printing proxy secrets:

```bash
printf 'HTTPS_PROXY set: %s\n' "${HTTPS_PROXY:+yes}"
printf 'HTTP_PROXY set: %s\n' "${HTTP_PROXY:+yes}"
printf 'NO_PROXY set: %s\n' "${NO_PROXY:+yes}"
```

**Difference:** curl may honor `HTTPS_PROXY`/`NO_PROXY` while Maven uses `settings.xml` proxies and/or transport/JVM configurations. Two successful TLS connections can reach Nexus through **different gateways** and present different credentials or network identities.

## 9.7 Understand where HTTP 401 originates

```text
Maven or curl
   ↓ outbound corporate proxy (optional)
   ↓ corporate gateway / ingress / reverse proxy (optional)
   ↓ Nexus Repository Manager
   ↓ selected group/hosted/proxy repository
```

The message `401 Unauthorized` does **not** by itself identify which HTTP component produced it. Look for server/gateway response headers, then correlate a precise timestamp and URL with Nexus and gateway access logs if you have authorized access.

### 401 vs 407 vs TLS
- **401:** origin server or gateway challenges/denies the request; look for `WWW-Authenticate`.
- **407:** proxy-specific authentication required; look for `Proxy-Authenticate`.
- **TLS/PKIX exception:** Java has not successfully completed TLS verification to the selected endpoint; inspect truststores and the actual TLS-terminating host.

## 9.8 Compare routes without leaking secrets

Use verbose curl only in a private local environment:

```bash
curl -v -u 'TOKEN_USERNAME' "$ARTIFACT_URL" -o /dev/null
```

It can show proxy routing, CONNECT, TLS and redirects, but may disclose `Authorization` or other sensitive data. **Never send unredacted verbose output** to chat or an issue tracker. Do not bypass TLS verification with `-k`: it would invalidate the certificate-trust portion of the test.

## 9.9 Enterprise-specific scenarios

| Scenario | Why curl succeeds but Maven fails | What to prove |
|---|---|---|
| Browser SSO works, API token fails | Browser session ≠ artifact credential | 200 on protected `.pom` with token |
| Curl targets group repo; Maven targets hosted repo | Different ACLs/requirements | Exact matching URLs |
| Curl uses proxy; Maven bypasses proxy | Different routing/policies | Proxy settings and CONNECT target |
| Redirect changes hostname | Credentials may not be forwarded to new origin | Location and target response |
| Maven mirror ID differs from server ID | Maven may omit auth | Selected ID and settings ID |
| TLS interception presents corporate CA | Java truststore may differ from OS | JVM truststore if TLS exception appears |
| HTTP 200 body is HTML | An SSO login page masquerades as success | Content-Type and body |

## 9.10 Decision path for your incident

1. Extract the exact URL Maven failed to fetch and its selected repository ID.
2. Curl **that** URL anonymously and with the Nexus token; confirm correct artifact content.
3. If curl 200 and Maven 401: inspect mirror/server ID (7.3), effective settings (7.4), password decryption (3), and alternate settings (8).
4. If curl returns 3xx/HTML: inspect gateway/SSO/redirect handling before assuming a Maven credential bug.
5. If curl and Maven use different proxies: make routes equivalent under approved IT configuration.
6. If both reach the same endpoint but outcomes differ, provide redacted URL/ID/status/timestamps to Nexus admins for server-side log correlation.
7. If Maven reports PKIX / `SSLHandshakeException`, switch to CA/truststore check 5. A true HTTP 401 normally means the HTTPS handshake for that request succeeded.

## Completion checklist
- [ ] Exact failing Maven `.pom`/`.jar` URL captured
- [ ] Maven repository/mirror ID captured
- [ ] Anonymous curl code recorded
- [ ] Authenticated curl code and artifact content verified
- [ ] Redirect chain/host changes identified
- [ ] Maven and curl proxy routes compared
- [ ] Auth server ID and token configuration checked
- [ ] Gateway vs Nexus source of 401 narrowed down

**Further reading:** https://maven.apache.org/settings.html · https://maven.apache.org/guides/mini/guide-mirror-settings.html · https://curl.se/docs/manpage.html
