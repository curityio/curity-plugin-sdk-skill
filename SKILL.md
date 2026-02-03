# Curity Plugin Development Skill

A comprehensive skill for developing Curity Identity Server plugins, with expert knowledge of the Curity Plugin SDK, best practices, and common patterns.

## Description

This skill enables AI coding agents to generate production-ready Curity Identity Server plugins following official patterns and best practices. It provides deep knowledge of:

- Plugin system architecture and lifecycle
- Multiple plugin types (authenticators, backchannel authenticators, authentication actions)
- Curity SDK services and APIs
- Request handlers and validation
- Testing with Spock framework
- Build configuration and deployment

## Installation

Install this skill by cloning or adding this repository to your coding agent's skills directory:

```bash
# For Cody or similar agents
git clone <repository-url> ~/.cody/skills/curity-plugin-development
```

Or reference directly in your agent configuration.

## Usage

Once installed, the agent will automatically use this skill when:

- Creating new Curity plugins
- Implementing authenticators or authentication actions
- Working with Curity SDK APIs
- Writing Spock tests for plugins
- Configuring Gradle builds for plugins

### Example Prompts

**Create a new authenticator:**
```
Create a backchannel authenticator for Vipps CIBA integration
```

**Implement request handler:**
```
Add a multi-screen OTP authenticator with SMS verification
```

**Add tests:**
```
Create Spock tests for the authentication handler
```

**Configure build:**
```
Add Gradle tasks for deploying to local Curity server
```

## Knowledge Base

This skill includes comprehensive documentation:

### Plugin Types
- **Authenticators** (`plugin-type-authenticator.md`) - Interactive authentication flows with views
- **Backchannel Authenticators** (`plugin-type-backchannel-authenticator.md`) - CIBA/API-only authentication
- **Authentication Actions** (`plugin-type-authentication-action.md`) - Post-authentication enrichment and policies

### Core Topics
- **Plugin System** (`plugin-system.md`) - Architecture and lifecycle
- **Request Handlers** (`request-handlers.md`) - GET/POST handling and validation
- **SDK Services** (`sdk-services.md`) - SessionManager, ExceptionFactory, HttpClient, etc.
- **Attributes** (`attributes.md`) - SubjectAttributes, ContextAttributes, AuthenticationAttributes
- **Templating** (`templating.md`) - Velocity templates and localization
- **Testing** (`testing.md`) - Spock framework with mocking and assertions
- **Build & Deployment** (`build-and-deployment.md`) - Gradle configuration and deployment
- **Quick Reference** (`quick-reference.md`) - Task-oriented index
- **Recipes** (`recipes.md`) - Complete working examples

## Patterns and Best Practices

The agent follows these Curity-specific patterns:

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

### Logging
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

The skill has access to Curity Plugin SDK documentation and follows:

- **API Version**: 10.6.1+
- **Java Version**: 21
- **Kotlin Version**: 2.0+
- **Test Framework**: Spock 2.3

## Source of Truth Priority

When generating code, the agent follows this priority:

1. **Current repository** (existing implementations, tests, build files)
2. **Curity Plugin SDK Javadocs** (exact interfaces, types, signatures)
3. **Agent docs** (`agent-docs/*.md`) - patterns, skeletons, recipes
4. **Public example plugins** (pattern/reference only; does not assume API details)

## Default Workflow

For plugin development requests, the agent:

1. Identifies plugin type and relevant documentation
2. Proposes implementation plan with files to change
3. Confirms required SDK interfaces/types
4. Implements minimal viable change set
5. Adds/updates unit tests
6. Provides verification commands and summary

## Security Gates

The agent enforces non-negotiable rules:

- Validate all external inputs in request handlers
- Never log secrets, tokens, or sensitive personal data
- Avoid new dependencies unless explicitly required
- Preserve configuration compatibility
- Provide tests for new behavior and bug fixes

## Version History

### 1.0.0 (2026-02-03)
- Initial release
- Authenticator plugins documentation
- Backchannel authenticator plugins documentation  
- Authentication action plugins documentation
- Testing guide with Spock
- Build and deployment documentation

## Contributing

To extend this skill:

1. Add new documentation to `agent-docs/`
2. Follow existing markdown structure
3. Include code skeletons and examples
4. Update `quick-reference.md` index
5. Add working examples when applicable

## Support

- **Documentation**: See `agent-docs/` directory
- **Examples**: See `vipps-authenticator/` directory
- **SDK Reference**: https://curity.io/docs/idsvr-java-plugin-sdk/
- **Curity Support**: https://support.curity.io/

## License

Apache License 2.0 - See LICENSE file for details.
