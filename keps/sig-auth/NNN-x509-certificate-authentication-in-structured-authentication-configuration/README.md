# KEP-NNNN: X.509 Certificate Authentication in StructuredAuthenticationConfiguration

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

This KEP proposes adding support for X.509 certificate authentication configuration within the StructuredAuthenticationConfiguration API. This enhancement extends the existing structured authentication framework to include certificate-based authentication with CEL (Common Expression Language) validation rules for both request-level and user-level validation.

The proposal introduces a new `X509` field in the `AuthenticationConfiguration` that allows administrators to configure certificate authentication with fine-grained control over certificate validation, issuer constraints, and custom validation logic using CEL expressions.

## Motivation

Currently, X.509 certificate authentication in Kubernetes is configured through command-line flags and lacks the flexibility and expressiveness provided by the StructuredAuthenticationConfiguration framework. This creates inconsistencies in how different authentication methods are configured and limits the ability to apply advanced validation rules to certificate-based authentication.

The existing certificate authentication mechanism has several limitations:
- Limited configurability compared to other authentication methods like JWT
- No support for custom validation rules beyond basic certificate chain validation
- Lack of runtime configurability and dynamic validation capabilities

### Goals

- Extend StructuredAuthenticationConfiguration to support X.509 certificate authentication
- Provide CEL-based validation rules for request-level validation (e.g., IP address restrictions)
- Provide CEL-based validation rules for user-level validation after certificate validation
- Maintain backward compatibility with existing certificate authentication mechanisms
- Enable fine-grained control over certificate issuer validation
- Support multiple certificate authentication configurations within a single cluster

### Non-Goals

- Replace the existing command-line flag-based certificate authentication (backward compatibility must be maintained)
- Change the fundamental certificate chain validation process

## Proposal

This proposal adds X.509 certificate authentication support to StructuredAuthenticationConfiguration by introducing new API types and extending the existing authentication framework.

### User Stories (Optional)

#### Story 1

As a cluster administrator, I want to configure certificate authentication with IP address restrictions so that certificates can only be used from specific network ranges, enhancing the security posture of my cluster.

Example configuration:
```yaml
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
x509:
- issuer:
    name: "my-ca"
  requestValidationRules:
  - expression: 'request.remoteaddr.startsWith("10.0.0.")'
    message: "Certificate authentication only allowed from internal network"
```

#### Story 2

As a cluster administrator, I want to apply custom validation rules to users authenticated via certificates, such as ensuring certain user attributes are present or meet specific criteria before allowing access.

Example configuration:
```yaml
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
x509:
- issuer:
    name: "my-ca"
  userValidationRules:
  - expression: 'user.username.startsWith("system:")'
    message: "User must be a system user or member of developers group"
```

### Notes/Constraints/Caveats (Optional)

- CEL expressions for request validation have access to a limited set of request attributes (initially just remote address)
- The certificate issuer name matching is based on the Common Name field of the certificate issuer
- User validation rules are evaluated after successful certificate authentication
- The feature requires the StructuredAuthenticationConfiguration feature gate to be enabled

### Risks and Mitigations

**Risk**: Complex CEL expressions could make debugging authentication failures difficult
**Mitigation**: Comprehensive logging and clear error messages will be provided. Documentation will include best practices for CEL expression design.

**Risk**: Misconfigured validation rules could lock out legitimate users
**Mitigation**: Validation rules will be optional, and the system will fail open if CEL evaluation encounters errors (with appropriate logging).

## Design Details

The implementation introduces several new API types:

```go
type X509AuthConfig struct {
    Issuer                 CertIssuer              `json:"issuer,omitempty"`
    RequestValidationRules []RequestValidationRule `json:"requestValidationRules,omitempty"`
    UserValidationRules    []UserValidationRule    `json:"userValidationRules,omitempty"`
}

type CertIssuer struct {
    Name string `json:"name,omitempty"`
}

type RequestValidationRule struct {
    Expression string `json:"expression,omitempty"`
    Message    string `json:"message,omitempty"`
}
```

The `AuthenticationConfiguration` is extended with:
```go
type AuthenticationConfiguration struct {
    JWT  []JWTAuthenticator `json:"jwt"`
    X509 []X509AuthConfig   `json:"x509,omitempty"`  // New field
    Anonymous *AnonymousAuthConfig `json:"anonymous,omitempty"`
}
```

### CEL Environment

Request validation rules have access to a `request` variable with the following structure:
```
request.remoteaddr (string) - The remote IP address of the client
```

User validation rules have access to a `user` variable with the standard UserInfo structure:
```
user.username (string) - The username from the certificate
user.uid (string) - The user ID
user.groups ([]string) - The user's groups
user.extra (map[string][]string) - Additional user attributes
```

### Test Plan

#### Prerequisite testing updates

- Update existing certificate authentication tests to ensure backward compatibility
- Add CEL compiler validation tests for the new expression types

#### Unit tests

New unit tests will cover:
- X509AuthConfig validation logic
- CEL expression compilation and evaluation
- Request and user validation rule processing

#### Integration tests

#### e2e tests

### Graduation Criteria

#### Alpha

- Feature implemented behind the StructuredAuthenticationConfiguration feature flag
- Basic X.509 authentication with CEL validation rules working
- Unit and integration tests completed and enabled
- API validation ensures only valid CEL expressions are accepted

#### Beta

#### GA

### Upgrade / Downgrade Strategy

The feature is additive and does not modify existing authentication behavior. Clusters can be upgraded with the feature disabled, and existing certificate authentication will continue to work unchanged.

When downgrading, the X509 configuration in StructuredAuthenticationConfiguration will be ignored, and certificate authentication will fall back to the existing command-line flag configuration.

### Version Skew Strategy

This feature only affects the API server component. No coordination with other cluster components is required. The feature is controlled by a feature gate and can be enabled/disabled independently on each API server instance.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: StructuredAuthenticationConfiguration
  - Components depending on the feature gate: kube-apiserver

###### Does enabling the feature change any default behavior?

No. The feature only adds new configuration options. Existing certificate authentication behavior remains unchanged when the feature is enabled but not configured.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Disabling the StructuredAuthenticationConfiguration feature gate will cause the API server to ignore X509 configuration in StructuredAuthenticationConfiguration and fall back to existing certificate authentication mechanisms.

###### What happens if we reenable the feature if it was previously rolled back?

The X509 configuration in StructuredAuthenticationConfiguration will be processed again, and certificate authentication will use the configured validation rules.

###### Are there any tests for feature enablement/disablement?

Yes. Unit tests will verify that the feature can be enabled and disabled, and that the appropriate authentication behavior is used in each case.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

###### What specific metrics should inform a rollback?

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No deprecations or removals are included in this rollout.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

- Check if X509 configuration is present in StructuredAuthenticationConfiguration
- Monitor authentication metrics filtered by authenticator type
- Review API server logs for X509 CEL evaluation messages

###### How can someone using this feature know that it is working for their instance?

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
- X509AuthConfig
- CertIssuer  
- RequestValidationRule (shared with existing JWT authentication)

These are configuration types only and do not create persistent objects.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

The AuthenticationConfiguration object will increase in size when X509 configuration is added, but this is a single configuration object per API server.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

Authentication latency may increase slightly due to CEL expression evaluation, but this is expected to be minimal.

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
  - Mitigations: TBD
  - Diagnostics: Runtime CEL evaluation errors logged with expression details
  - Testing: TBD

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

- 2024-08-25: Initial KEP draft created
- 2024-08-25: Prototype implementation completed with basic X509 support and CEL validation

## Drawbacks

## Alternatives

1. **Extend existing command-line flags**: Could add more flags for certificate validation, but this would not provide the flexibility of CEL expressions and would continue the inconsistent configuration approach.

2. **Use admission controllers**: Could implement certificate validation logic in admission controllers, would be less efficient.

3. **External authentication webhook**: Could use webhook authentication for certificate validation, but this would add network dependencies and latency to the authentication process.

## Infrastructure Needed (Optional)

No additional infrastructure is needed. The implementation uses existing Kubernetes CEL libraries and certificate validation mechanisms.