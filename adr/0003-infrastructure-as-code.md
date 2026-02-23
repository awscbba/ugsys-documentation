# ADR-0003: Infrastructure as Code with AWS CDK (Python)

**Date**: 2026-02-23
**Status**: Accepted
**Deciders**: AWSUGCBBA Platform Team

## Context

The platform requires shared AWS infrastructure: EventBridge bus, Route53 hosted zone, GitHub OIDC roles, KMS keys, and CloudWatch observability. This infrastructure must be versioned, reproducible, and deployable via CI/CD.

## Decision

Use **AWS CDK v2 (Python)** in `ugsys-platform-infrastructure` for all shared infrastructure.

### Stacks

| Stack | Resources |
|---|---|
| `EventBusStack` | EventBridge custom bus `ugsys-platform-bus`, 30-day archive, CloudWatch log group |
| `DnsStack` | Route53 public hosted zone for `cbba.cloud.org.bo` |
| `GithubOidcStack` | OIDC provider + per-repo deploy IAM roles for all 7 repos |
| `SecurityStack` | Shared KMS key with automatic rotation |
| `ObservabilityStack` | CloudWatch dashboard + per-service log groups |

### Deployment

```bash
just synth    # cdk synth
just deploy   # cdk deploy --all
```

CI/CD uses GitHub OIDC (no long-lived AWS credentials). Each service repo has a scoped deploy role created by `GithubOidcStack`.

## Alternatives Considered

### Terraform
Rejected — team has stronger CDK/Python familiarity. CDK also provides better L2/L3 construct abstractions for AWS-native resources.

### SAM
Rejected — SAM is Lambda-focused and lacks constructs for shared infrastructure like EventBridge and Route53.

## Consequences

- Infrastructure changes go through PR review like application code
- CDK bootstrap required once per account/region: `cdk bootstrap aws://ACCOUNT/REGION`
- Drift detection via `cdk diff` in CI
- All services get CloudWatch log groups and dashboard widgets automatically
