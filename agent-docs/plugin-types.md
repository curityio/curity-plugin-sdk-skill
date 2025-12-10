# Plugin Types Overview – Agent Reference

Use this file to decide which plugin type to generate based on the user’s request.

> NOTE: This file describes the plugin types conceptually. For implementation details
> and skeletons, see the respective `plugin-type-*.md` files.

## 1. Authenticator Plugins

Use when:

- Implementing a **custom authentication method**.
- Integrating an **external identity provider**.

Key responsibilities:

- Receive authentication input (credentials, tokens, etc.).
- Validate or exchange credentials.
- Produce an authenticated subject or identity, or fail.

See: `plugin-type-authentication.md` for implementation details and code skeletons.

## 2. Authentication Action Plugins

Use when:

- Running logic **after authentication** but before completing the flow.
- **Enriching** authentication with additional attributes or context.
- **Denying** authentication based on policies or conditions.
- **Prompting users** for additional input (terms acceptance, attribute collection, etc.).

Key responsibilities:

- Receive an already-authenticated context with `AuthenticationAttributes`.
- Either allow the authentication to continue (success), deny it (failed), or prompt for user input (pending).
- Optionally enrich the authentication with additional attributes.

See: `plugin-type-authentication-action.md` for implementation details and code skeletons.
