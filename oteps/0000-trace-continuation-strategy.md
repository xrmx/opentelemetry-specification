# Trace Continuation Strategy

## Motivation

OpenTelemetry currently defaults to continuing remote parent context when
extracting and starting server spans. This behavior works well inside a single
trust domain, but it is insufficient for common boundary scenarios discussed in
issue #1633.

When context is continued unchanged across trust boundaries, users report the
following problems:

- Multiple organizations share a trace ID even though they cannot share full
  trace data.
- Callee services inherit upstream sampling decisions that may be unsafe or
  undesirable for local policies.
- Baggage can be propagated to external systems where it may expose sensitive or
  internal information.
- Automatic instrumentation lacks a standard way to apply boundary behavior
  consistently without manual per-call code.

These scenarios are common in practice, for example:

- third-party API calls,
- webhooks,
- synthetic monitoring and edge traffic,
- mixed internal/external API gateway usage,
- local sidecars/proxies where trace headers can cause functional side effects.

Users need a standard strategy to preserve causal relationships without continuing
parentage across boundaries. A restart-with-link approach provides that strategy:
it preserves correlation through links while intentionally breaking parentage
continuation.

Standardizing this behavior in OpenTelemetry gives:

- safer default building blocks for boundary-aware instrumentation,
- a shared cross-language contract between propagation and tracing components,
- better interoperability for zero-code and library instrumentation.

## Explanation

This proposal defines a standard trace continuation strategy for trust and
propagation boundaries. The strategy allows an implementation to restart trace
parentage while preserving causal correlation through links.

At a configured boundary, extraction behavior changes from "continue the remote
parent" to "restart-with-link":

1. The propagator (or instrumentation code that invokes it) extracts the remote
   `SpanContext` from incoming headers.
2. Instead of making that remote context the active parent, it stores the
   extracted remote context in a dedicated context slot for deferred linking.
3. When the tracer starts the server span under boundary restart policy, it
   creates a new root span and adds a link to the stored remote context.

This creates two connected but independent traces:

- parentage continuity is intentionally broken at the boundary,
- correlation continuity is preserved using a span link.

This model addresses common boundary scenarios where direct continuation is
undesired, such as:

- untrusted or cross-organization traffic,
- calls where propagated baggage may contain sensitive data,
- traffic where upstream sampling decisions should not be inherited.

### Conceptual request flow

Normal continuation:

- Extract remote context.
- Start span as child of remote parent.
- Continue same trace ID and parent chain.

Boundary restart with link:

- Extract remote context.
- Store extracted remote context in the dedicated link-candidate context slot.
- Start a new root span.
- Add one link to the stored remote context.

The same behavior can be applied to inbound server instrumentation and other
protocols where extraction and span creation are decoupled.

## Internal details

### Behavioral contract

This proposal introduces a cross-component behavioral contract between
propagation and tracing:

- **Remote-parent-for-link context value**: a standard context semantic meaning
  "an extracted remote parent candidate that is eligible for linking but not
  for parentage."
- Language implementations MAY use language-specific key material, but MUST
  preserve this semantic contract across propagator and tracer
  components.

### Required Behavior

When boundary restart policy applies:

1. Extraction logic MUST parse incoming remote context according to existing
   propagator behavior.
2. Extraction logic MUST store valid extracted remote `SpanContext` as
   remote-parent-for-link in context.
3. Extraction logic MUST NOT make that extracted remote context the active
   parent span for default parentage.
4. Span start logic MUST create a new root span.
5. If remote-parent-for-link is present and valid, span start logic SHOULD add
   exactly one link to the new root span.

When boundary restart policy does not apply:

- Existing continuation behavior remains unchanged.

### Parentage Precedence

To avoid ambiguous behavior, span start logic MUST use the following parentage
order:

1. Explicit API parent override (if provided by the caller).
2. Active in-process span context.
3. Extracted remote parent context (normal continuation path only).
4. Root span if none apply.

The remote-parent-for-link value MUST NOT be used as a parent in this order.

### Interaction with Existing Functionality

- Existing propagation formats are unchanged.
- Existing APIs can remain source-compatible if the new context semantic is
  handled internally by instrumentation and SDK components.
- Linking is already a supported tracing concept, so correlation across the
  boundary is represented without introducing a new data model primitive.

### Error Modes and Handling

- Invalid extracted remote context: ignore it; start new root span without link.
- Missing remote-parent-for-link at span start: start new root span without
  link.
- Duplicate candidates in context: implementation SHOULD keep one deterministic
  candidate (for example, first valid extracted context) and create at most one
  link by default.

### Corner Cases

- If both current in-process span and remote-parent-for-link exist, current
  in-process span continues to follow normal parentage rules; remote candidate
  remains link-only.
- If sampling flags from remote context are present, they MUST NOT force
  parentage continuation under restart policy.
- If baggage handling is separately restricted at the boundary, this proposal
  remains compatible: trace restart-with-link does not require baggage
  propagation.

## Trade-offs and mitigations

### Trade-off: Trace tree continuity is broken at boundaries

Restarting at a boundary creates a new root span, so parent/child continuity is
no longer represented as a single tree in one trace.

Mitigation:

- Preserve causal relationship using a span link to the extracted remote
  context.
- Document this as intentional behavior and provide guidance for backend query
  patterns that use links.

### Trade-off: Increased implementation complexity

The proposal introduces additional boundary decision points and context
semantics that must be coordinated across SDK components.

Mitigation:

- Keep the normative contract small: one link-only context semantic and one
  restart-with-link behavior.
- Reuse existing span links and existing propagation parsing logic.
- Allow incremental adoption by making restart policy configurable and scoped.

### Trade-off: Potential cross-language divergence

Without clear behavioral requirements, language implementations could differ in
how they store link candidates, select parent precedence, or handle invalid
context.

Mitigation:

- Define language-agnostic MUST/SHOULD behavior for extraction, parentage
  precedence, and link creation.
- Keep key material language-specific, but keep semantic meaning standardized.
- Add conformance-style test scenarios as follow-up spec work.

### Trade-off: Policy misconfiguration can over-fragment traces

If boundary rules are too broad, users may unintentionally restart too many
requests and reduce end-to-end trace readability.

Mitigation:

- Recommend conservative defaults that preserve current continuation unless
  restart is explicitly configured.
- Encourage implementations to expose diagnostics (for example counts of
  continue vs restart decisions).
- Encourage staged rollout (dry-run/observe, then enforce).

### Trade-off: Relationship to baggage controls can be confusing

Users often want both trace restart and baggage suppression/filtering. If these
are not clearly separated, implementations may couple them inconsistently.

Mitigation:

- Specify that restart-with-link is independent of baggage behavior.
- Permit composition with baggage policies, but avoid requiring fine-grained
  baggage filtering in this OTEP.
- Leave advanced baggage filtering standardization to follow-up work.

### Trade-off: Security expectations may be overstated

Restarting trace parentage reduces correlation continuity but does not by itself
eliminate all metadata exposure risks.

Mitigation:

- Explicitly state that this proposal is not a full data-loss-prevention
  mechanism.
- Recommend combining boundary restart policy with network controls and baggage
  governance where required.

## Prior art and alternatives

### Prior art

The closest existing OpenTelemetry prior art is in
`opentelemetry-go-contrib` `otelhttp` via
[`WithPublicEndpointFn`](https://github.com/open-telemetry/opentelemetry-go-contrib/blob/instrumentation/net/http/otelhttp/v0.65.0/instrumentation/net/http/otelhttp/config.go#L99),
as used in
[handler logic](https://github.com/open-telemetry/opentelemetry-go-contrib/blob/dbf7b0a8a37a70ea1848bfdee02ff6c68b0fa9d6/instrumentation/net/http/otelhttp/handler.go#L103).
In that model, user-supplied logic decides when to start a new trace and link
to the extracted remote context.

Issue #1633 and related implementation issues across language ecosystems provide
additional prior art showing repeated demand for:

- boundary-aware propagation controls,
- restart-with-link semantics,
- automated instrumentation support for policy-driven behavior.

### Alternatives considered

1. Keep the status quo (always continue by default everywhere).
   - Rejected as insufficient for security, privacy, and policy-controlled
     boundary scenarios described by users.

2. Handle boundaries outside OpenTelemetry only (for example DMZ/proxy-only
   sanitization).
   - Useful as defense-in-depth, but rejected as the only solution because users
     still need in-process and instrumentation-level control where proxy-only
     approaches are impractical.

3. Modify W3C Trace Context semantics or header format in this proposal.
   - Rejected for this OTEP scope. This proposal standardizes OpenTelemetry
     behavior while remaining compatible with current propagation formats.

5. Drop context without preserving causal relation.
   - Rejected as a general strategy because it loses useful troubleshooting
     correlation. Restart-with-link preserves causality while breaking parentage.

## Open questions

1. Where should boundary policy configuration be standardized first?
   - This OTEP defines behavior, but does not yet fix a canonical configuration
     schema location (core spec vs dedicated configuration schema work).

2. Should this boundary policy configuration delegate to custom Propagators or
   should we instead extend them using a Rule based approach?

3. Should restart-with-link be complemented by standardized selective baggage
   filtering in this OTEP, or left for a follow-up OTEP?
   - Current proposal is compatible with baggage controls, but does not define
     fine-grained baggage policy semantics.

4. What observability requirements should be imposed on implementations for
   policy evaluation outcomes?
   - For example, whether implementations SHOULD emit counters/logs for
     "continued", "restarted-with-link", and "suppressed" decisions.

5. How should inbound route-based and outbound destination-based policy matching
   be made comparable across language ecosystems?
   - A minimal common model may be needed to prevent divergence while still
     allowing language-specific implementation details.

## Prototypes

<!--
Link to any prototypes or proof-of-concept implementations that you have created.
This may include code, design documents, or anything else that demonstrates the
feasibility of your proposal.

Depending on the scope of the change, prototyping in multiple programming
languages might be required.
-->

## Future possibilities

<!-- What are some future changes that this proposal would enable? -->
