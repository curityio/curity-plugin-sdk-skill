# GitHub Copilot Instructions – Curity Plugin Development

You are assisting with **Curity Identity Server plugin development**.

Follow these rules:

1. Prefer **correctness, security, and maintainability** over brevity or cleverness.
2. Use the **Curity Plugin SDK** concepts and patterns described in:
   - `agent-docs/plugin-system.md` - Core plugin architecture and lifecycle
   - `agent-docs/plugin-types.md` - Available plugin types overview
   - `agent-docs/plugin-type-authenticator.md` - Authenticator plugin patterns
   - `agent-docs/plugin-type-authentication-action.md` - Authentication action patterns
   - `agent-docs/request-handlers.md` - Request handler implementation and validation
   - `agent-docs/sdk-services.md` - Available SDK services (credential management, sessions, etc.)
   - `agent-docs/templating.md` - Velocity template structure and localization
   - `agent-docs/attributes.md` - Attributes framework and data handling
   - `agent-docs/testing.md` - Unit testing with Spock framework
   - `agent-docs/build-and-deployment.md` - Gradle build configuration and deployment
   - `agent-docs/recipes.md` - Complete working examples
   - `agent-docs/quick-reference.md` - Quick lookup guide
3. When generating plugin code:
   - Choose the appropriate plugin type from `agent-docs/plugin-types.md`
   - Start from the recommended skeleton in the relevant `plugin-type-*.md` file
   - Use SDK services according to `agent-docs/sdk-services.md`
   - Follow request handler patterns from `agent-docs/request-handlers.md`
   - Use the templating system as described in `agent-docs/templating.md`
   - Include unit tests following patterns in `agent-docs/testing.md`
   - Configure build according to `agent-docs/build-and-deployment.md`
4. Explain your reasoning briefly when the user asks “how” or “why”, but otherwise
   focus on producing complete, high-quality code.

If something is unclear, prefer to say what’s missing and refer to the relevant
`agent-docs/*.md` file instead of guessing Curity-specific APIs.
