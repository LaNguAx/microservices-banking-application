# NTLMv2 Browser SSO Against Active Directory — Architecture Options and Implementation Guidance

This document is a companion to [`jcifs-ng-ad-authentication.md`](jcifs-ng-ad-authentication.md).

The earlier document establishes an important negative result: **jcifs-ng 2.1.10 must not be used as the modern server-side validator for browser NTLMv2 through its deprecated `NtlmHttpFilter` / SMB challenge path**.

This document records the next stage of the investigation.

We now understand a working pattern used by `express-ntlm`: relay the browser's NTLM Type 1 / Type 3 tokens into an Active Directory LDAP SASL `GSS-SPNEGO` bind, allowing AD to generate the Type 2 challenge and validate the final NTLM response.

However, **that pattern is evidence that the requirement is technically possible; it is not automatically the architecture we should copy into Spring Boot**.

The coding agent must first evaluate whether Java/Spring and the target runtime have a better, more supportable way to let Active Directory validate the browser's NTLMv2 exchange.

> Target requirement: Spring Boot 3 / Jakarta Servlet, raw browser-side HTTP NTLM, Active Directory validation, NTLMv2, SMB1 disabled, no requirement to use browser-side Kerberos or `WWW-Authenticate: Negotiate`.

---

# 1. Executive decision

Do **not** begin by porting `express-ntlm` line-for-line.

Design the Spring application around a validator abstraction:

```text
Browser
   |
   | HTTP NTLM Type1 / Type3
   v
Spring Security / NtlmAuthenticationFilter
   |
   v
NtlmValidator
   |
   +--> Windows SSPI / WAFFLE                if running on Windows
   |
   +--> Samba winbind + ntlm_auth            strong open-source Linux option
   |
   +--> AD LDAP SASL relay                   possible pure-Java fallback/prototype
   |
   +--> dedicated commercial NTLM provider   if policy allows
```

The key architectural rule is:

> **Spring owns HTTP/security integration; a replaceable backend validator owns the NTLMSSP conversation with Active Directory.**

Do not couple controllers or Spring Security directly to jcifs, UnboundID, Samba, or any one authentication backend.

A conceptual interface is:

```java
public interface NtlmValidator {

    NtlmChallenge begin(byte[] type1Token);

    NtlmIdentity authenticate(
        NtlmHandshakeHandle handle,
        byte[] type3Token
    );

    void abort(NtlmHandshakeHandle handle);
}
```

The application should be able to replace the validation strategy without rewriting the HTTP NTLM filter.

---

# 2. What we learned from `express-ntlm`

`express-ntlm` is useful because it proves an important point:

```text
Browser NTLMv2 can be validated by AD
without the application knowing the user's password
and without jcifs' SMB1 challenge trick.
```

The relevant implementation is:

```text
https://github.com/einfallstoll/express-ntlm
```

Relevant files:

```text
lib/express-ntlm.js
lib/NTLM_AD_Proxy.js
lib/NTLM_Proxy.js
```

At the inspected commit:

```text
66516470e9e19aa7649d5fe53518431a7a896241
```

## 2.1 The pattern

The browser-facing side is ordinary HTTP NTLM:

```http
WWW-Authenticate: NTLM
Authorization: NTLM <base64 NTLMSSP token>
```

`express-ntlm` then opens an LDAP connection to the domain controller and sends the browser's raw NTLM token as the SASL credentials of an LDAPv3 bind using:

```text
mechanism = GSS-SPNEGO
credentials = raw NTLM token
```

Conceptually:

```text
LDAP BindRequest
  version = 3
  name = ""
  authentication = SASL {
      mechanism = "GSS-SPNEGO"
      credentials = browser NTLM token
  }
```

AD responds to the first stage with:

```text
LDAP result code 14 = saslBindInProgress
serverSaslCreds = NTLM Type2 challenge
```

The application forwards that Type 2 token back to the browser.

The browser generates Type 3 / NTLMv2 using that challenge.

The application then sends the browser's Type 3 token through a second SASL bind on the **same LDAP connection**.

If AD returns success, the NTLM authentication has been validated.

## 2.2 Why this matters

This explains how a Node implementation can authenticate NTLMv2 even though jcifs-ng's old HTTP filter cannot.

The Node implementation does **not** do this:

```text
jcifs getChallenge()
    -> SMB1-era challenge
    -> application-created Type2
    -> jcifs SMB logon
```

Instead it does this:

```text
Browser Type1
    -> AD through LDAP SASL
    -> AD-created Type2
    -> Browser Type3
    -> AD through same LDAP SASL exchange
    -> success/failure
```

That removes the dependency on jcifs' deprecated SMB challenge mechanism.

## 2.3 Important: treat `express-ntlm` as a protocol reference, not a security blueprint

The coding agent must **not** assume that because `express-ntlm` implements a flow, that flow is the best Spring Boot architecture or is permitted by the target AD security policy.

Use it to understand:

- HTTP NTLM state transitions,
- how Type1 and Type3 are relayed,
- why Type2 must come from the validating security context,
- why state/connection affinity matters,
- how AD can validate the final response without the user's password.

Do **not** blindly copy:

- its cache strategy,
- its connection identification strategy,
- its error handling,
- its LDAP transport assumptions,
- its security defaults,
- its process/runtime architecture.

The Spring implementation should be designed independently using current Java, Spring Security, AD, and infrastructure constraints.

---

# 3. The core requirement

The desired browser exchange is:

```text
1. Browser requests protected endpoint

2. Server responds:
   401
   WWW-Authenticate: NTLM

3. Browser sends:
   Authorization: NTLM <Type1>

4. Backend validator obtains a Type2 challenge from a security authority
   capable of validating the final NTLMv2 response.

5. Server responds:
   401
   WWW-Authenticate: NTLM <Type2>

6. Browser sends:
   Authorization: NTLM <Type3>

7. Backend validator validates Type3 against AD.

8. Only after AD-backed validation succeeds:
   Spring creates an authenticated principal.
```

The application never needs the user's password.

The application must never trust the identity fields inside Type3 merely because they parse successfully.

A Type3 token can claim a username. The username becomes trusted **only after the backend validator successfully completes the AD-backed NTLM exchange**.

---

# 4. Constraints already established

For this project, assume:

```text
SMB1 disabled                         YES
NTLMv2 required                       YES
browser sends raw HTTP NTLM           YES
browser-side Kerberos required        NO
browser-side SPNEGO required          NO
Spring Boot 3 / Jakarta               YES
jcifs NtlmHttpFilter usable           NO
jcifs SMB1 getChallenge trick usable  NO
```

Important distinction:

```text
"Browser must not use SPNEGO"
```

is different from:

```text
"SPNEGO may not appear anywhere in the backend infrastructure"
```

The LDAP relay pattern uses raw `NTLM` in HTTP but an LDAP SASL mechanism called `GSS-SPNEGO` on the backend-to-AD leg.

If security policy prohibits even that backend use, the LDAP relay option is eliminated.

---

# 5. Candidate architecture A — Windows SSPI through WAFFLE

If the Spring Boot process runs on Windows, or can run behind a Windows authentication tier, evaluate **WAFFLE first**.

Project:

```text
https://github.com/waffle/waffle
```

WAFFLE is a Java/Windows authentication framework that delegates authentication to Windows security APIs. Its current project documentation lists support for:

```text
NTLM
Kerberos
Negotiate
Spring Boot 2 / 3 / 4
Spring Security 5 / 6 / 7
javax and jakarta integrations
```

The attractive property is that the Java application does not implement NTLM validation cryptography and does not relay credentials to LDAP itself.

Conceptually:

```text
Browser NTLM
     |
     v
Spring / WAFFLE
     |
     v
Windows SSPI
     |
     v
Active Directory
```

## Advantages

- Mature Windows authentication subsystem performs the protocol work.
- AD identity/group information integrates naturally with Windows APIs.
- Avoids implementing an LDAP credential relay.
- Avoids jcifs' deprecated HTTP NTLM stack.
- Current WAFFLE project advertises Spring Boot 3 and Spring Security 6 compatibility.

## Disadvantages

- Native Windows dependency.
- Poor fit for a normal Linux Kubernetes workload unless Windows nodes are intentionally part of the platform.
- Must still confirm that the exact desired challenge scheme is `NTLM`, not only `Negotiate`, in the selected WAFFLE configuration.

## Agent rule

If the production service runs on Windows, **investigate WAFFLE before writing a custom NTLM protocol implementation**.

Do not reimplement a security protocol in Java when the operating system can validate it directly.

---

# 6. Candidate architecture B — Samba winbind + `ntlm_auth`

For a Linux deployment, especially where open-source software is preferred, evaluate Samba's **winbind + `ntlm_auth`** very seriously.

Official Samba documentation:

```text
https://www.samba.org/samba/samba/docs/man/manpages/ntlm_auth.1.html
```

`ntlm_auth` is explicitly designed as a helper for external programs that need NTLM authentication services from Samba/winbind.

Samba documents the helper mode:

```text
--helper-protocol=squid-2.5-ntlmssp
```

as a **server-side NTLMSSP authentication helper**.

It requires access to Samba's privileged winbind communication directory (`winbindd_privileged`).

This is important because it gives the application a real domain-backed NTLM validator without forcing the Spring application to implement NETLOGON itself.

Conceptually:

```text
Browser
   |
   | Type1 / Type3
   v
Spring Boot
   |
   | helper protocol
   v
long-lived ntlm_auth helper
   |
   v
winbindd
   |
   v
Active Directory
```

## 6.1 Why this is architecturally stronger than the LDAP relay

The purpose of `ntlm_auth` is authentication.

The purpose of the LDAP relay pattern is different: the application takes an NTLM credential exchange from one protocol endpoint and forwards it into a different LDAP SASL authentication exchange.

That distinction matters when AD hardening controls are considered.

Using `ntlm_auth`/winbind means:

- Samba acts as the domain member/security component,
- the application asks Samba to perform server-side NTLMSSP validation,
- the application does not manufacture challenges,
- the application does not implement NETLOGON,
- the application does not need to relay browser credentials into LDAP.

## 6.2 Typical helper state machine

The exact helper syntax must be validated against the installed Samba version, but the `squid-2.5-ntlmssp` family is stateful and conceptually behaves like:

```text
Spring -> helper:
YR <base64 Type1>

helper -> Spring:
TT <base64 Type2>

Spring -> browser:
401
WWW-Authenticate: NTLM <base64 Type2>

browser -> Spring:
Authorization: NTLM <base64 Type3>

Spring -> SAME helper conversation:
KK <base64 Type3>

helper -> Spring:
AF / OK   => authenticated
NA / ERR  => invalid credentials
BH        => helper failure
```

Samba's documentation specifically notes that the NTLMSSP Negotiate packet may be included with `YR`, and Squid's helper protocol documentation describes `YR`, `KK`, `TT`, success/failure responses, and the stateful nature of NTLM authentication.

Do not hard-code one historical response vocabulary without testing the actual Samba package used in production. Older integrations commonly use `AF`/`NA`; newer helper conventions may expose `OK`/`ERR` depending on protocol/version.

## 6.3 Do not spawn one process for every HTTP request

A naive implementation like this is wrong:

```java
new ProcessBuilder("ntlm_auth", ...).start();
```

for every request.

The helper protocol is stateful and process creation is expensive.

Prefer:

```text
NtlmAuthHelperPool
    |
    +-- helper process 1
    +-- helper process 2
    +-- helper process 3
```

A handshake leases one helper conversation from Type1 through Type3 and releases it after terminal success/failure.

Do not interleave multiple NTLM handshakes through a helper unless the chosen helper protocol explicitly supports concurrency and correlation. Squid's documentation notes that NTLM helper exchanges are stateful and not generally concurrent within one helper conversation.

## 6.4 Linux/Kubernetes caveat

`ntlm_auth` is not "just another Maven dependency".

A working environment requires Samba/winbind to be configured as a domain member and requires secure access to the privileged winbind interface.

In Kubernetes this needs deliberate infrastructure design.

Questions that must be answered before implementation:

```text
Where does winbindd run?
How is the Linux security component joined to AD?
Where are machine-account/domain-member secrets stored?
How are those secrets rotated?
Does each pod become a domain member?
Is there a dedicated internal authentication service instead?
How does a pod obtain access to winbindd_privileged?
What happens during pod rescheduling?
How is HA handled?
```

Possible topologies include:

```text
Spring pod
   -> local sidecar / colocated Samba component
   -> AD
```

or:

```text
Spring services
   -> dedicated internal NTLM validation service
      backed by Samba/winbind
   -> AD
```

Do **not** casually mount domain-member secret databases into arbitrary application pods.

## 6.5 Recommendation

For Linux, **this is the first open-source production architecture the agent should evaluate before choosing a custom LDAP relay**.

It may require more infrastructure work than a pure-Java prototype, but the authentication responsibility sits in software explicitly built for Windows domain authentication.

---

# 7. Candidate architecture C — Pure Java LDAP SASL relay

The previously documented `express-ntlm` pattern remains technically important and may be usable.

It is especially attractive when the constraints are:

```text
Linux runtime
no Windows SSPI
no Samba/winbind allowed
no commercial library
must stay inside the JVM
```

Architecture:

```text
Browser Type1
      |
      v
Spring Boot
      |
      | LDAP SASL GSS-SPNEGO
      | credentials = Type1
      v
AD
      |
      | saslBindInProgress + Type2
      v
Spring Boot
      |
      | forwards Type2
      v
Browser
      |
      | Type3
      v
Spring Boot
      |
      | SAME LDAP connection
      | SASL GSS-SPNEGO credentials = Type3
      v
AD
      |
      +--> SUCCESS / FAILURE
```

This is **not rejected** as an option.

But it should now be treated as:

> a pure-Java fallback / prototype whose compatibility with the organization's AD hardening policy must be proven before it becomes the production design.

---

# 8. Why the LDAP relay requires security review

This is the largest change from the earlier version of this document.

NTLM credential relay is a known class of Windows security attack.

Microsoft provides LDAP hardening controls specifically intended to reduce or block unsafe relay scenarios, including:

```text
LDAP signing
LDAP channel binding / Channel Binding Tokens (CBT)
```

Microsoft documentation explains that LDAP signing protects LDAP messages from tampering and that channel binding cryptographically binds authentication to the underlying TLS channel, helping prevent man-in-the-middle/session-relay scenarios.

Reference:

```text
https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing
```

Therefore, **do not assume that an `express-ntlm`-style relay will work in a hardened domain**.

And more importantly:

> Do not weaken LDAP signing, channel binding, or other AD protections merely to make the custom relay work unless the organization's security owner explicitly approves that tradeoff.

## 8.1 Why LDAPS alone is not automatically enough

It is tempting to reason:

```text
"We use LDAPS, therefore the relay is secure."
```

That conclusion is too simplistic.

Channel binding exists precisely because TLS encryption by itself does not solve every credential relay problem.

The browser's HTTPS channel and the application's LDAPS channel are separate TLS channels:

```text
Browser ===== HTTPS =====> Spring

Spring  ===== LDAPS =====> Domain Controller
```

If the NTLM authentication includes channel-binding expectations tied to the original browser/service TLS endpoint, relaying that credential material onto a different TLS channel may fail or may violate the intended protection model.

The actual behavior depends on:

- domain controller policy,
- client NTLM behavior,
- Extended Protection / CBT policy,
- LDAP signing requirements,
- LDAPS configuration,
- exact token contents.

The agent must test against the real AD policy instead of assuming compatibility.

---

# 9. Java implementation option for the LDAP relay

If the LDAP relay remains a permitted option after security review, use a low-level LDAP API that exposes generic multi-stage SASL binds.

A reasonable Java building block is UnboundID LDAP SDK:

```xml
<dependency>
    <groupId>com.unboundid</groupId>
    <artifactId>unboundid-ldapsdk</artifactId>
    <version>7.0.5</version>
</dependency>
```

Relevant type:

```java
com.unboundid.ldap.sdk.GenericSASLBindRequest
```

Its documentation explicitly states that:

- the caller supplies the SASL mechanism,
- the caller encodes the credentials,
- the caller interprets the result,
- the caller is responsible for every stage of multi-stage mechanisms.

Reference:

```text
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/GenericSASLBindRequest.html
```

For an intermediate bind response, UnboundID exposes:

```java
SASLBindInProgressException
```

and:

```java
getServerSASLCredentials()
```

Reference:

```text
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/SASLBindInProgressException.html
```

## Important correction to implementation style

Do **not** use Java's normal high-level GSS/Kerberos flow if the requirement is specifically raw browser NTLM.

The relay implementation needs explicit control over the byte tokens exchanged with AD.

Conceptually:

```java
GenericSASLBindRequest first = new GenericSASLBindRequest(
    null,
    "GSS-SPNEGO",
    new ASN1OctetString(type1)
);
```

Then capture AD's `serverSASLCredentials` from the `SASL_BIND_IN_PROGRESS` response and forward them to the browser.

For Type3, perform the next bind stage with the **same LDAP connection**.

Do not run directory searches in between the SASL stages.

The LDAP multi-stage bind is not complete until a final non-`SASL_BIND_IN_PROGRESS` response is received.

---

# 10. Recommended Spring-side architecture

Regardless of which validator wins, the Spring architecture should remain stable.

Suggested package structure:

```text
security/
  ntlm/
    NtlmAuthenticationFilter.java
    NtlmAuthenticationEntryPoint.java
    NtlmValidator.java
    NtlmHandshakeRegistry.java
    NtlmHandshakeHandle.java
    NtlmTokenInspector.java
    NtlmIdentity.java

    validator/
      WaffleNtlmValidator.java            # only if Windows/WAFFLE chosen
      SambaNtlmValidator.java             # only if winbind chosen
      LdapSaslNtlmValidator.java          # only if relay chosen
```

Not all implementations need to exist.

Create the interface and implement only the selected production strategy plus test doubles.

## 10.1 Responsibilities

```text
NtlmAuthenticationFilter
    - HTTP only
    - parse Authorization header
    - issue WWW-Authenticate responses
    - dispatch Type1/Type3 to validator
    - populate Spring Security only after success

NtlmValidator
    - talks to actual NTLM validation authority
    - returns Type2
    - validates Type3
    - owns backend-specific state

NtlmHandshakeRegistry
    - short-lived handshake correlation
    - expiration
    - cleanup

NtlmTokenInspector
    - validate NTLMSSP signature
    - validate message type
    - enforce size limits
    - parse identity metadata only after validation

NtlmIdentity
    - normalized domain/user identity
    - optional SID/groups if resolved separately
```

Spring controllers should never receive raw NTLM blobs.

---

# 11. HTTP filter state machine

Use an explicit state machine.

```text
UNAUTHENTICATED
   |
   | no Authorization
   v
SEND_INITIAL_401
   |
   | browser sends Type1
   v
VALIDATOR_BEGIN
   |
   | backend returns Type2 + handshake handle
   v
WAITING_FOR_TYPE3
   |
   | browser sends Type3
   v
VALIDATOR_AUTHENTICATE
   |
   +--> valid   -> AUTHENTICATED
   |
   +--> invalid -> REJECTED
   |
   +--> outage  -> SERVICE_FAILURE
```

## Initial request

Return:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: NTLM
Cache-Control: no-store
```

## Type1 request

1. Require `Authorization: NTLM <token>`.
2. Strict Base64 decode.
3. Enforce maximum token size.
4. Verify NTLMSSP signature.
5. Verify Type1.
6. Call `validator.begin(type1)`.
7. Receive:
   - Type2 bytes,
   - opaque handshake handle.
8. Correlate the handle with this client exchange.
9. Return:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: NTLM <base64 Type2>
Cache-Control: no-store
```

## Type3 request

1. Locate the outstanding handshake.
2. Verify not expired.
3. Strictly validate token structure/type.
4. Call `validator.authenticate(handle, type3)`.
5. If validation succeeds, create the Spring principal.
6. Destroy backend handshake state.
7. Continue the Spring filter chain.

On invalid credentials:

- destroy handshake state,
- do not authenticate,
- return the configured authentication failure response.

On backend outage:

- distinguish infrastructure failure from invalid credentials,
- avoid misleading security logs,
- fail closed.

---

# 12. Handshake correlation and connection affinity

NTLM is stateful.

A Type3 response is bound to the Type2 challenge previously delivered to that browser.

The backend validator may also require affinity to an exact backend security context.

Examples:

```text
LDAP relay:
    same LDAP connection must continue the SASL exchange

Samba helper:
    same leased helper conversation/process must continue the exchange

SSPI:
    same Windows security context handle must continue the exchange
```

Therefore the application must never do this:

```text
Type1 -> backend A -> Type2
Type3 -> unrelated backend B/new context
```

unless the chosen validation backend explicitly supports externalizing/reconstructing the state.

## Do not correlate by username

The username is not trusted before authentication.

## Do not correlate only by source IP

Many clients can share one NAT/proxy address.

## Do not assume servlet `HttpSession` alone solves everything

The browser/server HTTP connection behavior and reverse proxy behavior must be tested.

A useful design is an opaque, short-lived handshake ID stored server-side, mapped to backend state:

```text
handshakeId -> {
    backendType,
    backendHandle,
    createdAt,
    expiresAt,
    state
}
```

Do not expose backend secrets in a client cookie.

---

# 13. Reverse proxies, load balancers, and Kubernetes

NTLM is unusually sensitive to connection/state handling.

Before declaring the implementation complete, test the entire real path:

```text
Browser
  -> corporate proxy (if any)
  -> ingress / load balancer
  -> Spring instance
  -> NTLM validator backend
  -> AD
```

Questions:

```text
Does the load balancer preserve the needed HTTP behavior?
Can Type1 and Type3 reach different pods?
Where is handshake state stored?
Can backend security context be moved between pods?
Is sticky routing required during the handshake?
What happens on HTTP/2?
Does the ingress alter WWW-Authenticate or Authorization headers?
Does the browser retry on a new TCP connection?
```

If the validator state is process-local, ensure Type1 and Type3 reach the same application instance or route the opaque handshake to the instance that owns it.

Do not put a Java `LDAPConnection`, SSPI native handle, or running `Process` object into Redis.

If cross-pod routing is unavoidable, use an architecture in which the stateful NTLM exchange is owned by a dedicated authentication service and Spring talks to it using an opaque handshake ID.

---

# 14. Security requirements common to all implementations

The agent must enforce these rules regardless of backend choice.

## 14.1 HTTPS only

Never intentionally expose HTTP NTLM authentication over untrusted plaintext HTTP.

## 14.2 Never log raw tokens

Do not log:

```text
Authorization: NTLM ...
Type1 bytes
Type2 bytes
Type3 bytes
NT response
LM response
session keys
```

Even DEBUG logs should avoid raw credential material by default.

## 14.3 Strict input validation

Require:

```text
NTLMSSP\0 signature
message type 1 or 3 in client messages
reasonable token size
valid Base64
valid field offsets/lengths before parsing
```

Reject malformed tokens before invoking the backend validator.

## 14.4 Trust identity only after validation

Do not create:

```java
Authentication authenticated = true
```

until the AD-backed validator reports success.

Type3's username/domain/workstation fields are metadata, not proof.

## 14.5 Short handshake TTL

Handshake state should live for seconds, not hours.

Destroy it after:

```text
success
invalid credentials
malformed Type3
backend failure
client restart/new Type1
expiration
```

## 14.6 Rate limiting

Protect the authentication endpoint from uncontrolled handshake creation.

A malicious client should not be able to create unlimited:

```text
LDAP connections
SSPI contexts
ntlm_auth helpers
handshake objects
```

## 14.7 Disable unsafe fallback

Do not silently add Basic authentication or plaintext AD-password collection merely because NTLM failed.

Any fallback mechanism requires its own approved design.

---

# 15. jcifs-ng's role after this investigation

jcifs-ng may still be useful for parsing NTLMSSP structures:

```java
jcifs.ntlmssp.Type1Message
jcifs.ntlmssp.Type2Message
jcifs.ntlmssp.Type3Message
```

But do not let the presence of these classes pull the design back toward the deprecated HTTP authentication path.

Do **not** use the following for the production NTLMv2 validator:

```java
jcifs.http.NtlmHttpFilter
jcifs.http.NtlmSsp
SmbTransportPool.getChallenge(...)
SmbTransportPool.logon(...)
```

And do not enable SMB1 to resurrect that flow.

SMB1 remains disabled.

If the application separately uses jcifs for file shares, configure SMB2+ independently. That is unrelated to browser authentication.

---

# 16. Decision matrix

Use this as the starting architecture decision.

| Environment / constraint | First option to evaluate | Why |
|---|---|---|
| Spring runs on Windows | WAFFLE / Windows SSPI | Native Windows authentication stack validates NTLM; minimal custom protocol code |
| Linux / VM / Kubernetes; Samba allowed | winbind + `ntlm_auth` | Open-source domain-backed server-side NTLMSSP validator; avoids custom LDAP credential relay |
| Linux; no OS/native auth components; JVM-only required | LDAP SASL relay | Technically viable pattern, but must pass AD signing/CBT/security-policy tests |
| Commercial dependency permitted | Evaluate a dedicated NTLM/AD library | May provide a supported Java implementation without custom protocol plumbing |
| Someone proposes jcifs `NtlmHttpFilter` | Reject | Deprecated NTLMv1-only/SMB1-era path, not modern NTLMv2 validation |

This is intentionally **not** a final declaration that Samba is always best or that LDAP relay is always wrong.

The production answer depends on:

- operating system,
- Kubernetes constraints,
- AD policy,
- security controls,
- operational ownership,
- allowed dependencies.

---

# 17. Recommended implementation sequence for the offline agent

Do not start by writing production filter code.

Follow this sequence.

## Phase 1 — Confirm infrastructure facts

Document:

```text
Spring runtime OS
container/Kubernetes topology
AD/DC Windows Server versions
LDAP signing policy
LDAP channel binding policy
NTLM restriction policy
whether Samba/winbind is allowed
whether Windows nodes are available
whether native libraries are allowed
whether commercial libraries are allowed
browser policy for intranet NTLM
proxy/load-balancer topology
```

If these facts are unknown, mark them as blockers rather than inventing assumptions.

## Phase 2 — Build backend validator spikes, independent of Spring Security

Create tiny proof-of-concepts that accept a captured/live Type1 and continue a real browser handshake.

Evaluate in this order appropriate to the runtime:

### Windows runtime

```text
1. WAFFLE / SSPI
2. alternative only if WAFFLE cannot satisfy raw NTLM requirement
```

### Linux runtime

```text
1. Samba winbind + ntlm_auth
2. LDAP SASL relay
3. dedicated library if allowed
```

Measure:

```text
Does Type1 produce Type2?
Does valid Type3 authenticate?
Does invalid Type3 fail?
Does it work with NTLMv1 disabled?
Does it work with SMB1 disabled?
Does it work under current LDAP signing/CBT policy?
What identity does backend return?
How does it behave on DC failover?
```

## Phase 3 — Choose one validator

Write an ADR or design decision explaining why the chosen validator is selected.

Do not keep multiple unfinished authentication mechanisms enabled in production.

## Phase 4 — Integrate with Spring Security

Only after the validator is proven:

```text
implement NtlmAuthenticationFilter
implement handshake registry
map successful identity into Authentication
add failure handling
add observability
```

## Phase 5 — Test behind real infrastructure

Do not stop at `localhost`.

Test through the real load balancer/ingress and corporate browser policy.

---

# 18. Samba validator implementation guidance

If Samba is selected, isolate process management from Spring Security.

Suggested internal classes:

```text
SambaNtlmValidator
NtlmAuthHelperPool
NtlmAuthHelperProcess
NtlmAuthHelperConversation
SambaHelperProtocolParser
```

Conceptually:

```java
public final class SambaNtlmValidator implements NtlmValidator {

    private final NtlmAuthHelperPool pool;

    @Override
    public NtlmChallenge begin(byte[] type1) {
        NtlmAuthHelperConversation conversation = pool.lease();
        byte[] type2 = conversation.begin(type1);
        return new NtlmChallenge(type2, conversation.handle());
    }

    @Override
    public NtlmIdentity authenticate(
            NtlmHandshakeHandle handle,
            byte[] type3) {

        NtlmAuthHelperConversation conversation = pool.require(handle);

        try {
            return conversation.authenticate(type3);
        }
        finally {
            pool.release(conversation);
        }
    }
}
```

The helper pool must:

```text
bound process count
bound waiting queue
kill/restart broken helpers
apply I/O timeouts
never mix conversations
scrub sensitive buffers where practical
not log helper credential blobs
publish health metrics
```

Do not use `Runtime.exec(String)` with concatenated user data.

Type1/Type3 tokens should be transmitted only through the helper's documented stdin protocol.

---

# 19. LDAP relay implementation guidance

If LDAP relay is selected after policy validation, keep it equally isolated.

Suggested classes:

```text
LdapSaslNtlmValidator
LdapNtlmHandshake
DomainControllerSelector
LdapConnectionFactory
```

Conceptual begin flow:

```java
public NtlmChallenge begin(byte[] type1) throws LDAPException {
    LDAPConnection connection = openConnectionToSelectedDc();

    GenericSASLBindRequest request = new GenericSASLBindRequest(
        null,
        "GSS-SPNEGO",
        new ASN1OctetString(type1)
    );

    try {
        connection.bind(request);
        throw new NtlmAuthenticationException(
            "Unexpected final success during Type1 stage"
        );
    }
    catch (SASLBindInProgressException e) {
        ASN1OctetString serverCreds = e.getServerSASLCredentials();

        if (serverCreds == null) {
            connection.close();
            throw new NtlmAuthenticationException(
                "AD returned no SASL challenge"
            );
        }

        byte[] type2 = serverCreds.getValue();
        validateType2(type2);

        LdapNtlmHandshake handshake =
            registry.register(connection, type2);

        return new NtlmChallenge(
            type2,
            handshake.handle()
        );
    }
}
```

Conceptual Type3 flow:

```java
public NtlmIdentity authenticate(
        NtlmHandshakeHandle handle,
        byte[] type3) throws LDAPException {

    LdapNtlmHandshake handshake = registry.require(handle);
    LDAPConnection connection = handshake.connection();

    try {
        GenericSASLBindRequest request =
            new GenericSASLBindRequest(
                null,
                "GSS-SPNEGO",
                new ASN1OctetString(type3)
            );

        BindResult result = connection.bind(request);

        if (!ResultCode.SUCCESS.equals(result.getResultCode())) {
            throw new BadCredentialsException("NTLM rejected by AD");
        }

        return extractValidatedIdentity(type3);
    }
    finally {
        registry.remove(handle);
        connection.close();
    }
}
```

This is architectural pseudocode.

Compile and test against the actual selected UnboundID version and the real AD environment. Do not copy exception/API assumptions blindly.

---

# 20. Authentication versus authorization

Successful NTLM validation proves an AD identity.

It does not automatically define application permissions.

After validation, normalize identity:

```text
domain
username
canonical DOMAIN\user or UPN
SID if resolved
```

Then apply application authorization separately:

```text
AD group mapping
application database roles
Spring GrantedAuthority mapping
business permissions
```

Do not allow unvalidated Type3 group/user strings to become authorities.

If group lookup is required, perform it **after authentication** using an approved directory lookup mechanism.

---

# 21. Observability

Useful metrics:

```text
ntlm_handshake_started_total
ntlm_handshake_success_total
ntlm_bad_credentials_total
ntlm_handshake_timeout_total
ntlm_validator_error_total
ntlm_active_handshakes
ntlm_handshake_duration_seconds
```

Backend-specific metrics:

```text
Samba:
  helper_processes
  helper_busy
  helper_restart_total
  helper_queue_depth

LDAP relay:
  ldap_ntlm_connections
  ldap_bind_in_progress_total
  ldap_bind_failure_total
  ldap_dc_failover_total

WAFFLE/SSPI:
  native_context_error_total
  provider_failure_total
```

Safe structured logs may contain:

```text
handshake correlation ID
validator type
selected DC/service
result category
elapsed time
validated username AFTER success
```

Never log raw NTLM tokens.

---

# 22. Required test matrix

The implementation is not complete until these scenarios are tested.

## Functional

```text
valid domain user
invalid credentials / forged Type3
disabled user
locked user
expired user if domain policy exposes distinct failure
user from unexpected domain
browser without Authorization header
malformed Base64
non-NTLM Authorization scheme
Type2 sent by client
Type3 without prior Type1
second Type1 replacing an unfinished handshake
expired handshake
```

## Protocol/security

```text
NTLMv1 disabled in AD
SMB1 disabled
valid NTLMv2 authentication succeeds
replay stale Type3 against a new handshake fails
Type3 from handshake A cannot finish handshake B
oversized token rejected
raw tokens absent from logs
```

## Infrastructure

```text
DC unavailable during Type1
DC unavailable during Type3
validator timeout
application instance restart mid-handshake
load balancer sends Type3 to another pod
HTTP/1.1 behavior
HTTP/2 ingress behavior
corporate forward proxy present
multiple simultaneous users behind same NAT
```

## LDAP-relay-specific

```text
LDAP signing required
LDAP channel binding Never / When supported / Always as applicable
LDAP 389 with required signing
LDAPS 636
DC certificate validation
real production-equivalent AD policy
```

Do not weaken policy for the test merely to make it green.

## Samba-specific

```text
winbind healthy
winbind unavailable
machine trust broken
helper crashes mid-handshake
helper times out
helper pool exhausted
DC failover
pod/host restart
```

---

# 23. What the agent must NOT do

Do not:

```text
- enable SMB1
- use jcifs NtlmHttpFilter
- use jcifs getChallenge/logon as NTLMv2 validation
- generate an arbitrary Type2 challenge and then hope AD validates Type3 later
- LDAP simple-bind with a username extracted from Type3
- ask the browser/user for the Windows password as a hidden workaround
- trust Type3 username before backend validation
- spawn unbounded ntlm_auth processes
- retain LDAP/helper/SSPI handshakes indefinitely
- store raw NTLM tokens in logs, Redis, database, traces, or metrics
- disable LDAP signing/channel binding without explicit security approval
- assume express-ntlm is automatically production-safe because it works
- assume a localhost proof proves ingress/Kubernetes compatibility
```

---

# 24. Practical recommendation at this point

The investigation has reached this conclusion:

```text
We understand HOW NTLMv2 can be validated against AD without jcifs SMB1.
```

But we have **not** yet proven that the best implementation is the exact `express-ntlm` LDAP relay.

For the coding agent, the next decision should be:

```text
IF Windows runtime:
    evaluate WAFFLE / SSPI first

ELSE IF Linux and Samba/winbind is operationally acceptable:
    evaluate ntlm_auth + winbind first

ELSE:
    test the pure-Java LDAP SASL relay against the real AD hardening policy

THEN:
    choose one validator
    integrate it behind NtlmValidator
    keep Spring HTTP/security code independent of the validator
```

This is the central design direction.

The `express-ntlm` implementation remains extremely useful as a **reference demonstrating the handshake pattern**.

It should not be treated as the specification for our final Java implementation.

---

# 25. Source anchors for the offline agent

## Existing project reference

```text
jcifs-ng-ad-authentication.md
```

Use it for the exact jcifs-ng 2.1.10 limitations and why the old SMB challenge path is rejected.

## express-ntlm reference

```text
https://github.com/einfallstoll/express-ntlm/blob/66516470e9e19aa7649d5fe53518431a7a896241/lib/NTLM_AD_Proxy.js
https://github.com/einfallstoll/express-ntlm/blob/66516470e9e19aa7649d5fe53518431a7a896241/lib/NTLM_Proxy.js
https://github.com/einfallstoll/express-ntlm/blob/66516470e9e19aa7649d5fe53518431a7a896241/lib/express-ntlm.js
```

Use these to understand the protocol relay, not as code to port blindly.

## Samba

```text
https://www.samba.org/samba/samba/docs/man/manpages/ntlm_auth.1.html
https://wiki.squid-cache.org/Features/AddonHelpers
```

Key concepts:

```text
ntlm_auth
winbind
--helper-protocol=squid-2.5-ntlmssp
server-side NTLMSSP helper
stateful helper conversation
```

## WAFFLE

```text
https://github.com/waffle/waffle
```

Key concepts:

```text
Windows SSPI/native authentication
NTLM support
Spring Boot 3 support
Spring Security 6 support
```

## Microsoft LDAP security

```text
https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing
https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-signing-in-windows-server
```

Key concepts:

```text
LDAP signing
LDAP channel binding
CBT
NTLM relay protection
```

## UnboundID generic SASL

```text
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/GenericSASLBindRequest.html
https://docs.ldap.com/ldap-sdk/docs/javadoc/com/unboundid/ldap/sdk/SASLBindInProgressException.html
```

---

# Final directive to the coding agent

Do not interpret this document as "implement express-ntlm in Java."

Interpret it as:

> **We now know the protocol pattern and we know AD can validate NTLMv2 without jcifs' SMB1 mechanism. Before writing production code, choose the safest and most supportable validation backend for the actual runtime. Keep that backend behind a small `NtlmValidator` abstraction. Prefer native/domain authentication components (SSPI on Windows, Samba/winbind on Linux) when they fit the environment. Use the LDAP SASL relay only after proving it is compatible with the organization's LDAP signing/channel-binding policy and after security review.**

The implementation goal is not merely "make NTLM work."

The goal is to make NTLMv2 work **without weakening AD security controls, without depending on SMB1, and without locking Spring Security to one fragile protocol trick**.
