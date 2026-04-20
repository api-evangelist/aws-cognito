# Amazon Cognito (aws-cognito)
Amazon Cognito is an AWS service that provides authentication, authorization, and user management for web and mobile applications. It supports OAuth2, OIDC, SAML federation, and social identity providers. Cognito has two main components: User Pools for user authentication and app integration, and Federated Identities for granting temporary AWS credentials to authenticated users.

**URL:** [https://aws.amazon.com/cognito/](https://aws.amazon.com/cognito/)

**Run:** [Capabilities Using Naftiko](https://github.com/naftiko/fleet?utm_source=api-evangelist&utm_medium=readme&utm_campaign=company-api-evangelist&utm_content=repo)

## Tags:

 - Authentication, Authorization, AWS, Identity, Identity Provider, OAuth2, OIDC

## Timestamps

- **Created:** 2026-03-25
- **Modified:** 2026-04-19

## APIs

### Amazon Cognito Identity Provider
Control plane API for managing Cognito user pools, app clients, users, groups, identity providers, and resource servers.

**Human URL:** [https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html)

#### Tags:

 - Authentication, AWS, Identity Provider, OAuth2, User Pools

#### Properties

- [Documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html)
- [APIReference](https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/Welcome.html)
- [OpenAPI](openapi/aws-cognito-identity-provider-openapi.yaml)

### Amazon Cognito Identity (Federated Identities)
Federated identity service that issues temporary AWS credentials to authenticated users from Cognito user pools, social providers, and enterprise SAML IdPs.

**Human URL:** [https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-identity.html](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-identity.html)

#### Tags:

 - AWS, Credentials, Federation, Identity

#### Properties

- [Documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-identity.html)
- [APIReference](https://docs.aws.amazon.com/cognitoidentity/latest/APIReference/Welcome.html)
- [OpenAPI](openapi/aws-cognito-identity-openapi.yaml)

## Common Properties

- [Website](https://aws.amazon.com/cognito/)
- [Documentation](https://docs.aws.amazon.com/cognito/)
- [GettingStarted](https://docs.aws.amazon.com/cognito/latest/developerguide/getting-started-with-cognito-user-pools.html)
- [Pricing](https://aws.amazon.com/cognito/pricing/)
- [FAQ](https://aws.amazon.com/cognito/faqs/)
- [GitHubOrganization](https://github.com/aws-amplify)
- [Console](https://console.aws.amazon.com/cognito/)

## Features

| Name | Description |
|------|-------------|
| User Pools | Fully managed user directories with sign-up, sign-in, and user profile management. |
| OAuth2 and OIDC | Standards-based OAuth2 authorization server and OpenID Connect identity provider for apps. |
| SAML Federation | Integrate enterprise identity providers via SAML 2.0 for single sign-on. |
| Social Identity Providers | Sign in with Google, Facebook, Apple, and Amazon without custom backend code. |
| Multi-Factor Authentication | Built-in MFA with SMS, TOTP, and email verification options. |
| Customizable Auth Flows | Lambda triggers for custom authentication challenges and post-confirmation hooks. |
| Advanced Security Features | Risk-based adaptive authentication with compromised credential detection. |
| Federated Identities | Grant temporary AWS credentials to users authenticated via user pools or social providers. |
| Hosted UI | Pre-built customizable sign-in/sign-up pages with OAuth2 endpoint support. |
| Fine-Grained Authorization | Attribute-based access control with group-based IAM role assignment. |

## Use Cases

| Name | Description |
|------|-------------|
| Web and Mobile App Authentication | Add user registration, login, and session management to web and mobile applications. |
| Enterprise SSO Integration | Connect enterprise SAML identity providers for single sign-on to AWS-hosted applications. |
| API Authorization | Use Cognito JWT tokens to authorize access to API Gateway, AppSync, and custom APIs. |
| B2C Identity Management | Manage consumer user accounts with self-service registration and profile management. |
| Temporary AWS Credentials | Issue scoped AWS credentials to authenticated users for direct service access. |

## Integrations

| Name | Description |
|------|-------------|
| Amazon API Gateway | Validate Cognito JWTs for API Gateway authorizer integration. |
| AWS Amplify | Pre-built Amplify Auth library for easy Cognito integration in React, Vue, and mobile apps. |
| AWS Lambda | Trigger Lambda functions for custom authentication logic and user data enrichment. |
| Amazon DynamoDB | Use Cognito identity IDs as DynamoDB partition keys for per-user data isolation. |
| AWS IAM | Map Cognito groups to IAM roles for role-based access control to AWS services. |
| AWS AppSync | Use Cognito user pools as authorization mode for GraphQL API access control. |

## Artifacts

Machine-readable API specifications organized by format.

### OpenAPI

- [Amazon Cognito Identity Provider](openapi/aws-cognito-identity-provider-openapi.yaml)
- [Amazon Cognito Identity (Federated Identities)](openapi/aws-cognito-identity-openapi.yaml)

### JSON Schema

575 schema files covering UserPool, UserType, GroupType, IdentityPool, and supporting types.

### JSON Structure

575 JSON Structure files converted from JSON Schema.

### JSON-LD

- [Cognito Identity Provider Context](json-ld/aws-cognito-cognito-idp-context.jsonld)
- [Cognito Identity Context](json-ld/aws-cognito-cognito-identity-context.jsonld)

### Examples

575 example JSON files generated from JSON Schema definitions.

## Capabilities

Naftiko capabilities organized as shared per-API definitions composed into customer-facing workflows.

### Shared Per-API Definitions

- [Cognito Identity Provider](capabilities/shared/cognito-identity-provider.yaml) — 7 operations for user pool and user management
- [Cognito Identity](capabilities/shared/cognito-identity.yaml) — 3 operations for federated identity pool management

### Workflow Capabilities

| Workflow | APIs Combined | Tools | Persona |
|----------|--------------|-------|---------|
| [Identity Management Workflow](capabilities/identity-management-workflow.yaml) | Identity Provider, Federated Identity | 10 | Identity Engineer, Application Developer |

## Vocabulary

- [Amazon Cognito Vocabulary](vocabulary/aws-cognito-vocabulary.yaml) — Unified taxonomy mapping 6 resources, 6 actions, 1 workflow, and 2 personas

## Rules

- [Amazon Cognito Spectral Rules](rules/aws-cognito-spectral-rules.yml) — 16 rules across 7 categories enforcing Amazon Cognito API conventions

## Maintainers

**FN:** Kin Lane

**Email:** kin@apievangelist.com
