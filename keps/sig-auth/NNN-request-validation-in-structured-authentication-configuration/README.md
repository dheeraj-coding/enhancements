# KEP-NNNN: Request Validation in StructuredAuthenticationConfiguration

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP proposes adding a generic request validation layer to the StructuredAuthenticationConfiguration API. This enhancement introduces a new `RequestInfoValidation` field that allows administrators to configure CEL (Common Expression Language) validation rules that are applied to all authentication requests, regardless of the authentication method used.

The proposal introduces a new authentication layer that wraps existing authenticators and provides fine-grained control over request validation based on request attributes such as remote IP, headers, certificate information, and other request metadata.

## Motivation

Currently, Kubernetes authentication methods lack a unified way to apply request-level validation rules across different authentication mechanisms. Each authentication method (JWT, X.509, etc.) has its own validation logic, and there's no consistent way to apply common security policies like IP address restrictions, header validation, or certificate issuer checks across all authentication methods.

The existing authentication framework has several limitations:
- No unified request validation layer across authentication methods
- Limited ability to apply security policies based on request metadata
- Lack of runtime configurability for request validation rules
- No support for custom validation logic beyond what each authenticator provides

### Goals

- Introduce a generic request validation layer that works with any authentication method
- Provide CEL-based validation rules for request-level validation (e.g., IP address restrictions, header validation)
- Enable fine-grained control over request attributes and metadata
- Maintain backward compatibility with existing authentication mechanisms
- Support multiple request validation rules within a single cluster
- Allow validation based on certificate information when available

### Non-Goals

- Replace existing authentication methods or their specific validation logic
- Change the fundamental authentication flow or user validation process
- Provide authorization capabilities (this is purely authentication-focused)

## Proposal

This proposal adds a generic request validation layer to StructuredAuthenticationConfiguration by introducing new API types and extending the existing authentication framework with a wrapper authenticator.

### User Stories (Optional)

#### Story 1

As a cluster administrator, I want to restrict authentication requests to specific IP address ranges across all authentication methods, so that I can ensure all authentication attempts come from trusted networks regardless of whether users authenticate via JWT, certificates, or other methods.

Example configuration:
```yaml
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
requestInfoValidation:
- expression: 'request.RemoteIP.startsWith("10.0.0.") || request.RemoteIP.startsWith("192.168.")'
  message: "Authentication only allowed from internal networks"
jwt:
- issuer:
    url: "https://example.com"
```

#### Story 2

As a cluster administrator, I want to apply different validation rules based on the authentication method being used, such as requiring specific certificate issuers for certificate-based authentication while allowing broader access for JWT authentication.

Example configuration:
```yaml
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
requestInfoValidation:
- expression: 'request.Header.Authorization.Scheme == "Bearer" || (request.CertificateIssuerName != "" && request.CertificateIssuerName == "trusted-ca")'
  message: "Invalid authentication method or untrusted certificate issuer"
```

#### Story 3

As a cluster administrator, I want to restrict certificate-based authentication to only local IP addresses while allowing other authentication methods from remote addresses, ensuring that X.509 certificate communication between API server and etcd is limited to localhost while external clients can use token-based authentication.

Example configuration:
```yaml
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
requestValidationRules:
- expression: 'request.Header.Authorization.Scheme != "" || request.RemoteIP.startsWith("127.0.0.1") || request.RemoteIP.startsWith("::1")'
  message: "Certificate-based authentication only allowed from localhost"
```

### Notes/Constraints/Caveats (Optional)

- CEL expressions for request validation have access to a comprehensive set of request attributes
- The request validation layer is applied after successful authentication but before returning the user info
- Certificate information is only available when certificate-based authentication is used
- The feature requires the StructuredAuthenticationConfiguration feature gate to be enabled

### Risks and Mitigations

**Risk**: Complex CEL expressions could make debugging authentication failures difficult
**Mitigation**: Comprehensive logging and clear error messages will be provided. Documentation will include best practices for CEL expression design.

**Risk**: Misconfigured validation rules could lock out legitimate users
**Mitigation**: Validation rules will be optional, and the system will fail open if CEL evaluation encounters errors (with appropriate logging).

**Risk**: Performance impact from CEL evaluation on every authentication request
**Mitigation**: CEL expressions are compiled once at startup and cached. The evaluation overhead is minimal for simple expressions.

## Design Details

The implementation introduces several new API types:

```go
type RequestValidationRule struct {
    Expression string `json:"expression"`
    Message    string `json:"message"`
}
```

The `AuthenticationConfiguration` is extended with:
```go
type AuthenticationConfiguration struct {
    JWT                    []JWTAuthenticator      `json:"jwt"`
    RequestInfoValidation  []RequestValidationRule `json:"requestInfoValidation,omitempty"`  // New field
    Anonymous              *AnonymousAuthConfig    `json:"anonymous,omitempty"`
}
```

### Request Validation Layer Architecture

The request validation layer is implemented as a wrapper authenticator that:

1. Receives authentication requests
2. Delegates to the underlying authenticator (JWT, X.509, etc.)
3. If authentication succeeds, evaluates CEL validation rules against request metadata
4. Returns the authentication result only if all validation rules pass

### CEL Environment

Request validation rules have access to a `request` variable with the following structure (TBD open to suggestions):
```
request.RemoteIP (string) - The remote IP address of the client
request.RemotePort (string) - The remote port of the client
request.Header.Host (string) - The Host header value
request.Header.UserAgent (string) - The User-Agent header value
request.Header.Authorization.Scheme (string) - The authorization scheme (Bearer, Basic, etc.)
request.CertificateIssuerName (string) - The certificate issuer name (when available)
```

### Implementation Details

The request validation layer is implemented in a new package:
`k8s.io/apiserver/pkg/authentication/request/requestvalidation`

Key components:
- `RequestValidator` struct that wraps any authenticator
- `RequestInfo` type that represents the CEL-accessible request data
- CEL compilation and evaluation logic
- Integration with the existing authentication configuration system

### Test Plan

#### Prerequisite testing updates

- Update existing authentication tests to ensure backward compatibility
- Add CEL compiler validation tests for the new expression types

#### Unit tests

New unit tests will cover:
- RequestValidationRule validation logic
- CEL expression compilation and evaluation
- Request validation rule processing
- Integration with different authenticator types

#### Integration tests

Integration tests will verify:
- TBD

#### e2e tests

E2e tests will validate:
- TBD

### Graduation Criteria

#### Alpha

- Feature implemented behind the StructuredAuthenticationConfiguration feature gate
- Basic request validation with CEL validation rules working
- Unit and integration tests completed and enabled
- API validation ensures only valid CEL expressions are accepted

#### Beta

- TBD

#### GA

- TBD

### Upgrade / Downgrade Strategy

The feature is additive and does not modify existing authentication behavior. Clusters can be upgraded with the feature disabled, and existing authentication will continue to work unchanged.

When downgrading, the RequestValidationRules configuration in StructuredAuthenticationConfiguration will be ignored, and authentication will fall back to the existing behavior without request validation.

### Version Skew Strategy

This feature only affects the API server component. No coordination with other cluster components is required. The feature is controlled by a feature gate and can be enabled/disabled independently on each API server instance.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: StructuredAuthenticationConfiguration
  - Components depending on the feature gate: kube-apiserver

###### Does enabling the feature change any default behavior?

No. The feature only adds new configuration options. Existing authentication behavior remains unchanged when the feature is enabled but not configured.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Disabling the StructuredAuthenticationConfiguration feature gate will cause the API server to ignore RequestValidationRules configuration and fall back to existing authentication mechanisms without request validation.

###### What happens if we reenable the feature if it was previously rolled back?

The RequestValidationRules configuration will be processed again, and authentication will use the configured validation rules.

###### Are there any tests for feature enablement/disablement?

Yes. Unit tests will verify that the feature can be enabled and disabled, and that the appropriate authentication behavior is used in each case.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

A rollout could fail if:
- CEL expressions are malformed and prevent API server startup
- Validation rules are too restrictive and block legitimate authentication

Impact on running workloads is minimal since this only affects new authentication requests.

###### What specific metrics should inform a rollback?


###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Testing will include upgrade/downgrade scenarios to ensure configuration is properly ignored when the feature is disabled.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No deprecations or removals are included in this rollout.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

- Check if RequestValidationRules configuration is present in StructuredAuthenticationConfiguration
- Monitor authentication metrics for request validation failures
- Review API server logs for request validation messages

###### How can someone using this feature know that it is working for their instance?

- Authentication requests that should be blocked by validation rules are rejected
- API server logs show request validation evaluation messages

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

- Authentication success rate should not be impacted by CEL evaluation failures

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No. This feature only depends on the CEL library which is already used by other Kubernetes components.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No new API calls are generated by this feature.

###### Will enabling / using this feature result in introducing new API types?

Yes, new API types are introduced:
- RequestValidationRule

These are configuration types only and do not create persistent objects.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

The AuthenticationConfiguration object will increase in size when RequestValidationRules configuration is added, but this is a single configuration object per API server.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

Minimal CPU increase during authentication due to CEL evaluation. Memory usage will increase slightly to store compiled CEL expressions. The impact is expected to be negligible.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No. This feature only affects the API server and does not create additional processes or file handles.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

This feature only affects authentication processing within the API server. If the API server is unavailable, authentication cannot occur regardless of this feature. etcd availability does not impact this feature as it does not store or retrieve data from etcd during authentication.

###### What are other known failure modes?

- [CEL Expression Compilation Failure]
  - Detection: API server startup logs will show compilation errors
  - Mitigations: Fix CEL expressions in configuration and restart API server
  - Diagnostics: Clear error messages in startup logs indicating which expressions failed
  - Testing: Unit tests cover invalid CEL expression scenarios

- [CEL Expression Runtime Failure]  
  - Detection: Authentication errors in API server logs and metrics
  - Mitigations: System fails open with appropriate logging
  - Diagnostics: Runtime CEL evaluation errors logged with expression details
  - Testing: Unit tests cover runtime evaluation failures

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

- 2024-09-05: Initial KEP draft created for generic request validation layer

## Drawbacks

## Alternatives

1. **Extend individual authenticators**: Could add validation logic to each authenticator type, but this would duplicate code and create inconsistencies.

2. **Use admission controllers**: Could implement request validation logic in admission controllers, but this would be less efficient and wouldn't prevent authentication.

3. **External authentication webhook**: Could use webhook authentication for request validation, but this would add network dependencies and latency.

4. **Network policies**: Could use network policies for IP restrictions, but this wouldn't provide the flexibility of CEL expressions or access to request metadata.

## Infrastructure Needed (Optional)

No additional infrastructure is needed. The implementation uses existing Kubernetes CEL libraries and authentication mechanisms.
