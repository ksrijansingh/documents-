# documents-
OSIF Framework
Architecture, Implementation & Developer Adoption Guide
A practical guide for onboarding developers to the common libraries, connectors, MCP,
and observability shown in the OSIF workspace.
Document details Value
Audience New developers, connector authors, application teams, reviewers, platform engineers
Workspace observed osif-api-template, osif-common-library, osif-common-logger, HTTP/Kafka/Salesforce/SAP
connectors, common error service, MCP, observability, orders API
Document type Architecture and implementation guide
Version 1.0 — baseline guide, to be verified against repository source
Important scope note: This guide is based on the repository names visible in the supplied workspace screenshot. The
screenshot does not expose source code, package APIs, configuration keys, or runtime behavior. Examples below are
reference patterns, not claims about exact existing symbols. Before copying a code sample into the project, verify it
against the actual package exports, pyproject.toml files, and tests.
OSIF Framework — Architecture & Developer Guide Page 1
Contents
1. Executive summary and intended outcomes
2. Workspace map and responsibility boundaries
3. Recommended runtime architecture and request flow
4. Common logger
5. Common library
6. HTTP connector
7. Salesforce connector
8. MCP: tool and connector integration
9. Observability
10. Error handling and resilience
11. Configuration, secrets, and environment management
12. Build, local development, and release workflow
13. Testing strategy and quality gates
14. Security and production readiness
15. New developer onboarding: first working integration
16. Troubleshooting guide
17. Architecture decisions and change-minimisation rules
18. Repository audit checklist and documentation gaps
OSIF Framework — Architecture & Developer Guide Page 2
1. Executive summary and intended outcomes
OSIF appears to be a modular integration framework: shared capabilities live in common packages,
protocol/vendor-specific behavior lives in connectors, MCP provides a tool-oriented integration surface, and
observability provides telemetry. The central architectural goal should be to let application developers compose these
parts without re-implementing logging, error mapping, HTTP behavior, authentication plumbing, retries, timeouts, or
telemetry in every service.
A production-grade framework should make the normal path easy and the unsafe path difficult. Common packages
should define consistent contracts; connectors should implement vendor/protocol details behind those contracts;
application APIs should orchestrate business use cases rather than duplicate infrastructure logic.
What a new developer should be able to do
● Understand which package owns each responsibility and where a change belongs.
● Install and run a sample API locally using the repository's supported workflow.
● Call an HTTP endpoint through the HTTP connector and understand request, response, timeout, and error behavior.
● Configure Salesforce authentication and execute a simple operation without embedding credentials in code.
● Expose a connector operation through MCP only when the tool contract and security rules permit it.
● Read logs, metrics, and traces to diagnose a failed request.
● Add tests and ship a change without breaking existing connectors or applications.
Core principles
● One responsibility per package: shared contracts belong in common libraries; vendor-specific API details belong
in connectors.
● Stable public contracts: consumers should depend on documented interfaces, not internal modules.
● Explicit configuration: timeouts, retry policies, endpoints, authentication, and telemetry exporters should be
configurable and validated.
● Safe defaults: bounded timeouts, no secret logging, controlled retries, and clear error translation.
● Backward compatibility: additive changes first; breaking changes require versioning, migration guidance, and
tests.
● Observable by design: logs, metrics, and traces should share correlation identifiers without exposing sensitive
payloads.
OSIF Framework — Architecture & Developer Guide Page 3
2. Workspace map and responsibility boundaries
Workspace folder Intended responsibility Typical owner / change
osif-api-template Starter structure and conventions for new
API services.
API skeleton, settings, health endpoints,
sample tests, local run instructions.
osif-common-library Shared abstractions, contracts, configuration
helpers, error models, and reusable utilities.
Cross-cutting behavior used by multiple
apps/connectors; avoid vendor-specific
code.
osif-common-logger Consistent structured logging and
correlation context.
Log format, redaction, log levels,
request context integration.
osif-connector-common-errorser
vice
Shared error translation or error-service
integration.
Canonical error envelope, mapping, safe
messages, downstream error
normalization.
osif-connector-http HTTP client operations and protocol-level
concerns.
Request construction, authentication
hooks, timeout, status mapping,
response parsing.
osif-connector-kafka Kafka messaging integration. Producer/consumer behavior,
serialization, delivery and consumer
error handling.
osif-connector-salesforce Salesforce API integration. OAuth/token lifecycle, Salesforce
endpoints, query/write operations, API
errors.
osif-connector-sap SAP integration. SAP-specific authentication, endpoint
conventions, data mapping, vendor
errors.
osif-mcp MCP server/tool definitions and dispatch to
approved capabilities.
Tool schemas, input validation,
invocation policy, tool result/error
mapping.
osif-observability Metrics, traces, health signals, and
instrumentation integration.
Instrumentation hooks, exporters,
dashboards, telemetry configuration.
osif-orders-api Example/business API consuming the
framework.
Business routes, orchestration, domain
validation, dependency wiring.
These are responsibility boundaries inferred from folder names. Confirm actual exports and behavior in each package's
README, pyproject.toml, source tree, tests, and dependency graph before treating this table as the authoritative
contract.
3. Recommended runtime architecture and request flow
The intended dependency direction should point from business applications toward stable shared contracts, then
toward concrete connector implementations. Common packages should not import a business API or depend on a
concrete connector unless a documented plugin mechanism requires it.
Client
|
v
API route / application service (e.g., osif-orders-api)
|-- request validation and business rules
|-- correlation/request context
v
Common contracts / configuration / error model
|
+--> HTTP connector ------> External REST API
+--> Salesforce connector -> Salesforce APIs
OSIF Framework — Architecture & Developer Guide Page 4
+--> Kafka connector ------> Kafka broker
+--> SAP connector --------> SAP system
|
+--> Common logger: structured logs
+--> Observability: metrics, traces, health
+--> MCP adapter: validated tool call -> approved service/connector
Typical request lifecycle
1 The API receives a request and validates its schema and authentication/authorization.
2 A request or correlation ID is created or propagated; the ID is added to log and trace context.
3 The application service calls a stable connector interface with explicit operation parameters.
4 The connector validates configuration, applies authentication, and sends the outbound request with a finite timeout.
5 A retry policy, if enabled, retries only eligible transient failures and only when the operation is safe to retry.
6 The connector parses the response and translates transport/vendor errors into the framework's canonical error
contract.
7 Logs and telemetry record outcome, duration, retry count, and dependency name without recording secrets or
sensitive bodies.
8 The API maps the canonical result/error into the public API response and returns it to the caller.
Dependency rules
● A connector may depend on common contracts, logger abstractions, and observability hooks.
● Common libraries should avoid depending on concrete connector packages.
● Business APIs should not reimplement connector authentication or transport retry loops.
● MCP should dispatch to approved operations; it should not bypass connector validation or authorization.
● Instrumentation must not change business results. Telemetry exporter failures should not normally take down
business traffic.
4. Common logger
Why it exists
Without a shared logger, each service tends to invent different formats, correlation fields, exception formatting, and
redaction behavior. That makes incidents harder to investigate and encourages accidental logging of credentials or
customer data. A common logger should standardize the shape and safety of logs while allowing the application to
choose log level and deployment destination.
Recommended capabilities
● Structured JSON logs with stable fields such as timestamp, severity, service, environment, message, correlation_id,
trace_id, span_id, operation, and duration_ms where relevant.
● Consistent log-level configuration through environment-specific settings.
● Context propagation across async FastAPI requests and outbound connector calls.
● Central redaction of known sensitive keys (authorization, access_token, refresh_token, client_secret, password, API
keys).
● Exception logging that preserves stack traces internally but returns safe, non-sensitive messages to callers.
● A documented integration point for standard Python logging so third-party libraries can be routed consistently.
OSIF Framework — Architecture & Developer Guide Page 5
How an application should use it
# Illustrative pattern only — replace imports with the actual package API.
from fastapi import FastAPI
import logging
app = FastAPI()
logger = logging.getLogger(__name__)
@app.get("/health")
async def health():
logger.info("health_check_succeeded", extra={"operation": "health"})
return {"status": "ok"}
Prefer the framework's documented logger factory or configuration entry point if one exists. Do not configure multiple
root handlers in every module; initialize logging once at application startup. Avoid f-strings that eagerly serialize large
or sensitive objects, and never log access tokens, passwords, full authorization headers, or unredacted
request/response bodies.
Logger acceptance criteria
● A single request can be followed across API and connector logs using a correlation ID.
● Log output is valid structured JSON in the configured production mode.
● Secrets and configured sensitive fields are redacted in normal and exception paths.
● Logger initialization is idempotent and does not duplicate handlers on reload.
● Logging failures do not expose secrets or cause request processing to fail.
5. Common library
Why it exists
The common library is the framework's shared contract layer. It should reduce repeated boilerplate and provide
consistent configuration, models, exceptions, interfaces, and utilities. It should not become a dumping ground for
unrelated helpers or absorb all business logic.
Good candidates for the common library
● Canonical request/response and error models shared across components.
● Typed configuration models and validation helpers, where they are genuinely shared.
● Connector interfaces/protocols and common result types.
● Correlation-context helpers and shared constants.
● Small, deterministic utilities with multiple consumers.
● Compatibility adapters and common exception base classes.
What should not go into it
● Salesforce object-specific business rules, SAP-specific mappings, or HTTP endpoint-specific logic.
● Application-specific business workflows such as order placement.
● Unbounded generic utility modules with unclear ownership.
● Hidden network calls during import or global mutable client instances.
● A second retry/timeout implementation if a dedicated resilience package already owns that behavior.
OSIF Framework — Architecture & Developer Guide Page 6
Design guidance
Keep public APIs small, typed, and documented. Prefer dependency injection over module-level singletons. Validate
configuration early at startup. Use semantic versioning or the organization's equivalent release policy, and test
downstream compatibility before releasing a shared package.
Example connector contract
# Reference shape, not an existing OSIF symbol.
from typing import Protocol, Any
class Connector(Protocol):
async def execute(self, operation: str, payload: dict[str, Any]) -> Any:
"""Execute a documented operation and return a typed result."""
If the repository already has a connector base class, ToolDefinition, or result model, extend and document that
contract instead of introducing a competing abstraction.
6. HTTP connector
Why it exists
The HTTP connector centralizes outbound HTTP concerns so every service does not have to independently
implement URL construction, headers, authentication hooks, timeout handling, response parsing, status-code
mapping, and connection lifecycle management.
Expected responsibilities
Concern Production-grade behavior
Configuration Base URL, allowed endpoint/path, timeout, TLS verification, headers, auth reference, and pool
limits are externalized and validated.
Client lifecycle Reuse an async client/connection pool; close it during application shutdown. Avoid creating a
new client for every request.
Timeouts Set finite connect/read/write/pool timeouts. Distinguish timeout errors from other transport errors.
Status handling Handle successful, redirect, client-error, and server-error responses consistently; do not assume
every response is JSON.
Retries Retry only configured transient failures/statuses, within a total deadline; respect Retry-After
where appropriate.
Security TLS verification enabled by default; prevent SSRF via endpoint allow-lists or strict URL policy
when user input can influence destinations.
Observability Record dependency, operation, duration, status family, timeout/retry count, and trace context.
Response size Apply limits or streaming for large responses; avoid unbounded body buffering.
Illustrative use from an application
# Illustrative pseudocode; adapt to the actual OSIF connector API.
result = await http_connector.get(
path="/v1/customers/123",
timeout=5.0,
headers={"Accept": "application/json"},
)
Prefer named operations and typed inputs over passing arbitrary URLs from untrusted callers. If the connector
supports generic methods, validate HTTP methods, destination hosts, path traversal, redirects, and request headers.
Ensure idempotency semantics are documented for POST/PATCH operations.
OSIF Framework — Architecture & Developer Guide Page 7
HTTP connector checklist
● Unit tests cover 2xx, 4xx, 5xx, invalid JSON, empty body, connection failure, timeout, and cancellation.
● Tests verify timeout and retry configuration are honored.
● A retry cannot multiply beyond the configured attempt/deadline budget.
● Response errors are normalized without leaking remote payloads or credentials.
● Client sessions close cleanly at application shutdown.
7. Salesforce connector
Why it exists
Salesforce has vendor-specific authentication, token lifecycle, API versioning, query conventions, pagination, rate
limits, and error payloads. A dedicated connector isolates those details from business APIs and keeps
Salesforce-specific behavior out of the common library.
Responsibilities to document and implement
● Authentication flow supported by the organization (for example OAuth client credentials or JWT bearer flow); do not
assume one flow until the current implementation is inspected.
● Secure token acquisition, caching, expiry handling, refresh/re-authentication, and concurrent token refresh behavior.
● Instance/base URL resolution and API version configuration.
● Typed operations for query, create, update, delete, and bulk operations if supported.
● Pagination handling for query results and explicit limits for bulk data.
● Salesforce error parsing, including authentication errors, validation failures, limits, and transient service errors.
● Rate-limit awareness and safe backoff; avoid retrying non-idempotent writes without an idempotency strategy.
● Redaction of access tokens, client secrets, session IDs, and sensitive Salesforce record data.
Recommended call flow
1 Load validated configuration and retrieve secrets from the approved secret manager/environment injection
mechanism.
2 Acquire a valid token; refresh safely when expired or rejected, avoiding refresh stampedes.
3 Build the request using a configured API version and validated operation.
4 Send with bounded timeouts and trace/correlation context.
5 Parse success/error responses into typed results or canonical errors.
6 Emit safe logs and telemetry: operation, status family, duration, retry count, and limit signals.
Salesforce-specific pitfalls
● Do not hardcode request timeouts, API versions, instance URLs, or credentials in connector source.
● Do not assume a 200 response means the operation is semantically successful; validate the response structure.
● Do not log SOQL query parameters or record bodies if they may contain sensitive data.
● Do not retry authentication failures indefinitely; use a bounded re-authentication path.
● Handle pagination and API limits explicitly rather than silently returning partial results.
For the current OSIF Salesforce connector, perform a targeted audit for hardcoded timeout values, token caching,
pagination, API version configuration, and error normalization. Move behavior to configuration only where it is genuinely a
OSIF Framework — Architecture & Developer Guide Page 8
deployment concern, and preserve existing defaults to avoid unnecessary breaking changes.
8. MCP: tool and connector integration
Purpose
MCP (Model Context Protocol) can expose approved application capabilities as tools with machine-readable names,
descriptions, input schemas, and results. In OSIF, MCP should be an adapter layer—not a replacement for
connectors, authorization, validation, or business services.
Recommended separation
Layer Responsibility
Tool definition Stable tool name, description, input schema, and expected output contract.
Validation Validate types, required fields, nested objects, bounds, enums, and unknown fields according
to policy.
Authorization Check caller identity, tenant, permissions, and whether this tool/operation is allowed.
Dispatcher Resolve tool name to a registered handler; reject unknown or disabled tools.
Application service Apply business rules and orchestrate operations.
Connector Perform vendor/protocol-specific I/O with authentication, timeout, error mapping, and
telemetry.
Result mapper Return a bounded, structured result or safe error to the MCP client.
Tool design rules
● Use explicit, stable tool names and concise descriptions that state side effects.
● Define complete schemas, including nested objects, arrays, required fields, formats, bounds, and enum values.
● Validate inputs at runtime even when the schema is advertised to the client.
● Use allow-listed handlers rather than dynamically importing arbitrary modules or evaluating tool-provided code.
● Separate read-only tools from write/destructive tools; require stronger authorization and, where appropriate,
confirmation for high-impact actions.
● Use bounded output sizes and redact sensitive fields in results.
● Map exceptions to safe, structured tool errors; do not return tracebacks, tokens, or internal configuration.
Illustrative tool definition
# Reference only. Adapt to the actual ToolDefinition type and MCP SDK version.
tool = {
"name": "get_customer",
"description": "Retrieve a customer by an approved identifier.",
"inputSchema": {
"type": "object",
"properties": {"customer_id": {"type": "string", "minLength": 1}},
"required": ["customer_id"],
"additionalProperties": False,
},
}
OSIF Framework — Architecture & Developer Guide Page 9
MCP request flow
MCP client
-> transport / MCP server
-> tool-name lookup
-> schema + runtime validation
-> authentication / authorization
-> application handler
-> OSIF connector
-> normalized result/error
-> bounded MCP response
Do not let MCP invoke connector internals directly if that bypasses application authorization or business rules. Use the
same service layer used by normal API routes wherever possible.
9. Observability
Why it exists
Logs explain discrete events, metrics show trends and saturation, and traces show the path and timing of a request
across services. Observability should let an engineer answer: what failed, for whom or which operation, where, how
often, and what changed—without inspecting sensitive payloads.
Recommended signals
Signal Useful fields / measures Avoid
Logs service, environment, severity, correlation_id, trace_id,
operation, dependency, outcome, error_code
Tokens, passwords, full payloads,
sensitive record fields
Metrics request count, error count, latency histogram, active
requests, connector duration, retry count, timeout count
High-cardinality labels such as customer
ID, email, raw URL, or request ID
Traces API span, application span, outbound dependency span,
status, duration, trace context
Sensitive attributes and unbounded
request/response bodies
Health Liveness, readiness, dependency status where
appropriate
Expensive checks on every liveness probe
Implementation guidance
● Instrument API boundaries and connector calls consistently; use standard OpenTelemetry conventions if compatible
with the stack.
● Propagate trace context across async boundaries and outbound HTTP calls.
● Keep metric labels low-cardinality. Put unique IDs in logs/traces, not metric dimensions.
● Expose liveness separately from readiness. Readiness can check whether the service can accept traffic; liveness
should not restart a process just because a remote dependency is temporarily down.
● Treat telemetry exporters as non-critical dependencies: buffer or drop telemetry safely according to policy rather
than blocking business requests indefinitely.
● Document exporter endpoints, sampling, retention, dashboards, and alert thresholds per environment.
Minimum operational dashboard
● Request rate, error rate, and p50/p95/p99 latency by route or operation.
● Outbound call latency and error rate by connector/dependency.
● Timeouts, retries, circuit-breaker state (if implemented), and rate-limit responses.
● Process health, worker saturation, queue depth where applicable, and telemetry exporter health.
OSIF Framework — Architecture & Developer Guide Page 10
10. Error handling and resilience
Canonical error contract
A shared error contract lets applications return predictable responses while preserving useful internal diagnostics. The
exact OSIF model should be discovered in the common library or error-service connector and reused; avoid inventing
a second envelope.
# Example response shape for documentation; align with the existing OSIF contract.
{
"error": {
"code": "DEPENDENCY_TIMEOUT",
"message": "The downstream service did not respond in time.",
"correlation_id": "server-generated-id",
"retryable": true
}
}
Suggested error categories
Category Typical handling
Validation / schema Return a client error; do not retry.
Authentication / authorization Return safe 401/403 or equivalent; never retry blindly.
Not found / conflict Map according to operation semantics; normally not retryable.
Rate limited Back off within a deadline; honor Retry-After when supported.
Transient network / selected 5xx Potentially retry if the operation is safe and budget remains.
Timeout Return a distinct canonical code; consider retry only if safe and deadline allows.
Unexpected internal failure Log stack trace internally with correlation ID; return generic safe error.
Resilience rules
● Set a finite timeout for every outbound operation.
● Use bounded retries with exponential backoff and jitter; never retry all exceptions indiscriminately.
● Retry only transient failures and only when the operation's idempotency semantics allow it.
● Use an end-to-end deadline so retries do not exceed the caller's request budget.
● Use circuit breakers only where they provide value and define their metrics, thresholds, and recovery behavior.
● Avoid stacking retry loops in the API, common library, and connector. Define one clear policy owner.
● Make cancellation propagate; do not keep downstream calls running after the client disconnects unless explicitly
required.
If OSIF already has or plans a separate resilience package, keep retry/timeout/circuit-breaker policy there and let
connectors call it through a stable wrapper. Connector-specific code should provide operation context and
idempotency information, not copy/paste resilience logic.
11. Configuration, secrets, and environment management
Configuration principles
● Use typed settings with startup validation and clear error messages for missing required values.
● Separate non-secret configuration (timeouts, base URLs, API versions, feature flags) from secret material.
OSIF Framework — Architecture & Developer Guide Page 11
● Keep development, test, staging, and production values outside source control.
● Document defaults, units, allowed ranges, and precedence (for example environment variables over local defaults).
● Fail fast on invalid critical configuration rather than failing unpredictably on the first request.
Secret handling
● Use the approved secret manager or deployment secret injection mechanism.
● Never commit .env files with real credentials; provide .env.example with placeholders only.
● Do not print settings objects wholesale, since they may contain secret fields.
● Rotate credentials and document minimum required permissions/scopes.
● Use least-privilege service identities and separate credentials by environment.
Example environment template
# Illustrative names only; align to the repository's actual settings model.
APP_ENV=local
LOG_LEVEL=INFO
HTTP_CONNECT_TIMEOUT_SECONDS=3
HTTP_READ_TIMEOUT_SECONDS=10
SALESFORCE_INSTANCE_URL=https://example.invalid
SALESFORCE_API_VERSION=vXX.X
# Supply secrets through the approved secret manager; never commit real values.
12. Build, local development, and release workflow
The exact commands depend on each repository's pyproject.toml and packaging approach. The following is a safe
discovery workflow; replace placeholders with the commands defined by the project.
1 Clone or open the workspace and inspect the root README, each package README, pyproject.toml, lockfile, and
supported Python version.
2 Create and activate a virtual environment using the team's standard tool.
3 Install each package in the documented editable/development mode, respecting the workspace's dependency order.
4 Copy the example environment file and fill in local non-production values; obtain secrets through the approved
process.
5 Run formatting, linting, type checks, and unit tests before launching an application.
6 Start the sample API using the repository's documented ASGI entry point and local server command.
7 Call health/readiness endpoints, then run a small HTTP connector test against a mock server or approved sandbox.
8 Run Salesforce tests using mocked responses by default; use a sandbox only for explicitly marked integration tests.
9 Inspect logs and metrics, then test a timeout/error path—not only the happy path.
Suggested package release discipline
● Pin or constrain internal package versions explicitly in consuming applications.
● Build wheels and test installation from the built artifact, not only from the source checkout.
● Publish immutable versions; avoid overwriting released artifacts.
● Run compatibility tests against at least one representative consumer such as osif-orders-api.
● Maintain a changelog with added, changed, deprecated, and removed APIs.
OSIF Framework — Architecture & Developer Guide Page 12
13. Testing strategy and quality gates
Test layer What it proves Examples
Unit Small logic is deterministic and correct. Error mapping, schema validation, config validation,
retry eligibility.
Contract A connector obeys its public interface and
output/error contract.
HTTP connector returns canonical errors; MCP result
matches tool schema.
Component Package works with controlled
dependencies.
Mock HTTP server, fake Salesforce token endpoint,
exporter test double.
Integration Real dependency interoperability. Salesforce sandbox query; Kafka test broker; SAP test
environment.
End-to-end A user-visible workflow works through the
app.
Orders API route invokes connector and returns
expected public response.
Resilience Failure modes remain bounded and
diagnosable.
Timeout, connection reset, 429, 500, malformed
response, cancellation.
Security Sensitive data and unauthorized
operations are blocked.
Secret redaction, authz failure, invalid MCP tool input,
SSRF attempt.
Definition of done for a framework change
● Public API and intended package ownership are documented.
● Unit tests cover normal, invalid, and failure paths.
● No secrets or sensitive payloads appear in logs, traces, or test snapshots.
● Timeout, retry, and error behavior are explicitly tested where applicable.
● Type checking, linting, formatting, and tests pass in CI.
● Dependency additions are justified, pinned/controlled, and reviewed for security.
● Consumer impact and migration instructions are documented.
14. Security and production readiness
● Authentication vs authorization: validate identity and separately check whether the caller may invoke the
requested operation.
● Input validation: validate at the API/MCP boundary and again at critical connector boundaries where trust changes.
● SSRF protection: do not allow arbitrary untrusted URLs to drive outbound HTTP requests; restrict hosts, schemes,
ports, and redirects.
● Data minimization: send only required fields and avoid storing payloads unless necessary.
● Transport: use TLS verification; do not add insecure bypasses as a production workaround.
● Dependency security: scan packages, keep lockfiles current, and review transitive dependencies.
● Least privilege: use narrow Salesforce scopes and service permissions; separate environments.
● Safe errors: public messages should not disclose stack traces, internal hostnames, secrets, or implementation
details.
● Resource limits: cap request sizes, response sizes, concurrent calls, and execution time.
● Auditability: record security-relevant actions with actor, operation, outcome, and correlation ID, subject to data
policy.
OSIF Framework — Architecture & Developer Guide Page 13
15. New developer onboarding: first working integration
A practical path
1 Read this guide and the root README. Identify the actual supported Python version, package manager, and test
commands.
2 Run the template API without changing framework internals. Confirm health and readiness behavior.
3 Trace one existing route in osif-orders-api from route to service to connector. Draw the real dependency path.
4 Use the common logger and verify that a correlation ID appears in logs.
5 Make a read-only HTTP connector call against a mock endpoint, then force a timeout and inspect the normalized
error.
6 Review the Salesforce connector configuration and auth flow. Run unit tests with mocked token and API responses
before using a sandbox.
7 Register a safe read-only MCP tool that calls an existing application service. Test invalid input, unauthorized use,
and oversized output.
8 Enable observability in local mode and verify request count, latency, error, and trace context if supported.
9 Add tests and update package documentation before submitting a pull request.
Example feature implementation checklist
Step Action Avoid
1. Define contract Document operation, inputs, outputs, errors, and side
effects.
Starting with a route that embeds all logic.
2. Validate Use existing shared schema/configuration
conventions.
Duplicating validators with inconsistent rules.
3. Implement Put vendor/protocol I/O in the relevant connector. Putting Salesforce-specific code in
common-library.
4. Instrument Use shared logging and observability hooks. Logging full payloads or credentials.
5. Test Mock downstream behavior and cover failure paths. Tests that only cover 200 responses.
6. Document Explain configuration, examples, errors, and
compatibility.
Undocumented environment variables and
hidden defaults.
OSIF Framework — Architecture & Developer Guide Page 14
16. Troubleshooting guide
Symptom Likely causes First checks
Import/package error Package not installed, wrong environment,
stale editable install, wrong public import.
Inspect pyproject.toml, active interpreter,
package exports, and dependency constraints.
Configuration missing Variable name mismatch or settings
validation not wired at startup.
Compare .env.example and settings model; do
not print secret values.
HTTP timeout Downstream latency, network issue,
too-short timeout, connection pool
saturation.
Check connector timing metrics, timeout type,
host reachability, pool settings.
Repeated downstream
calls
Retry loops duplicated across layers or
retry conditions too broad.
Count attempts and inspect policy ownership,
backoff, and idempotency.
Salesforce 401/invalid
session
Expired token, wrong auth flow, scope or
instance URL mismatch.
Inspect sanitized auth error, token refresh logic,
API version and sandbox/prod config.
Salesforce partial results Pagination not followed or API limit
reached.
Check next-record/locator handling, result count,
and limit telemetry.
No correlation across
logs
Context not propagated through async task
or connector boundary.
Inspect middleware/context setup and outbound
trace propagation.
MCP tool rejected Name mismatch, invalid schema, tool not
registered, authorization denied.
Check registry, exact tool name, schema,
permissions, and safe error code.
Metrics missing Instrumentation not initialized, exporter
misconfigured, sampling/filtering.
Check startup hooks, exporter endpoint, service
name, and exporter logs.
Application fails when
telemetry is down
Exporter is blocking or telemetry errors are
propagated into business code.
Use bounded export, safe fallback, and separate
telemetry from business failure.
17. Architecture decisions and change-minimisation rules
Rules that reduce unnecessary changes
● Inspect and reuse the current public API before adding a new abstraction.
● Keep connector method signatures stable; add optional configuration with safe defaults when possible.
● Introduce new behavior behind opt-in configuration if its effect on existing consumers is uncertain.
● Do not change authentication, timeout, retry, or error behavior globally without compatibility tests.
● Keep business behavior in the application/service layer and transport/vendor behavior in connectors.
● Centralize policy once, but do not force every package to depend on an oversized common library.
● Prefer additive model fields and documented deprecations over removing fields immediately.
● Add a small vertical-slice example that demonstrates the intended integration rather than documenting isolated
classes only.
OSIF Framework — Architecture & Developer Guide Page 15
Recommended decision records
Decision Questions to settle
Connector contract Common base class or Protocol? Sync or async? Typed result model? How are capabilities
discovered?
Error contract Canonical fields, HTTP status mapping, retryable classification, safe public message.
Resilience ownership Which layer owns timeout/retry/circuit breaker? How is total deadline enforced?
Observability standard Logging fields, trace propagation, metric names/labels, exporter failure policy.
MCP security Authn/authz model, tool allow-list, write-operation policy, audit requirements.
Configuration Settings library, precedence, secret source, startup validation, deprecation policy.
Versioning Release cadence, semantic versioning, compatibility window, migration policy.
18. Repository audit checklist and documentation gaps
Because only a workspace screenshot was supplied, the following audit is the required next step before calling this
documentation implementation-exact.
Inspect every package
● README.md: install, purpose, quick start, configuration, examples, support owner.
● pyproject.toml: Python version, runtime dependencies, optional extras, package name, build backend,
lint/test/type-check commands.
● src/ package: public exports, module boundaries, async/sync behavior, dependency injection, initialization side
effects.
● tests/: unit/integration split, mocks, coverage, external service requirements.
● Configuration: environment variable names, defaults, validation, secret references, environment-specific behavior.
● CI/CD: build, tests, lint, security scanning, versioning, publication, deployment.
● Examples: one API using HTTP connector, one Salesforce flow, one MCP tool, one observability setup.
Per-package README template
# Package name
## Purpose
## When to use / when not to use
## Public API and supported operations
## Installation and supported Python versions
## Quick start
## Configuration reference (name, type, default, required, secret?)
## Authentication and authorization
## Timeouts, retries, errors, and idempotency
## Logging, metrics, traces, and correlation IDs
## Testing (unit, mocked integration, sandbox)
## Security considerations
## Compatibility and versioning
## Troubleshooting
## Maintainers / support channel
Acceptance criteria for the complete documentation effort
● Every package has a README using the template above.
● A workspace-level architecture document explains dependency direction and request flow.
OSIF Framework — Architecture & Developer Guide Page 16
● A new developer can run a sample service from a clean checkout using only documented steps.
● At least one tested example demonstrates HTTP, one demonstrates Salesforce, and one demonstrates MCP.
● Configuration reference is generated or checked against actual settings models.
● Error codes and response envelopes are documented from source, not guessed.
● CI verifies documentation examples or runs smoke tests for the documented quick starts.
● A maintainer owns each package and documents support/escalation expectations.
Appendix A. Suggested documentation structure
docs/
architecture/
overview.md
dependency-rules.md
request-lifecycle.md
decisions/
packages/
common-library.md
common-logger.md
common-errorservice.md
connector-http.md
connector-salesforce.md
connector-kafka.md
connector-sap.md
mcp.md
observability.md
guides/
local-development.md
configuration-and-secrets.md
testing.md
production-deployment.md
troubleshooting.md
examples/
http-integration/
salesforce-integration/
mcp-tool/
Appendix B. Production-readiness checklist
■ Public contracts are documented and typed.
■ All outbound calls have finite timeouts.
■ Retries are bounded, observable, and safe for the operation.
■ Secrets and sensitive fields are redacted.
■ Correlation IDs and traces propagate across API and connector calls.
■ Metrics use low-cardinality labels.
■ MCP tools validate inputs and enforce authorization.
■ Errors are normalized and public messages are safe.
■ Shutdown closes HTTP clients and other resources cleanly.
■ Unit, contract, failure-mode, and integration tests are in CI.
■ Configuration is validated at startup and secrets are not committed.
■ Dependency versions and release compatibility are managed.
■ Runbooks, dashboards, and ownership are documented.
OSIF Framework — Architecture & Developer Guide Page 17