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

See: `plugin-type-authenticator.md` for implementation details and code skeletons.

## 2. Backchannel Authenticator Plugins

Use when:

- **Clients integrate with Curity using CIBA** (Client Initiated Backchannel Authentication).
- Authentication with external provider uses **any backchannel protocol** (not necessarily CIBA).
- **Cannot show screens** to the client - pure API-based flow.
- Authentication happens **out-of-band** (mobile push, separate device, polling).

Key responsibilities:

- Initiate authentication with external provider.
- Poll or receive notifications about authentication status.
- Map external states to Curity states (STARTED, SUCCEEDED, FAILED, EXPIRED).
- Return authentication attributes on success.

See: `plugin-type-backchannel-authenticator.md` for implementation details and code skeletons.

## 3. Authentication Action Plugins

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

## 4. Event Listener Plugins

Use when:

- **Audit logging** — recording authentication events, token issuance, account changes.
- **External notifications** — sending events to SIEM systems, webhooks, or message queues.
- **Analytics** — tracking login patterns, failure rates, or usage metrics.
- **Side effects** — triggering downstream actions when specific events occur.

Key responsibilities:

- Implement `EventListener<T>` to receive events of type T.
- Use `getEventType()` to filter which events are received (or return `Event.class` for all).
- Event listeners are passive observers — they cannot modify the flow.
- Must be thread-safe (only `@ConfigurationScope` services available).

See: `plugin-type-event-listener.md` for implementation details and code skeletons.

## 5. Token Procedure Plugins

Use when:

- **Customizing token issuance** for a specific OAuth grant type.
- **Implementing token exchange** (RFC 8693) with an external identity provider.
- **Adding custom claims** to access tokens or ID tokens.
- **Controlling delegation and scope** behavior during token issuance.

Key responsibilities:

- Override the token issuance procedure for one or more OAuth flows.
- Validate incoming tokens (for token exchange).
- Use token issuers/introspecters to produce or inspect tokens.
- Return a ResponseModel with the token response.

See: `plugin-type-token-procedure.md` for implementation details and code skeletons.

## 6. Other Plugin Types (Not Yet Fully Documented)

The SDK supports additional plugin types. These are listed here for awareness — detailed implementation guides are not yet available, but the SDK interfaces follow the same descriptor + configuration + implementation pattern.

| Plugin Type | Descriptor Interface | Use Case |
|-------------|---------------------|----------|
| Data Access Provider | `DataAccessProviderPluginDescriptor` | Custom storage backends for accounts, tokens, credentials, sessions |
| Claims Provider | `ClaimsProviderPluginDescriptor` | Custom claims sources for OAuth/OIDC tokens |
| Consentor | `ConsentorPluginDescriptor` | Custom consent flow UI and logic |
| Authorization Manager | `AuthorizationManagerPluginDescriptor` | Custom authorization policies (OAuth, SCIM, GraphQL) |
| Application | `ApplicationPluginDescriptor` | Custom web endpoints |
| Alarm Handler | `AlarmHandlerPluginDescriptor` | Custom alarm/alert handling |
| Email Provider | `EmailProviderPluginDescriptor` | Custom email delivery (SMTP, API-based) |
| SMS Provider | `SmsPluginDescriptor` | Custom SMS delivery |
| SAML Attribute Provider | `SamlAttributeProviderPluginDescriptor` | SAML attribute mapping |
| Signing Consentor | `SigningConsentorPluginDescriptor` | Signing-related consent flows |
