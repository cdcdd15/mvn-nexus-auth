# Check 9 — Verify Maven/Nexus routing through mirrors, proxies and redirects

**Problem:** Authenticated curl succeeds but Maven receives HTTP `401 Unauthorized`. The requests may not be equivalent: Maven may select another repository URL, use a corporate proxy, encounter a gateway, follow redirects differently, or send credentials to a different endpoint ID.

## Three independent routing layers

| Layer | Question | Evidence |
|---|---|---|
| **Maven mirror** | Which repository URL does Maven choose? | `mvn -X`, `<mirrors>` in effective settings |
| **Outbound HTTP(S) proxy** | Does Maven connect through a proxy? | `<proxies>`, system/environment config, debug lines |
| **Nexus reverse proxy / SSO gateway** | Who returned 401, and did a redirect occur? | HTTP response headers / Nexus logs |

A Maven `mirror` is **not** a network proxy. It replaces the artifact repository endpoint in Maven's repository selection. A corporate HTTP proxy forwards network traffic. A reverse proxy/gateway is on the server side and may terminate TLS and enforce authentication.

## 9.1 Determine Maven's real artifact endpoint

```bash
mvn -X -e clean install > /tmp/mvn-check9.log 2>&1

grep -Ei 'Using mirror|Downloading from|from/to|401|Unauthorized' \
  /tmp/mvn-check9.log
```

Record: artifact coordinates, repository ID, repository base URL, and HTTP status. Compute the exact artifact URL if necessary using Maven's path layout, or use the full URL from the error/debug output. For example:

```text
Base: https://nexus.company.com/repository/maven-public/
Path: org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom
```

Avoid comparing Maven's `.pom` request to a browser's `/#browse` URL: one targets artifact storage, the other is UI navigation.

## 9.2 Compare curl with the *same* artifact URL

```bash
ARTIFACT_URL='https://nexus.company.com/repository/maven-public/org/apache/commons/commons-lang3/3.14.0/commons-lang3-3.14.0.pom'

curl -sS -o /dev/null -w 'anonymous=%{http_code} redirect=%{redirect_url}\n' \
  "$ARTIFACT_URL"

curl -sS -o /dev/null -w 'authenticated=%{http_code} redirect=%{redirect_url}\n' \
  -u 'TOKEN_USERNAME' "$ARTIFACT_URL"
```

`%{http_code}` and `%{redirect_url}` are literal curl format variables, **not placeholders**. `TOKEN_USERNAME` and the URL are examples. When using `-u 'TOKEN_USERNAME'`, curl prompts for the password interactively.

If anonymous and authenticated requests both return `200`, the artifact may be publicly readable; that comparison alone does not establish that Nexus validated your token. Test a known protected artifact when authorized.

## 9.3 Read redirects without leaking credentials

```bash
curl -sS -D - -o /dev/null "$ARTIFACT_URL"
```

Check for HTTP `301`, `302`, `307`, `308` and `Location:`. `-L` makes curl follow redirects, which changes what you are testing. Curl normally protects credentials from being forwarded across different hosts by default, but do not use `--location-trusted` casually and do not reveal credentials to an untrusted redirect target.

A gateway might redirect artifact requests to an SSO login HTML page; receiving HTML `200` after following redirects is **not** evidence that Maven downloaded the requested `.pom`. Check the response content type and whether it is real artifact XML.

## 9.4 Inspect Maven proxy configuration

Example structure in `settings.xml`:

```xml
<proxies>
  <proxy>
    <id>company-proxy</id>
    <active>true</active>
    <protocol>http</protocol>
    <host>proxy.company.com</host>
    <port>8080</port>
    <nonProxyHosts>localhost|127.0.0.1|*.company.com</nonProxyHosts>
  </proxy>
</proxies>
```

This is **illustrative**, not a recommendation to add one. The `<protocol>` describes proxy configuration; the target repository can still be `https://...`. Proxy-specific credentials, if needed, belong in the corresponding proxy settings, not in Maven repository `<servers>`.

Check the effective proxy configuration via `mvn help:effective-settings` where available, and examine runtime JVM/transport-specific proxy flags. An HTTP `407 Proxy Authentication Required` points more directly to proxy authentication than HTTP `401`, although corporate gateways may return other statuses.

## 9.5 Check whether curl and Maven take the same network route

Curl may use environment variables such as `HTTPS_PROXY`, `HTTP_PROXY`, and `NO_PROXY`, whereas Maven typically relies on its own Maven proxy settings (and possibly JVM/transport configuration). Compare without exposing credentials:

```bash
printf 'HTTPS_PROXY configured: %s\n' "${HTTPS_PROXY:+yes}"
printf 'NO_PROXY configured: %s\n' "${NO_PROXY:+yes}"
```

Use `curl -v` locally to see a connection target, proxy `CONNECT` tunnel, TLS negotiation, and HTTP status:

```bash
curl -v -u 'TOKEN_USERNAME' "$ARTIFACT_URL" -o /dev/null
```

**Redact** `Authorization`, `Proxy-Authorization`, cookies, token-bearing URLs, hostnames and any corporate network details before sharing the output. Curl verbose logs can contain request headers or authentication state. Do not share raw traces.

## 9.6 Identify which component returned the 401

Useful response headers (where exposed):

- `WWW-Authenticate` — which HTTP auth challenge is being offered?
- `Server` — clue, not definitive proof, about server/gateway.
- `Via` — intermediary information, if supplied.
- `Location` — redirect target.
- Request/trace ID — useful for Nexus or gateway administrators.

A Nexus repository manager, front-end reverse proxy, SSO gateway, and outbound proxy can each reject access for different reasons. If available, request read-only server-side logs for the failing timestamp/endpoint. Never assume the `401` must come directly from Nexus.

## 9.7 Differences that create deceptive results

| Curl test | Maven request | Why results differ |
|---|---|---|
| Nexus UI homepage | `.pom` artifact URL | Browser/SSO vs repository HTTP authentication |
| Group `maven-public` | Hosted `maven-releases` | Different ACLs and auth policy |
| HTTPS hostname A | Redirected hostname B | Different authentication origin |
| Curl uses `HTTPS_PROXY` | Maven bypasses proxy | Different gateways/path/policies |
| Curl sends a valid user token | Maven selects wrong server ID | Credentials absent from Maven request |
| Curl receives login HTML | Maven expects POM/JAR | Successful page delivery ≠ artifact download |

## 9.8 Recommended proof sequence

1. Take the exact `.pom` or `.jar` URL Maven failed on.
2. Curl it without credentials and with token credentials; record **HTTP codes**.
3. Inspect any redirect (`Location`) and whether the response is actual artifact content.
4. Confirm Maven effective mirror ID and URL (#7).
5. Compare configured Maven proxy and curl routing.
6. If identical paths still differ, collect redacted debug output and ask the Nexus/gateway admin to correlate request logs.

**Success criterion:** Maven and curl target the same protected artifact through equivalent routing, using the expected authentication mechanism. If they still differ, the problem is sufficiently isolated for examining auth transport behavior and server logs.

**References:** [Maven Settings](https://maven.apache.org/settings.html), [Maven Mirrors](https://maven.apache.org/guides/mini/guide-mirror-settings.html), [curl manual](https://curl.se/docs/manpage.html).
