# jcifs-ng 2.1.10 AD Authentication Guide

This document is a self-contained reference for an offline coding agent implementing Active Directory authentication with **jcifs-ng 2.1.10** in a modern Java/Spring application.

It answers the specific design questions that matter before implementation and makes the constraints explicit so an offline agent does not infer unsupported behavior from the JAR alone.

> Scope: jcifs-ng **2.1.10**, server-side HTTP NTLM, SMB protocol configuration, and Spring Boot 3 / Jakarta integration.

## Executive decision

`jcifs-ng` is a good SMB client library, but its built-in HTTP NTLM server-side stack is **legacy**.

In 2.1.10:

- `jcifs.http.NtlmHttpFilter` exists but is `@Deprecated` and documented as **NTLMv1 only**.
- `jcifs.http.NtlmSsp` is also `@Deprecated`.
- `jcifs.smb.NtlmPasswordAuthentication` is deprecated in favor of `NtlmPasswordAuthenticator`.
- `SmbTransportPool.getChallenge(...)` and `logon(...)` are deprecated and explicitly documented as **broken** and slated for removal; the library says to use real Active Directory authentication instead.
- The HTTP classes use `javax.servlet.*`, not `jakarta.servlet.*`, so they are not directly usable as Spring Boot 3 filters.

Therefore there are two distinct cases:

1. **Modern AD / NTLMv2 / Spring Boot 3**: do **not** build a new authentication system around `NtlmHttpFilter`, `getChallenge()`, and `logon()`. jcifs-ng 2.1.10 does not provide a supported modern server-side NTLMv2 implementation.
2. **Legacy compatibility requirement where NTLMv1 is explicitly allowed by the domain and security policy**: a custom Jakarta filter may reproduce the historical jcifs flow. This is compatibility code, not a modern recommended authentication design.

An offline agent must not silently cross from case 1 into case 2.

---

## 1. Is `jcifs.http.NtlmHttpFilter` supported in 2.1.10?

The class is present, but it is deprecated:

```java
/**
 * @deprecated NTLMv1 only
 */
@Deprecated
public class NtlmHttpFilter implements Filter
```

The implementation itself explains why:

```java
/*
 * The Filter can only work with NTLMv1 as it uses a man-in-the-middle
 * technique that NTLMv2 specifically thwarts. A real NTLM Filter would
 * need to do a NETLOGON RPC that JCIFS will likely never implement.
 */
```

### Conclusion

Treat `NtlmHttpFilter` as **legacy compatibility code**, not as the supported modern way to authenticate browsers against Active Directory.

The fact that it is shipped in the JAR does not mean it is a recommended integration point.

---

## 2. What does jcifs-ng say about `getChallenge()` and `logon()`?

The relevant `SmbTransportPool` methods are deprecated:

```java
@Deprecated
void logon(CIFSContext tc, Address dc) throws CIFSException;

@Deprecated
byte[] getChallenge(CIFSContext tc, Address dc) throws CIFSException;
```

Their Javadoc says, in substance:

- the functionality is broken,
- it will be removed at some point,
- use actual Active Directory authentication instead.

### Consequence

Do not interpret the existence of:

```text
getChallenge -> Type2 -> Type3 -> NtlmPasswordAuthentication -> logon
```

as proof that this is still a supported authentication architecture.

It is the historical implementation path retained for compatibility.

---

## 3. What is the historical server-side HTTP NTLM flow?

The old jcifs HTTP stack performs the following sequence.

```text
Browser                      Application                     Domain Controller
   |                              |                                  |
   | request protected URL        |                                  |
   |----------------------------->|                                  |
   |                              |                                  |
   | 401 WWW-Authenticate: NTLM   |                                  |
   |<-----------------------------|                                  |
   |                              |                                  |
   | Authorization: NTLM Type1    |                                  |
   |----------------------------->|                                  |
   |                              | -- getChallenge() -------------->|
   |                              |<-- 8-byte challenge -------------|
   |                              |                                  |
   |                              | build Type2 using challenge      |
   | 401 + NTLM Type2             |                                  |
   |<-----------------------------|                                  |
   |                              |                                  |
   | Authorization: NTLM Type3    |                                  |
   |----------------------------->|                                  |
   |                              | parse domain/user/LM/NT response |
   |                              | build external-hash credentials  |
   |                              | -- logon()/SMB validation ------>|
   |                              |<-- accepted / rejected ----------|
   |                              |                                  |
   | authenticated request        |                                  |
```

`NtlmSsp.authenticate(...)` parses the HTTP `Authorization: NTLM ...` token.

For Type 1 it creates a Type 2 message using the supplied challenge.

For Type 3 it extracts:

- domain,
- username,
- LM response,
- NT response,

and constructs:

```java
new NtlmPasswordAuthentication(
    type3.getDomain(),
    type3.getUser(),
    challenge,
    lmResponse,
    ntResponse
)
```

The five-argument constructor exists specifically for raw externally supplied hashes produced by the NTLM HTTP authentication path.

### Important limitation

This historical pass-through mechanism is exactly the part jcifs-ng labels legacy/broken. Do not generalize it into a modern NTLMv2 design.

---

## 4. Why does this not become NTLMv2 just because `Type3Message` supports NTLMv2?

jcifs-ng contains NTLMv2 primitives for its SMB client authentication path. That is different from providing a secure server-side HTTP NTLM validator.

`Type3Message` can construct NTLMv2 responses when jcifs is acting as an SMB client and has the user's password/hash material.

The deprecated HTTP filter instead performs a challenge pass-through / man-in-the-middle style validation flow. The project explicitly states that NTLMv2 defeats that technique.

Therefore:

```text
NTLMv2-capable crypto primitives in jcifs-ng
                     !=
Supported NTLMv2 HTTP server authentication
```

An offline agent must keep those two concepts separate.

---

## 5. Exact SMB2 configuration keys in 2.1.10

For jcifs-ng 2.1.x, the canonical protocol range properties are:

```properties
jcifs.smb.client.minVersion=...
jcifs.smb.client.maxVersion=...
```

The old properties:

```properties
jcifs.smb.client.enableSMB2=true
jcifs.smb.client.disableSMB1=true
```

are legacy/deprecated compatibility properties.

### Defaults in 2.1.10

If `minVersion` / `maxVersion` are not supplied, jcifs-ng falls back to:

```text
minVersion = SMB1
maxVersion = SMB210
```

`PropertyConfiguration` also defaults the old compatibility flags to:

```text
enableSMB2 = true
disableSMB1 = false
```

So SMB2 is enabled by default, but SMB1 is also permitted by default.

### SMB2-only example

For ordinary SMB client access where SMB1 must be disabled:

```properties
jcifs.smb.client.minVersion=SMB202
jcifs.smb.client.maxVersion=SMB210
```

Do not apply this blindly to the historical HTTP NTLM validation path.

---

## 6. Why SMB2-only breaks the old HTTP challenge path

The historical HTTP filter obtains the authentication challenge through:

```java
getTransportPool().getChallenge(...)
```

which ultimately asks the SMB transport for the server encryption key.

In jcifs-ng 2.1.10, `SmbTransportImpl.getServerEncryptionKey()` returns an encryption key only for an SMB1 negotiation object. For SMB2 negotiation it returns `null`.

Therefore the old challenge-pass-through flow is fundamentally coupled to the SMB1-era mechanism.

### Rule for the offline agent

Do not try to make the deprecated HTTP NTLM mechanism "modern" by forcing SMB2.

For normal file-share/client operations, use SMB2+.

For the legacy HTTP NTLM compatibility flow, understand that it depends on SMB1-era behavior and is not a modern NTLMv2 solution.

---

## 7. Spring Boot 3 / Jakarta compatibility

jcifs-ng 2.1.10 HTTP classes import:

```java
javax.servlet.Filter
javax.servlet.http.HttpServletRequest
javax.servlet.http.HttpServletResponse
```

The 2.1.10 POM declares:

```text
javax.servlet:javax.servlet-api:3.1.0
```

Spring Boot 3 uses Jakarta Servlet:

```java
jakarta.servlet.*
```

Therefore `jcifs.http.NtlmHttpFilter` cannot be directly registered as a Spring Boot 3/Jakarta filter.

### Do not do this

Do not attempt to solve the namespace mismatch by mixing `javax.servlet` and `jakarta.servlet` APIs in one filter chain.

### If legacy compatibility is explicitly required

Write a small application-owned Jakarta filter and reuse only the jcifs protocol/model classes that are not tied to `javax.servlet`.

That application-owned filter should be treated as a port of the old algorithm, not as a wrapper around `NtlmHttpFilter`.

---

# Implementation instructions for an offline agent

The following instructions are intentionally explicit.

## Step 0: classify the requirement before writing code

Before implementation, determine which statement is true.

### Case A - modern domain

Any of the following means **stop the jcifs HTTP implementation**:

- NTLMv2 is required.
- NTLMv1 is disabled by AD policy.
- SMB1 is disabled and cannot be enabled for this path.
- Security policy forbids NTLMv1.
- The application must provide a supported production-grade Windows Integrated Authentication flow.

In that case, do not fabricate a jcifs solution. Record that jcifs-ng 2.1.10 does not provide the required server-side NTLMv2 validation mechanism.

### Case B - legacy domain compatibility

Proceed only when all of these are explicitly true:

- the environment intentionally allows NTLMv1,
- the security owner accepts the legacy flow,
- the relevant domain controller/server can provide the SMB1 challenge path,
- the implementation exists only for compatibility with that environment.

---

## Step 1: dependencies

Use the pinned library version:

```xml
<dependency>
    <groupId>eu.agno3.jcifs</groupId>
    <artifactId>jcifs-ng</artifactId>
    <version>2.1.10</version>
</dependency>
```

Spring Boot 3 supplies the Jakarta servlet API.

Do not add `javax.servlet-api` to the application just to instantiate the deprecated jcifs HTTP filter.

---

## Step 2: create one jcifs context

Construct a long-lived `CIFSContext` from `PropertyConfiguration` and `BaseContext`.

Conceptually:

```java
Properties properties = new Properties();
properties.setProperty("jcifs.smb.client.domain", adDomain);

Configuration configuration = new PropertyConfiguration(properties);
CIFSContext baseContext = new BaseContext(configuration);
```

The DC name/address belongs in application configuration consumed by the validator; note that `jcifs.http.domainController` is a convention used by the deprecated HTTP layer, not a core `PropertyConfiguration` SMB setting.

Manage the context as an application singleton and close it on shutdown.

Never recreate a full context on every request.

Do not put passwords or other secrets in source control.

---

## Step 3: implement a Jakarta `OncePerRequestFilter`

For Spring Boot 3, own the servlet integration in the application.

Suggested shape:

```java
final class LegacyNtlmAuthenticationFilter extends OncePerRequestFilter {
    // parse Authorization header
    // manage handshake state
    // populate Spring Security Authentication only after validation
}
```

The filter should protect only the intended routes.

Do not copy unrelated behavior from the old filter.

---

## Step 4: handshake state machine

Implement the HTTP state machine explicitly.

### No Authorization header

Return:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: NTLM
```

Do not continue the Spring filter chain.

### NTLM Type 1

1. Base64-decode the token after `NTLM `.
2. Parse it as `Type1Message`.
3. Resolve the configured domain controller.
4. Obtain the legacy challenge using the same semantics as the old jcifs filter.
5. Store the challenge in the HTTP session or another request-correlated server-side state store.
6. Build `Type2Message` using the Type 1 message and challenge.
7. Return `401` with:

```http
WWW-Authenticate: NTLM <base64-Type2>
```

Do not authenticate the request yet.

### NTLM Type 3

1. Retrieve the challenge saved during Type 1 -> Type 2.
2. Parse the Type 3 token with `Type3Message`.
3. Extract domain and username.
4. Extract LM and NT response arrays.
5. Construct the historical external-hash credential object with the five-argument constructor.
6. Wrap the base context with those credentials before calling the deprecated validator:

```java
CIFSContext authContext = baseContext.withCredentials(ntlmPasswordAuthentication);
```

7. Validate the response using the historical `getChallenge()` / `logon()` path only if this environment has explicitly accepted that legacy behavior.
8. On success, create a Spring Security `Authentication` representing `DOMAIN\\username`.
9. Store authentication in the `SecurityContext`.
10. Remove temporary challenge material from the session.
11. Continue the filter chain.

On validation failure, clear temporary state and return another `401` with `WWW-Authenticate: NTLM`.

### Critical implementation detail

The deprecated jcifs source calls `transportContext.getTransportPool().logon(transportContext, dc)` after producing an `NtlmPasswordAuthentication`. If you reproduce the flow yourself, **do not accidentally validate with the anonymous/base context**. The credentials extracted from Type 3 must be attached to the context used for validation via `withCredentials(...)`.

---

## Step 5: challenge correlation is mandatory

Never validate a Type 3 message against a newly generated challenge.

The Type 3 response is tied to the exact Type 2 challenge sent to the browser.

Maintain per-client handshake state:

```text
session-id -> { challenge, createdAt, handshakeState }
```

At minimum:

- expire challenge state quickly,
- replace it when a new handshake starts,
- delete it after success or terminal failure,
- never log challenge-response blobs at INFO level,
- never share one challenge across unrelated sessions.

---

## Step 6: Spring Security integration

Do not treat the jcifs credential object as the application authorization model.

After successful AD validation, translate the result into an application principal.

Example conceptual shape:

```java
UsernamePasswordAuthenticationToken authentication =
    UsernamePasswordAuthenticationToken.authenticated(
        principal,
        null,
        authorities
    );

SecurityContextHolder.getContext().setAuthentication(authentication);
```

The application identity should contain stable information such as:

```text
domain
username
canonical principal name
AD SID or directory ID if separately resolved
application roles
```

Authentication answers "who is this?".

Application authorization remains a separate concern.

---

## Step 7: do not use SMB protocol settings as HTTP authentication policy

The jcifs configuration object contains settings for its SMB client stack.

Do not assume that enabling SMB2 or changing SMB dialects upgrades the HTTP authentication mechanism.

For normal SMB resources, prefer an explicit SMB2+ range such as:

```properties
jcifs.smb.client.minVersion=SMB202
jcifs.smb.client.maxVersion=SMB210
```

For the legacy HTTP NTLM compatibility flow, do not force SMB2 and then expect `getChallenge()` to work.

Keep the two concerns separate in code and configuration.

---

## Step 8: browser behavior

Windows Integrated Authentication only works automatically when browser and environment policy permit it.

The server-side filter should only emit standards-compatible NTLM challenge responses. It should not attempt browser-specific credential collection.

Expect deployment configuration outside the Java application to control whether browsers automatically send the logged-in user's Windows credentials.

Do not add an HTML form asking users for their Windows password as a substitute for integrated authentication.

---

## Step 9: security requirements

If the legacy flow is implemented, enforce at least these controls:

- HTTPS only.
- Restrict the feature to the internal trusted network if that is the intended environment.
- Never enable Basic fallback unless it is a separately approved requirement.
- Never set `jcifs.http.insecureBasic=true`.
- Do not log `Authorization` headers.
- Do not log LM/NT response bytes.
- Do not persist challenge-response material.
- Bound handshake state lifetime.
- Rate-limit repeated failed handshakes.
- Clear authentication/session state on failure.
- Treat domain and username from Type 3 as untrusted input until the AD validation succeeds.

---

## Step 10: testing matrix

The offline agent should not call the implementation complete until these cases are tested.

### Handshake

- first request gets `401 + WWW-Authenticate: NTLM`,
- Type 1 produces a Type 2 challenge,
- valid Type 3 authenticates,
- invalid Type 3 returns `401`,
- stale Type 3 is rejected,
- Type 3 from another session is rejected,
- missing challenge state is rejected.

### Identity

- domain is preserved,
- username is preserved,
- authenticated principal is available through Spring Security,
- anonymous requests never reach protected controllers.

### Operational

- domain controller unavailable,
- DNS lookup failure,
- challenge acquisition failure,
- session expiration mid-handshake,
- repeated browser retry loop,
- app restart during handshake.

### Compatibility gate

Also prove, in the target environment, whether NTLMv1 and the required SMB1 challenge behavior are actually allowed.

If not, stop and replace this design. Do not weaken domain policy just to make the application fit jcifs-ng's deprecated HTTP mechanism.

---

# What not to build

An offline agent must not do any of the following.

### Do not register `NtlmHttpFilter` directly in Spring Boot 3

It is `javax.servlet` based and deprecated.

### Do not copy `NtlmHttpFilter` and merely replace imports

That hides the real issue. The problem is not only `javax` vs `jakarta`; the authentication algorithm itself is legacy NTLMv1-only.

### Do not assume `NtlmPasswordAuthenticator` is a drop-in replacement for the five-argument raw-hash constructor

It is the modern credential class for jcifs SMB client authentication. It does not represent a supported continuation of the old HTTP pass-through design.

### Do not claim NTLMv2 support because `Type3Message` contains NTLMv2 code

That code supports jcifs acting as an SMB client with credentials. It does not make jcifs a modern AD-side HTTP NTLM validator.

### Do not force SMB2 and expect `getChallenge()` to work

The historical challenge path depends on SMB1-era server encryption-key behavior.

### Do not add a domain service account password to source control

All credentials must come from the approved runtime secret provider.

---

# Recommended class boundary if the legacy path is approved

Keep compatibility code isolated so it can later be removed.

```text
backend/
  ...
  security/
    ad/
      LegacyNtlmAuthenticationFilter
      NtlmHandshakeState
      NtlmChallengeStore
      JcifsLegacyAdValidator
      AdPrincipal
```

Suggested responsibilities:

```text
LegacyNtlmAuthenticationFilter
    HTTP 401/Authorization handshake only

NtlmChallengeStore
    challenge correlation + expiry

JcifsLegacyAdValidator
    all jcifs-specific DC resolution/challenge/validation code

AdPrincipal
    normalized authenticated application identity
```

No controller should know about NTLM Type 1/2/3 messages.

No business service should depend on jcifs classes.

This creates a clean seam for replacing the mechanism later.

---

# Decision table

| Question | Answer for jcifs-ng 2.1.10 |
| --- | --- |
| Is `jcifs.http.NtlmHttpFilter` present? | Yes |
| Is it deprecated? | Yes |
| Why? | NTLMv1-only legacy HTTP authentication design |
| Does jcifs-ng provide a supported modern server-side NTLMv2 HTTP filter? | No |
| Are `getChallenge()` / `logon()` still present? | Yes |
| Are they supported modern APIs? | No; deprecated and documented as broken |
| Is SMB2 enabled by default? | Yes |
| Is SMB1 also allowed by default? | Yes |
| Canonical SMB dialect keys | `jcifs.smb.client.minVersion`, `jcifs.smb.client.maxVersion` |
| Default protocol range | `SMB1` through `SMB210` |
| Can the old HTTP filter be directly used with Spring Boot 3? | No; it uses `javax.servlet` |
| Can a Jakarta port be written? | Technically yes for legacy NTLMv1 compatibility, but that does not make the design modern or supported |
| Should a modern NTLMv2 AD system be built on this flow? | No |

---

# Source anchors for offline verification

When internet access is later available, verify against tag `jcifs-ng-2.1.10` in `AgNO3/jcifs-ng`:

```text
src/main/java/jcifs/http/NtlmHttpFilter.java
src/main/java/jcifs/http/NtlmSsp.java
src/main/java/jcifs/http/NtlmServlet.java
src/main/java/jcifs/SmbTransportPool.java
src/main/java/jcifs/smb/SmbTransportPoolImpl.java
src/main/java/jcifs/smb/SmbTransportImpl.java
src/main/java/jcifs/smb/NtlmPasswordAuthentication.java
src/main/java/jcifs/smb/NtlmPasswordAuthenticator.java
src/main/java/jcifs/ntlmssp/Type2Message.java
src/main/java/jcifs/ntlmssp/Type3Message.java
src/main/java/jcifs/config/PropertyConfiguration.java
src/main/java/jcifs/config/BaseConfiguration.java
README.md
pom.xml
```

These files are the source of truth for every important claim in this document.

---

# Final instruction to an offline coding agent

Do not reason from "the method exists" to "the project supports this architecture".

For jcifs-ng 2.1.10, the server-side HTTP NTLM classes are retained legacy code. The project itself marks the key path deprecated and broken.

If the target environment explicitly requires old NTLMv1 compatibility, isolate the implementation behind a Jakarta/Spring Security adapter and reproduce only the minimum historical challenge-response flow.

If the target environment requires modern NTLMv2 or production-grade Windows Integrated Authentication, stop and choose a real AD authentication mechanism instead of extending jcifs-ng's deprecated HTTP path.
