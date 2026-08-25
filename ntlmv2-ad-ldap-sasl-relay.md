# NTLMv2 Browser Authentication Against Active Directory via LDAP SASL Relay

This document is a companion to [`jcifs-ng-ad-authentication.md`](jcifs-ng-ad-authentication.md).

The earlier document explains why `jcifs-ng 2.1.10` must **not** be used as a modern browser-side NTLMv2 validator through its deprecated HTTP/SMB challenge path. This document describes the viable alternative that mirrors the proven design used by `express-ntlm`'s `NTLM_AD_Proxy`:

```text
Browser sends raw NTLM Type 1 / Type 3
            ↓
Spring Boot relays those NTLM tokens
            ↓
LDAP SASL bind using mechanism "GSS-SPNEGO"
            ↓
Active Directory generates Type 2 and validates Type 3
```

This is intended as a self-contained implementation guide for an offline coding agent.

> Target: Spring Boot 3 / Jakarta Servlet, NTLMv2 browser SSO, Active Directory, SMB1 disabled, no browser-side Kerberos/SPNEGO requirement.

---

# 1. Executive decision

For the target environment, the authentication architecture should be:

```text
Domain-joined Windows browser
        |
        | HTTP Authorization: NTLM <Type1>
        v
Spring Boot 3 authentication filter
        |
        | LDAPv3 SASL bind
        | mechanism = GSS-SPNEGO
        | SASL credentials = raw NTLM Type1 bytes
        v
Active Directory Domain Controller
        |
        | LDAP result 14: SASL_BIND_IN_PROGRESS
        | serverSaslCreds = NTLM Type2 challenge
        v
Spring Boot
        |
        | HTTP 401
        | WWW-Authenticate: NTLM <Type2>
        v
Browser
        |
        | HTTP Authorization: NTLM <Type3 / NTLMv2 response>
        v
Spring Boot
        |
        | SAME LDAP CONNECTION
        | second GSS-SPNEGO SASL bind
        | SASL credentials = raw NTLM Type3 bytes
        v
Active Directory
        |
        | LDAP SUCCESS => authenticated
        | anything else => rejected
```

The application does **not** know the user's password and does **not** calculate or verify the NTLMv2 response itself.

Active Directory performs the verification.

The application is acting as a controlled authentication relay between the browser's NTLM HTTP exchange and an LDAP SASL authentication exchange with the domain controller.

### Important terminology

The browser-facing protocol is still raw HTTP NTLM:

```http
WWW-Authenticate: NTLM
Authorization: NTLM <base64>
```

The backend-to-AD LDAP bind uses the SASL mechanism named:

```text
GSS-SPNEGO
```

That does **not** mean the browser must use `Negotiate`, Kerberos, or SPNEGO.

Microsoft explicitly documents that Active Directory's LDAP `GSS-SPNEGO` mechanism can use **Kerberos or NTLM** as the underlying authentication protocol. In this design the tokens being relayed are NTLM tokens.

---

# 2. Why this works when the old jcifs path does not

The old jcifs HTTP flow attempted to obtain an SMB server challenge and then validate the browser response through an SMB logon operation.

That old mechanism is:

```text
Browser NTLM
   ↓
jcifs getChallenge()
   ↓
SMB1-era server encryption key
   ↓
Type2 created by application
   ↓
Type3 validated through SMB logon
```

That is exactly the legacy path that breaks when SMB1 is unavailable and which jcifs-ng documents as deprecated/broken.

The LDAP relay architecture is different:

```text
Browser Type1
   ↓
AD receives Type1 through LDAP SASL
   ↓
AD itself creates the Type2 challenge
   ↓
Browser computes Type3 using that exact challenge
   ↓
AD receives Type3 on the same LDAP authentication exchange
   ↓
AD verifies the user's NTLM response
```

No SMB challenge is used.

Therefore:

```text
SMB1                        NOT REQUIRED
SMB2                        IRRELEVANT TO THIS AUTH FLOW
jcifs NtlmHttpFilter        NOT USED
jcifs getChallenge/logon    NOT USED
browser Kerberos            NOT REQUIRED
browser Negotiate           NOT REQUIRED
NTLMv2                      SUPPORTED BY AD
LDAP to DC                  REQUIRED
```

SMB protocol version and NTLM protocol version are separate concerns.

---

# 3. Proven reference: what `express-ntlm` actually does

The important reference implementation is:

```text
https://github.com/einfallstoll/express-ntlm
```

Relevant files:

```text
lib/express-ntlm.js
lib/NTLM_AD_Proxy.js
lib/NTLM_Proxy.js
```

At commit:

```text
66516470e9e19aa7649d5fe53518431a7a896241
```

## 3.1 LDAP request produced by `NTLM_AD_Proxy`

`NTLM_AD_Proxy.js` creates an LDAPv3 SASL bind whose authentication choice contains:

```text
mechanism   = "GSS-SPNEGO"
credentials = ntlm_token
```

Critically, the implementation passes the supplied **raw NTLM token** directly as the SASL credential bytes.

The code conceptually encodes:

```text
BindRequest
  version = 3
  name = ""
  authentication = SASL {
      mechanism = "GSS-SPNEGO"
      credentials = raw NTLM token
  }
```

Do not add an additional custom NTLM challenge or SMB challenge.

Do not transform Type1 into credentials belonging to a service account.

The browser's exact token is relayed.

## 3.2 Handling the first bind

`NTLM_AD_Proxy` recognizes:

```text
LDAP resultCode 0  = success
LDAP resultCode 14 = saslBindInProgress
```

For result code `14`, it reads the LDAP `serverSaslCreds` element and returns those bytes as the challenge.

`express-ntlm.js` then sends those bytes directly to the browser:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: NTLM <base64(serverSaslCreds)>
```

## 3.3 Handling Type 3

`express-ntlm` keeps the proxy object alive after Type1.

That proxy owns the open LDAP socket.

When the browser sends Type3, it calls `proxy.authenticate(type3)` using the **same LDAP connection**.

The second SASL bind carries the raw Type3 token.

If AD returns LDAP success, the user is considered authenticated.

## 3.4 Connection affinity in `express-ntlm`

The Node implementation intentionally associates authentication state with the HTTP connection:

```text
request.connection.id
```

and stores the corresponding live AD proxy connection in a cache.

This is not an implementation accident.

The NTLM exchange is stateful. The Type3 step must continue the authentication exchange initiated by Type1, against the same DC and the same LDAP connection.

---

# 4. Microsoft protocol facts this design relies on

Active Directory publishes supported SASL mechanisms through the RootDSE attribute:

```text
supportedSASLMechanisms
```

The target DC should advertise:

```text
GSS-SPNEGO
```

Microsoft's AD protocol documentation states that `GSS-SPNEGO` LDAP authentication supports:

```text
Kerberos
or
NTLM
```

References:

```text
https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/e1cbe214-d73b-4c58-aad2-bee399ccdfb8
https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/e7d814a5-4cb5-4b0d-b408-09d79988b550
https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/989e0748-0953-455d-9d37-d08dfbf3998b
```

The implementation must verify this capability against the real target domain before depending on it.

---

# 5. Recommended Java stack

Use Spring Security for application authentication state and use a low-level LDAP library that lets the application supply arbitrary SASL credential bytes.

Recommended dependency:

```xml
<dependency>
    <groupId>com.unboundid</groupId>
    <artifactId>unboundid-ldapsdk</artifactId>
    <version>7.0.5</version>
</dependency>
```

The relevant UnboundID class is:

```java
com.unboundid.ldap.sdk.GenericSASLBindRequest
```

Its purpose is explicitly to support generic SASL mechanisms where the caller:

- supplies the mechanism name,
- encodes the credentials,
- handles multi-stage processing,
- interprets server SASL credentials.

For result code `SASL_BIND_IN_PROGRESS`, UnboundID exposes:

```java
SASLBindInProgressException
```

and:

```java
getServerSASLCredentials()
```

References:

```text
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/GenericSASLBindRequest.html
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/SASLBindInProgressException.html
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/BindResult.html
```

### jcifs-ng's role

`jcifs-ng 2.1.10` may still be used for its protocol model/parser classes:

```java
jcifs.ntlmssp.Type1Message
jcifs.ntlmssp.Type2Message
jcifs.ntlmssp.Type3Message
```

Those classes can help validate and parse NTLMSSP blobs.

Do **not** use:

```java
jcifs.http.NtlmHttpFilter
jcifs.http.NtlmSsp
SmbTransportPool.getChallenge(...)
SmbTransportPool.logon(...)
```

for this architecture.

---

# 6. Suggested package structure

Keep this integration isolated from application business logic.

```text
security/
  ntlm/
    NtlmAuthenticationFilter.java
    NtlmRelayService.java
    NtlmHandshakeRegistry.java
    NtlmHandshake.java
    NtlmTokenInspector.java
    AdNtlmPrincipal.java
    NtlmAuthenticationException.java
```

Responsibilities:

```text
NtlmAuthenticationFilter
    HTTP challenge-response orchestration only

NtlmRelayService
    opens LDAP connection
    performs first SASL bind
    performs final SASL bind

NtlmHandshakeRegistry
    temporary live handshake storage
    timeout/cleanup

NtlmHandshake
    live LDAPConnection
    selected DC
    state
    createdAt / expiresAt

NtlmTokenInspector
    validates NTLMSSP signature/type
    parses Type3 identity metadata
    optionally enforces NTLMv2 response shape

AdNtlmPrincipal
    normalized application identity after AD success
```

No controller and no business service should manipulate raw NTLM tokens.

---

# 7. Handshake state machine

Use an explicit state machine.

```text
NEW
  |
  | no Authorization
  v
CHALLENGED_WITH_NTLM
  |
  | browser sends Type1
  v
WAITING_FOR_TYPE3
  |
  | AD returned Type2
  | live LDAP connection retained
  v
browser sends Type3
  |
  +---- AD success ----> AUTHENTICATED
  |
  +---- AD failure ----> FAILED
```

A handshake must never jump directly from `NEW` to Type3.

A stale Type3 must never be accepted.

A Type3 belonging to another handshake must never be accepted.

---

# 8. HTTP filter behavior

Implement the web layer as a Spring Boot 3 `OncePerRequestFilter` using Jakarta Servlet APIs.

Conceptual shell:

```java
public final class NtlmAuthenticationFilter extends OncePerRequestFilter {

    private final NtlmRelayService relayService;
    private final NtlmHandshakeRegistry registry;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // 1. If already authenticated, continue.
        // 2. Read Authorization header.
        // 3. No token -> issue initial NTLM 401.
        // 4. Type1 -> start LDAP SASL exchange and return Type2.
        // 5. Type3 -> continue SAME LDAP exchange and authenticate.
        // 6. Never trust identity metadata until AD returns success.
    }
}
```

## 8.1 Initial request

If no application authentication exists and no NTLM Authorization header is present:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: NTLM
Cache-Control: no-store
```

Do not call the protected controller.

## 8.2 Type1 request

Expected header:

```http
Authorization: NTLM <base64 Type1>
```

Process:

1. Strictly split the scheme and token.
2. Require scheme `NTLM`.
3. Base64-decode with a maximum accepted size.
4. Verify NTLMSSP signature:

```text
4e 54 4c 4d 53 53 50 00
N  T  L  M  S  S  P \0
```

5. Verify message type is `1`.
6. Create a new handshake ID.
7. Select one DC.
8. Open one LDAP connection to that DC.
9. Send the Type1 bytes through a generic `GSS-SPNEGO` SASL bind.
10. Expect `SASL_BIND_IN_PROGRESS`.
11. Extract `serverSASLCredentials`.
12. Verify those credentials contain a valid NTLM Type2 token.
13. Store the **open LDAP connection** in the handshake object.
14. Return:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: NTLM <base64 Type2>
```

The application must not generate its own Type2 challenge.

AD owns the challenge.

## 8.3 Type3 request

Expected header:

```http
Authorization: NTLM <base64 Type3>
```

Process:

1. Locate the outstanding handshake.
2. Require state `WAITING_FOR_TYPE3`.
3. Verify it is not expired.
4. Verify NTLMSSP signature.
5. Verify message type is `3`.
6. Optionally parse `Type3Message` for domain/user/workstation.
7. Send the exact Type3 bytes as SASL credentials over the handshake's **existing LDAP connection**.
8. Require LDAP result `SUCCESS`.
9. Only now trust the identity from the Type3 token.
10. Build a Spring Security principal.
11. Close the LDAP connection.
12. Remove the temporary handshake.
13. Continue the filter chain.

On any failure:

- close the LDAP connection,
- remove handshake state,
- clear Spring Security state,
- return a fresh `401 + WWW-Authenticate: NTLM` unless the failure is a server/DC outage.

---

# 9. UnboundID implementation sketch

The following is architectural pseudocode. The coding agent must compile it against the actual selected UnboundID version and adapt checked exceptions/API details rather than copy blindly.

## 9.1 Start negotiation

```java
public NtlmHandshake begin(byte[] type1) throws LDAPException {
    LDAPConnection connection = openConnectionToSelectedDc();

    GenericSASLBindRequest request = new GenericSASLBindRequest(
        null,
        "GSS-SPNEGO",
        new ASN1OctetString(type1)
    );

    try {
        BindResult unexpected = connection.bind(request);
        throw new NtlmAuthenticationException(
            "Expected SASL bind-in-progress but got " + unexpected.getResultCode());
    }
    catch (SASLBindInProgressException inProgress) {
        ASN1OctetString serverCredentials = inProgress.getServerSASLCredentials();

        if (serverCredentials == null) {
            connection.close();
            throw new NtlmAuthenticationException("AD returned no Type2 token");
        }

        byte[] type2 = serverCredentials.getValue();
        validateType2(type2);

        return new NtlmHandshake(
            UUID.randomUUID().toString(),
            connection,
            type2,
            Instant.now(),
            Instant.now().plusSeconds(30)
        );
    }
    catch (LDAPException e) {
        connection.close();
        throw e;
    }
}
```

### Critical point

`SASL_BIND_IN_PROGRESS` is **not an authentication failure**.

It is the expected first-stage response.

UnboundID models it with `SASLBindInProgressException`, specifically so callers can inspect `serverSASLCredentials` and continue the multi-stage bind.

## 9.2 Complete authentication

```java
public void complete(NtlmHandshake handshake, byte[] type3) throws LDAPException {
    validateType3(type3);

    GenericSASLBindRequest request = new GenericSASLBindRequest(
        null,
        "GSS-SPNEGO",
        new ASN1OctetString(type3)
    );

    BindResult result = handshake.connection().bind(request);

    if (result.getResultCode() != ResultCode.SUCCESS) {
        throw new NtlmAuthenticationException(
            "AD rejected NTLM authentication: " + result.getResultCode());
    }
}
```

In practice, a failed bind may be surfaced as `LDAPBindException`; handle it as authentication failure unless the exception represents a network/server problem.

Do not retry the Type3 against another DC.

If the bound DC or socket dies, abort the handshake and restart from Type1.

---

# 10. Raw token handling: do not accidentally change the protocol

The safest first implementation should reproduce the working `express-ntlm` behavior exactly:

```text
HTTP NTLM Type1 bytes
        ↓ unchanged
LDAP SASL credentials

LDAP serverSaslCreds Type2 bytes
        ↓ unchanged
HTTP WWW-Authenticate: NTLM

HTTP NTLM Type3 bytes
        ↓ unchanged
LDAP SASL credentials
```

Do not:

- generate another challenge,
- hash anything yourself,
- reconstruct a Type3 message,
- replace the domain/user fields,
- insert service-account credentials,
- downgrade to NTLMv1,
- wrap the token in a homemade SPNEGO structure unless the target DC demonstrably requires that behavior.

The reference `express-ntlm` implementation sends raw NTLMSSP bytes as the `GSS-SPNEGO` SASL credential payload.

A robust implementation should fail closed if the returned server SASL credential bytes are not what was expected.

---

# 11. Token validation and parsing

At minimum implement:

```java
boolean hasNtlmSignature(byte[] token)
int messageType(byte[] token)
```

Require:

```text
bytes 0..7 = "NTLMSSP\0"
```

and read the little-endian message type from the NTLMSSP header.

Expected directions:

```text
browser -> server : Type1 or Type3
AD -> server      : Type2
```

Reject a browser-supplied Type2.

Reject an AD response that is not Type2 during the negotiation step.

## Optional jcifs parser

After basic length/signature checks, use:

```java
new Type1Message(bytes)
new Type2Message(bytes)
new Type3Message(bytes)
```

for safer field parsing instead of manually indexing raw buffers everywhere.

## Enforcing NTLMv2

SMB1 being disabled does **not** automatically prove that the HTTP token is NTLMv2.

If the project requirement is strictly NTLMv2, either rely on a domain policy that rejects NTLMv1 or explicitly inspect the NT challenge response format before accepting Type3.

An NTLMv2 NT challenge response contains:

```text
16-byte NT proof
followed by an NTLMv2 client-challenge blob
```

and is therefore larger than the classic 24-byte NTLMv1 response. The blob begins with the NTLMv2 response-version fields (`01 01 00 00`).

If implementing this check, use a well-tested parser and bounds-check every offset.

Do not implement authentication cryptography yourself; this check is only a policy guard. AD remains the validator.

---

# 12. Handshake storage and lifecycle

A handshake contains a **live LDAP socket**, so it cannot simply be put in Redis or serialized into a shared session store.

Recommended model:

```java
record NtlmHandshake(
    String id,
    LDAPConnection connection,
    String domainController,
    Instant createdAt,
    Instant expiresAt,
    State state
) {}
```

Store it in local memory with a short timeout.

Example:

```text
maximum handshake lifetime: 20-60 seconds
```

The exact value is an operational decision.

Cleanup must close the LDAP connection.

Run cleanup on:

- successful authentication,
- authentication failure,
- malformed token,
- timeout,
- session invalidation,
- application shutdown.

Never leave half-completed LDAP binds open indefinitely.

---

# 13. Correlating Type1 and Type3

`express-ntlm` correlates by the underlying HTTP socket.

In Spring Boot there are two practical choices.

## Option A: HTTP connection identity

Closest to `express-ntlm`, but application servers and reverse proxies do not always expose a reliable client-connection identity.

This becomes especially fragile behind:

- reverse proxies,
- HTTP/2 multiplexing,
- connection pooling,
- ingress controllers.

## Option B: opaque handshake/session cookie

Recommended for a Spring application when the infrastructure preserves pod affinity.

Flow:

1. On Type1 create an unguessable random handshake ID.
2. Store `{handshakeId -> live LDAPConnection}` locally.
3. Set the ID in an HttpOnly/Secure cookie or server-side HTTP session.
4. Browser returns the cookie alongside Type3.
5. Resolve the matching live handshake.
6. Destroy it immediately after completion.

Never store the Type1/Type3 token itself in the browser cookie.

The cookie contains only an opaque identifier.

---

# 14. Kubernetes / multiple instances: critical deployment constraint

Because a live `LDAPConnection` exists only inside one JVM, Type1 and Type3 must reach the **same application instance**.

This is mandatory.

If Type1 hits pod A:

```text
pod A owns LDAP socket A -> DC
```

and Type3 hits pod B:

```text
pod B does not own the authentication exchange
```

so authentication must fail/restart.

Therefore one of these is required during the handshake:

```text
load-balancer sticky sessions
or
connection affinity
or
a dedicated NTLM authentication edge/service
```

Do not attempt to copy the LDAP socket through Redis.

Do not open a new LDAP connection on Type3.

Do not fail over to a different DC halfway through the handshake.

A scalable design may authenticate once through a dedicated NTLM adapter and then issue the rest of the application a normal internal session/JWT/token.

That prevents every downstream microservice from needing NTLM state.

---

# 15. Recommended Spring Security boundary

NTLM belongs at one authentication boundary only.

After AD validates Type3, translate the identity into a normal application principal.

Example:

```java
AdNtlmPrincipal principal = new AdNtlmPrincipal(
    domain,
    username,
    workstation
);

Authentication authentication =
    UsernamePasswordAuthenticationToken.authenticated(
        principal,
        null,
        authorities
    );

SecurityContextHolder.getContext().setAuthentication(authentication);
```

Do not pass raw NTLM tokens to downstream services.

Do not make every microservice authenticate against AD independently.

Recommended boundary:

```text
Browser
  ↓ NTLM
Gateway / authentication service
  ↓ internal authenticated identity
Backend services
```

After successful SSO, downstream authorization should use standard application identity/roles.

---

# 16. Identity enrichment

The successful Type3 tells the application that AD accepted the NTLM authentication exchange for the identity represented in the NTLM message.

The application may then separately query directory information such as:

```text
userPrincipalName
objectSid
displayName
memberOf
mail
```

Do this as a separate operation with an application-approved directory read identity if necessary.

Do not reuse the relayed user's LDAP connection for arbitrary directory queries unless the security design explicitly requires it and supports the resulting SASL security layer.

Authentication and directory lookup should be separate concerns.

---

# 17. Configuration model

Example application configuration:

```yaml
security:
  ntlm:
    enabled: true
    domain: EXAMPLE
    handshake-timeout: 30s
    domain-controllers:
      - dc01.example.local
      - dc02.example.local
    ldap:
      mode: ldap # or ldaps, based on approved AD policy
      port: 389
```

Do not put credentials into this configuration because this relay design does not require a user's password or a service-account password for the authentication exchange itself.

If a separate service account is used for post-auth directory lookups, load that secret from the approved secret store.

---

# 18. DC selection and failover

Failover is allowed **before** the Type2 challenge exists.

Example Type1 behavior:

```text
try dc01
  connection failed -> close
try dc02
  first SASL stage succeeds -> pin handshake to dc02
```

After a DC returns Type2:

```text
handshake is pinned to that exact DC and LDAP connection
```

If the socket dies after Type2:

```text
abort handshake
return fresh NTLM 401
browser starts again with Type1
```

Never send Type3 generated from DC A's challenge to DC B on a newly created authentication context.

---

# 19. LDAP vs LDAPS and AD hardening

This section is critical.

This architecture is functionally an **NTLM authentication relay** to LDAP. Modern Active Directory hardening features are specifically designed to limit unsafe relay scenarios.

The fact that `express-ntlm` can implement this protocol does not guarantee that every hardened AD domain will permit it.

## 19.1 LDAP signing

Microsoft documents that a DC can require LDAP signing and reject SASL LDAP binds that do not request the required integrity protection.

The simple raw relay implemented by `express-ntlm` does not possess the user's NTLM session key and therefore cannot trivially become a full signed LDAP client after the bind.

A domain with strict LDAP signing requirements may reject this approach on port 389.

## 19.2 LDAP channel binding / CBT

For SASL authentication over TLS/LDAPS, Microsoft can require LDAP Channel Binding Tokens.

CBT binds the authentication exchange to the TLS channel and is specifically useful for preventing relayed NTLM authentication.

If the DC has strict channel binding requirements, taking an NTLM token originating from the browser's HTTP/TLS channel and replaying it into a different LDAP/TLS channel may fail by design.

Microsoft references:

```text
https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing
https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023
```

## 19.3 Required rule for the coding agent

Do **not** weaken domain security policy in code or documentation just to make this design work.

Before production implementation, test the target DC policy.

If the domain intentionally blocks this relay through LDAP signing/channel binding requirements, stop and escalate the architecture decision instead of silently disabling protections.

---

# 20. Security requirements

At minimum:

- HTTPS on the browser-facing application.
- Never enable Basic authentication as a fallback unless separately approved.
- Never collect the Windows password in an HTML form.
- Never log `Authorization` headers.
- Never log Type1, Type2, or Type3 blobs.
- Never persist NTLM challenge-response material.
- Bound Base64/token sizes before allocation/parsing.
- Use strict NTLMSSP signature/type validation.
- Use short handshake expiration.
- Close abandoned LDAP connections.
- Rate-limit repeated failed authentication attempts.
- Protect the handshake cookie/session identifier.
- Regenerate the application session ID after successful authentication.
- Do not trust `domain`, `username`, or `workstation` until AD returns bind success.
- Do not treat the workstation field as a trustworthy device identity.
- Keep DC certificate validation enabled when using LDAPS; never install a trust-all TLS manager.

---

# 21. Browser requirements

The browser must be willing to perform integrated NTLM authentication for the application URL.

Typical enterprise policy determines whether credentials are sent automatically based on:

- domain membership,
- browser policy,
- intranet/trusted-site classification,
- hostname/FQDN,
- security zone.

The Java application cannot force a browser to disclose Windows credentials.

Its job is only to issue the NTLM HTTP challenge correctly.

Initial challenge:

```http
WWW-Authenticate: NTLM
```

Type2 challenge:

```http
WWW-Authenticate: NTLM <base64-Type2>
```

---

# 22. Reverse proxy and HTTP protocol considerations

NTLM has historically been connection-oriented and can interact badly with proxies that multiplex or pool unrelated clients.

Before production rollout verify the entire path:

```text
browser
  -> corporate proxy if any
  -> ingress/load balancer
  -> Spring Boot instance
```

Questions to validate:

- Does the browser perform the Type1/Type3 exchange over a stable HTTP connection?
- Does the load balancer preserve affinity between the two requests?
- Is HTTP/2 enabled and does it affect NTLM behavior in the target browser/proxy stack?
- Does the ingress reuse backend connections across unrelated clients?
- Is the handshake session cookie preserved?

Do not assume local direct testing proves Kubernetes/production behavior.

---

# 23. Error semantics

Recommended responses:

## No auth / restart handshake

```http
401 Unauthorized
WWW-Authenticate: NTLM
```

## Malformed NTLM message

```text
400 Bad Request
```

Do not echo the malformed token.

## AD rejects credentials

Prefer restarting the authentication challenge:

```http
401 Unauthorized
WWW-Authenticate: NTLM
```

## DC unavailable / network failure

Use an application/server failure such as:

```text
503 Service Unavailable
```

rather than pretending the user supplied invalid credentials.

## Handshake expired / wrong pod

Delete any state and restart:

```http
401 Unauthorized
WWW-Authenticate: NTLM
```

---

# 24. Testing plan

The agent must not mark this implementation complete without a real AD integration test.

## 24.1 Protocol tests

- No Authorization -> `401 + WWW-Authenticate: NTLM`.
- Type1 is accepted.
- AD returns LDAP `SASL_BIND_IN_PROGRESS`.
- `serverSaslCreds` is present.
- Returned challenge is NTLM Type2.
- Browser receives the exact Type2 returned by AD.
- Browser sends Type3.
- Same LDAP connection receives Type3.
- AD success creates Spring authentication.
- Wrong/stale Type3 is rejected.
- Type3 without outstanding handshake is rejected.

## 24.2 NTLMv2 requirement tests

Test with domain policy enforcing NTLMv2 / rejecting NTLMv1.

Do not infer NTLMv2 from SMB2.

## 24.3 SMB1-disabled test

Explicitly prove:

```text
SMB1 remains disabled
and authentication still succeeds
```

There should be no SMB traffic involved in this implementation.

## 24.4 DC failure tests

- first DC unreachable -> try next DC before Type2,
- DC dies after Type2 -> restart handshake,
- LDAP timeout,
- LDAP connection reset,
- invalid certificate on LDAPS -> fail closed.

## 24.5 Multi-instance tests

Run at least two application replicas and verify:

- no stickiness -> expected handshake failures are understood,
- configured stickiness/affinity -> Type1 and Type3 reach the same pod,
- expired pod-local handshake cleans up correctly.

## 24.6 AD hardening tests

Test against actual configured policies for:

```text
LDAP signing
LDAP channel binding
NTLM restrictions
```

If the environment blocks the relay, record the exact DC event/result and stop rather than weakening policy automatically.

---

# 25. Suggested implementation sequence for the offline coding agent

Follow this order.

## Phase 1 - proof of protocol

Build a minimal non-Spring Java spike:

```text
input: captured/test Type1 token
open LDAP connection
send GenericSASLBindRequest(GSS-SPNEGO, Type1)
confirm SASL_BIND_IN_PROGRESS
print only token type/length, never token bytes
```

Goal: prove the target DC returns an NTLM Type2 through `serverSaslCreds`.

## Phase 2 - two-stage LDAP bind

Add Type3 continuation on the same LDAP connection and prove:

```text
valid domain user -> SUCCESS
invalid/stale response -> rejected
```

## Phase 3 - HTTP adapter

Add Spring `OncePerRequestFilter` and relay:

```text
HTTP Type1 <-> LDAP first stage
HTTP Type3 <-> LDAP final stage
```

## Phase 4 - Spring Security

Create a normalized principal and persistent authenticated application session.

## Phase 5 - production lifecycle

Add:

- handshake registry,
- expiration,
- connection cleanup,
- metrics,
- rate limiting,
- DC failover before challenge,
- load-balancer affinity.

## Phase 6 - security validation

Validate with the AD/security team:

- NTLMv2 enforcement,
- LDAP signing policy,
- CBT policy,
- allowed ports,
- browser policy,
- deployment topology.

---

# 26. Observability

Safe metrics:

```text
ntlm_handshake_started_total
ntlm_handshake_success_total
ntlm_handshake_failure_total
ntlm_handshake_expired_total
ntlm_dc_connection_failure_total
ntlm_sasl_bind_in_progress_total
ntlm_sasl_bind_rejected_total
ntlm_handshake_duration_seconds
ntlm_open_handshakes
```

Safe structured log fields:

```text
handshakeId hash/short ID
selected DC hostname
state transition
LDAP result code
elapsed milliseconds
```

Do not log:

```text
Authorization header
raw Type1
raw Type2
raw Type3
LM response
NT response
session keys
```

User/domain may be logged only according to the application's normal identity/privacy logging policy and preferably only after successful authentication.

---

# 27. Things the agent must NOT do

Do not use jcifs `NtlmHttpFilter`.

Do not re-enable SMB1.

Do not call jcifs `getChallenge()`.

Do not call jcifs `logon()` to validate the browser's response.

Do not calculate a Type2 challenge locally.

Do not use a new LDAP connection for Type3.

Do not switch domain controllers after Type2.

Do not send the user's NTLM token to an arbitrary LDAP host.

Do not trust the username before AD success.

Do not silently fall back to Basic authentication.

Do not use a trust-all TLS configuration.

Do not weaken LDAP signing or CBT policy automatically.

Do not confuse:

```text
HTTP "NTLM" scheme
LDAP "GSS-SPNEGO" SASL mechanism
Kerberos
SMB version
NTLM version
```

They are separate layers.

---

# 28. Architecture summary

The intended implementation is:

```text
                         Browser-facing leg
                  -------------------------------
                  raw HTTP NTLM, not Negotiate

Domain-joined browser
        |
        | Type1
        v
+-------------------------------+
| Spring Boot NTLM Filter       |
|                               |
| - validate token shape        |
| - create handshake            |
| - keep live LDAP connection   |
+---------------+---------------+
                |
                | LDAP SASL Bind
                | mech: GSS-SPNEGO
                | credential: raw NTLM Type1
                v
+-------------------------------+
| Active Directory DC           |
|                               |
| creates NTLM Type2 challenge  |
+---------------+---------------+
                |
                | resultCode 14
                | serverSaslCreds = Type2
                v
+-------------------------------+
| Spring Boot                   |
+---------------+---------------+
                |
                | 401 + WWW-Authenticate: NTLM Type2
                v
Domain-joined browser
        |
        | Type3 / NTLMv2 response
        v
+-------------------------------+
| SAME Spring instance          |
| SAME handshake                |
| SAME LDAP connection          |
+---------------+---------------+
                |
                | second GSS-SPNEGO SASL bind
                | credential: raw NTLM Type3
                v
+-------------------------------+
| Active Directory DC           |
| validates NTLM credentials    |
+---------------+---------------+
                |
            LDAP SUCCESS
                |
                v
+-------------------------------+
| Spring Security principal     |
| normal application session    |
+-------------------------------+
```

No SMB1 is involved.

---

# 29. Source-of-truth links

## express-ntlm reference implementation

```text
https://github.com/einfallstoll/express-ntlm/blob/66516470e9e19aa7649d5fe53518431a7a896241/lib/NTLM_AD_Proxy.js
https://github.com/einfallstoll/express-ntlm/blob/66516470e9e19aa7649d5fe53518431a7a896241/lib/NTLM_Proxy.js
https://github.com/einfallstoll/express-ntlm/blob/66516470e9e19aa7649d5fe53518431a7a896241/lib/express-ntlm.js
```

## Microsoft AD protocol documentation

```text
https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/e1cbe214-d73b-4c58-aad2-bee399ccdfb8
https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/e7d814a5-4cb5-4b0d-b408-09d79988b550
https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/989e0748-0953-455d-9d37-d08dfbf3998b
https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing
https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/ldap-session-security-settings-requirements-adv190023
```

## UnboundID LDAP SDK

```text
https://github.com/pingidentity/ldapsdk
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/GenericSASLBindRequest.html
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/SASLBindInProgressException.html
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/BindResult.html
```

## Companion jcifs analysis

```text
jcifs-ng-ad-authentication.md
```

---

# 30. Final instruction to an offline coding agent

Implement this as an **AD-backed NTLM relay**, not as custom NTLM cryptography and not as an SMB authentication hack.

The fundamental invariant is:

```text
AD creates the Type2 challenge
and
AD validates the Type3 response
```

The Spring application only transports the tokens and manages the multi-stage state.

The most important implementation properties are:

```text
raw browser NTLM tokens
        +
LDAP SASL mechanism GSS-SPNEGO
        +
same DC
        +
same live LDAP connection
        +
short-lived handshake state
        +
AD bind SUCCESS as the sole authentication decision
```

If the target Active Directory security policy rejects this relay because of LDAP signing, channel binding, or NTLM restrictions, do not work around those controls silently. Record the incompatibility and escalate the architecture decision.