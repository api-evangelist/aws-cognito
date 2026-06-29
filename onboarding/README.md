# Programmatic API Onboarding — AWS Cognito

A single-file, zero-dependency Node.js (18+) CLI that reproduces SoundCloud's
`sc-api-auth.mjs` pattern for AWS Cognito: register an application / obtain credentials
programmatically instead of clicking through a dashboard, so agents and developers
can onboard at the command line.

- Script: [`aws-cognito-api-auth.mjs`](aws-cognito-api-auth.mjs)
- Run `node aws-cognito-api-auth.mjs --help` for usage and the required environment variables.
- Story / rationale: https://apievangelist.com/2026/07/30/aws-cognito-has-credentials-not-front-door/

Part of the API Evangelist "Programmatic API Onboarding for the Agentic Moment" series.
