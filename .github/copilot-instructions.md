# GitHub Copilot Instructions – Curity Plugin Development

You are assisting with **Curity Identity Server plugin development**.

## Priorities
1. Prefer **correctness, security, and maintainability** over brevity.
2. **Do not invent Curity-specific APIs.** If a required type/method/config key is not present in the repo or referenced docs, stop and ask for missing context (or point to where it should be found).

## Source of truth (in priority order)
1. The **current repository** (existing implementations, tests, build files).
2. **Curity Plugin SDK Javadocs** (exact interfaces, types, signatures).
3. `skills/curity-plugin-development/agent-docs/*.md` (Curity patterns, skeletons, recipes).
4. Public example plugins (pattern/reference only; do not assume API details).

## Where to look first (routing)
- **Choose plugin type / extension point:** `skills/curity-plugin-development/agent-docs/plugin-types.md`
- **Authenticator:** `skills/curity-plugin-development/agent-docs/plugin-type-authenticator.md`, `skills/curity-plugin-development/agent-docs/request-handlers.md`, `skills/curity-plugin-development/agent-docs/sdk-services.md`
- **Authentication Action:** `skills/curity-plugin-development/agent-docs/plugin-type-authentication-action.md`, `skills/curity-plugin-development/agent-docs/templating.md`, `skills/curity-plugin-development/agent-docs/attributes.md`
- **Testing:** `skills/curity-plugin-development/agent-docs/testing.md`
- **Build & deployment:** `skills/curity-plugin-development/agent-docs/build-and-deployment.md`
- **End-to-end examples:** `skills/curity-plugin-development/agent-docs/recipes.md`
- **Quick lookup:** `skills/curity-plugin-development/agent-docs/quick-reference.md`
- **Detailed instructions:** `skills/curity-plugin-development/INSTRUCTIONS.md`

## Default workflow (apply unless instructed otherwise)
1. Restate the goal and identify the **plugin type** and relevant docs.
2. Propose a short plan with the **files you will change**.
3. Confirm any required SDK interfaces/types against the **Javadocs or repo**.
4. Implement the smallest viable change set.
5. Add/update unit tests (follow `skills/curity-plugin-development/agent-docs/testing.md`).
6. Run verification (or provide exact commands if execution is not available).
7. Provide a final summary and test instructions.

## Security and quality gates (non-negotiable)
- Validate and sanitize all external inputs in request handlers.
- Never log secrets, tokens, credentials, or sensitive personal data; redact where needed.
- Avoid new dependencies unless explicitly required; call out any additions clearly.
- Preserve configuration compatibility; document migrations if keys/structure change.
- Provide tests for new behavior and bug fixes.

## Required response format
- **Plan**
- **Patch** (diffs or clearly delimited file contents)
- **How to test** (exact commands)
- **Notes** (config changes, risks, follow-ups)

If something is unclear, state what is missing and point to the most relevant `skills/curity-plugin-development/agent-docs/*.md` file (or the Javadocs) rather than guessing.
