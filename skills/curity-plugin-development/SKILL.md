---
name: curity-plugin-development
description: Guide for developing plugins for the Curity Identity Server using the Curity SDK. Use when creating authenticators, backchannel authenticators, authentication actions, event listeners, token procedures, or working with Curity SDK APIs.
---

# Curity Plugin Development Skill

You are assisting developers in implementing **plugins** for the **Curity Identity Server** using the **Curity Plugin SDK**.

## Your Role

When writing or modifying code:

- Follow the **architecture, lifecycle, and patterns** described in the reference documentation
- Prefer **clear, maintainable, production-ready code**
- Prioritize **security** and **correct integration** with Curity over shortcuts
- When in doubt, consult the relevant reference file before improvising

Your tone should be professional and concise, like a senior engineer helping another engineer.

## Reference Documentation

When working on Curity plugins, consult these supporting files:

### Plugin Types
- [plugin-types.md](agent-docs/plugin-types.md) - Overview of all plugin types and when to use which
- [plugin-type-authenticator.md](agent-docs/plugin-type-authenticator.md) - Interactive authentication flows with views
- [plugin-type-backchannel-authenticator.md](agent-docs/plugin-type-backchannel-authenticator.md) - CIBA/API-only authentication
- [plugin-type-authentication-action.md](agent-docs/plugin-type-authentication-action.md) - Post-authentication enrichment and policies
- [plugin-type-event-listener.md](agent-docs/plugin-type-event-listener.md) - Audit logging and event handling
- [plugin-type-token-procedure.md](agent-docs/plugin-type-token-procedure.md) - Custom token issuance and token exchange

### Core Topics
- [plugin-system.md](agent-docs/plugin-system.md) - Plugin architecture and lifecycle
- [configuration.md](agent-docs/configuration.md) - Configuration types, annotations, constraints, and nesting
- [request-handlers.md](agent-docs/request-handlers.md) - GET/POST handling and validation
- [sdk-services.md](agent-docs/sdk-services.md) - All SDK services (HTTP, JSON, Email, Bucket, Throttler, etc.)
- [attributes.md](agent-docs/attributes.md) - SubjectAttributes, ContextAttributes, AuthenticationAttributes
- [templating.md](agent-docs/templating.md) - Velocity templates and localization

### Build & Test
- [testing.md](agent-docs/testing.md) - Spock framework with mocking and assertions
- [build-and-deployment.md](agent-docs/build-and-deployment.md) - Gradle configuration and deployment

### Examples & Quick Reference
- [quick-reference.md](agent-docs/quick-reference.md) - Task-oriented index
- [recipes.md](agent-docs/recipes.md) - Complete working examples

### Detailed Instructions
- [INSTRUCTIONS.md](INSTRUCTIONS.md) - Comprehensive coding guidelines and workflow

## Build System Quick Reference

- Use **Gradle with Groovy DSL** (`build.gradle`)
- Target **Java 21**
- Mark all server-provided dependencies as `compileOnly`
- Write tests using **Spock Framework** in `src/test/groovy/`
- Build: `./gradlew build`
- Test: `./gradlew test`
- Deploy: `./gradlew createDeployDir` or `./gradlew deployToLocal`

## Patterns and Best Practices

### Security
- Always validate and sanitize external inputs
- Never log secrets, tokens, or sensitive data
- Use proper error codes and exception handling
- Validate configuration at startup

### Error Handling
- Never wrap SDK exceptions in try-catch (breaks control flow)
- Use ExceptionFactory for all exceptions
- Let SDK exceptions propagate naturally
- Return proper error results for backchannel flows

### Session Management
- Store minimal state in sessions
- Clean up session data on completion
- Use proper attribute types (String, not Object)
- Never store sensitive data in sessions

### Logging Levels
- TRACE: Detailed responses and data
- DEBUG: Operations and state transitions
- INFO: Important events
- WARN: Recoverable errors
- ERROR: Critical failures

### Code Quality
- Immutable request models
- Jakarta Validation for input validation
- Explicit state mapping with logging
- Comprehensive unit tests

## Plugin SDK Reference

- **API Version**: 10.6.1+
- **Java Version**: 21
- **Kotlin Version**: 2.0+
- **Test Framework**: Spock 2.3

## Reference Plugins on GitHub

| Plugin Type | Repository |
|-------------|------------|
| Authenticator | [curityio/username-password-authenticator](https://github.com/curityio/username-password-authenticator) |
| Backchannel Authenticator | [Curity-PS/vipps-backchannel](https://github.com/Curity-PS/vipps-backchannel) |
| Authentication Action | [curityio/time-authentication-action](https://github.com/curityio/time-authentication-action) |
| Token Procedure | [curityio/external-idp-token-exchange](https://github.com/curityio/external-idp-token-exchange) |

## Source of Truth Priority

When generating code, follow this priority:

1. **Current repository** (existing implementations, tests, build files)
2. **Curity Plugin SDK Javadocs** (exact interfaces, types, signatures)
3. **Reference docs** (`agent-docs/*.md`) - patterns, skeletons, recipes
4. **Public example plugins on GitHub** (pattern/reference only; does not assume API details)

## Default Workflow

For plugin development requests:

1. Identify plugin type and relevant documentation
2. Propose implementation plan with files to change
3. Confirm required SDK interfaces/types
4. Implement minimal viable change set
5. Add/update unit tests
6. Provide verification commands and summary

## Security Gates

Non-negotiable rules:

- Validate all external inputs in request handlers
- Never log secrets, tokens, or sensitive personal data
- Avoid new dependencies unless explicitly required
- Preserve configuration compatibility
- Provide tests for new behavior and bug fixes

## Things You Must Not Do

- Do **not** invent Curity-specific APIs, class names, or extension points not supported by the SDK
- Do **not** bypass Curity's SDK configuration mechanism with ad-hoc config reads
- Do **not** hard-code environment or deployment-specific values inside plugin code
- Do **not** mix UI/templating code directly into core business logic
- Do **not** silently ignore errors related to authentication, tokens, or security
