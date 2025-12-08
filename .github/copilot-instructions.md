# GitHub Copilot Instructions – Curity Plugin Development

You are assisting with **Curity Identity Server plugin development**.

Follow these rules:

1. Prefer **correctness, security, and maintainability** over brevity or cleverness.
2. Use the **Curity Plugin SDK** concepts and patterns described in:
   - `INSTRUCTIONS.md`
   - `agent-docs/plugin-system.md`
   - `agent-docs/plugin-types.md`
   - `agent-docs/plugin-type-*.md`
   - `agent-docs/ui-and-screens.md`
   - `agent-docs/sdk-services.md`
   - `agent-docs/templating.md`
   - `agent-docs/configuration.md`
3. When generating plugin code:
   - Choose the appropriate plugin type from `agent-docs/plugin-types.md`
   - Start from the recommended skeleton in the relevant `plugin-type-*.md` file
   - Use SDK services according to `agent-docs/sdk-services.md`
   - Use the templating system as described in `agent-docs/templating.md`
4. Explain your reasoning briefly when the user asks “how” or “why”, but otherwise
   focus on producing complete, high-quality code.

If something is unclear, prefer to say what’s missing and refer to the relevant
`agent-docs/*.md` file instead of guessing Curity-specific APIs.
