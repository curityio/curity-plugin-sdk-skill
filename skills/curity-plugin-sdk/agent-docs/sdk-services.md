# SDK Services – Agent Reference

This document describes commonly used SDK services and types available for plugin development.

## Accessing SDK Services

**All SDK services are available** to plugins through dependency injection. To use any service:

1. Add a getter method to your plugin's Configuration interface
2. Annotate it with `@Description` to document its purpose
3. Use the correct return type from the SDK packages

**Example**:
```kotlin
import se.curity.identityserver.sdk.config.Configuration
import se.curity.identityserver.sdk.config.annotation.Description
import se.curity.identityserver.sdk.service.ExceptionFactory
import se.curity.identityserver.sdk.service.SessionManager
import se.curity.identityserver.sdk.service.credential.UserCredentialManager
import se.curity.identityserver.sdk.service.AccountManager
import se.curity.identityserver.sdk.service.sms.SmsSender
import se.curity.identityserver.sdk.service.authentication.AuthenticatorInformationProvider

interface MyAuthenticatorConfig : Configuration {
    @Description("Factory for creating SDK exceptions")
    fun getExceptionFactory(): ExceptionFactory
    
    @Description("Manager for session state")
    fun getSessionManager(): SessionManager
    
    @Description("User credential verification service")
    fun getUserCredentialManager(): UserCredentialManager
    
    @Description("Account lookup service")
    fun getAccountManager(): AccountManager
    
    @Description("SMS sending service")
    fun getSmsSender(): SmsSender
    
    @Description("Authenticator information provider")
    fun getAuthenticatorInformationProvider(): AuthenticatorInformationProvider
}
```

The Curity runtime automatically injects the correct implementation when your plugin is loaded. You can then access these services in your request handlers via the configuration object.

## 1. Credential Management

### UserCredentialManager

**Package**: `se.curity.identityserver.sdk.service.credential.UserCredentialManager`

**Purpose**: Verify and update user credentials (passwords).

**Injection**: Add to configuration interface:
```kotlin
@Description("The user credential manager to use for verifying passwords.")
fun getUserCredentialManager(): UserCredentialManager
```

**Key Methods**:

- `verify(SubjectAttributes, String): CredentialVerificationResult` - Verifies a password for a subject
- `update(SubjectAttributes, String): CredentialUpdateResult` - Updates a password for a subject

**Example Usage**:
```kotlin
val userCredentialManager = config.getUserCredentialManager()
val subjectAttributes = SubjectAttributes.of(username)
val verificationResult = userCredentialManager.verify(subjectAttributes, password)

if (verificationResult is CredentialVerificationResult.Accepted) {
    // Credential verification successful
    return Optional.of(AuthenticationResult(username))
}
```

**Important Notes**:
- `UserCredentialManager` replaces the deprecated `CredentialManager`
- Extends `CredentialVerifier` interface
- Returns sealed interface types (`Accepted` or `Rejected`)
- Always use `SubjectAttributes` to identify the user
- `SubjectAttributes` must contain an attribute named `subject` (use `SubjectAttributes.of(username)` shorthand)

### CredentialVerificationResult

**Package**: `se.curity.identityserver.sdk.service.credential.CredentialVerificationResult`

**Type**: Sealed interface with two subtypes

**Subtypes**:
- `CredentialVerificationResult.Accepted` - Password verification succeeded
- `CredentialVerificationResult.Rejected` - Password verification failed

**Usage Pattern**:
```kotlin
when (verificationResult) {
    is CredentialVerificationResult.Accepted -> {
        // Handle successful verification
    }
    is CredentialVerificationResult.Rejected -> {
        // Handle failed verification
    }
}

// Or use instanceof check
if (verificationResult is CredentialVerificationResult.Accepted) {
    // Success
}
```

## 2. Exception Handling

### ExceptionFactory

**Package**: `se.curity.identityserver.sdk.service.ExceptionFactory`

**Purpose**: Create properly typed exceptions for the Curity server to handle.

**Injection**: Add to configuration interface:
```kotlin
@Description("Factory for creating SDK exceptions")
fun getExceptionFactory(): ExceptionFactory
```

**Key Methods**:

- `internalServerException(ErrorCode): RuntimeException`
- `internalServerException(ErrorCode, String): RuntimeException`
- `badRequestException(ErrorCode): RuntimeException`
- `badRequestException(ErrorCode, String): RuntimeException`
- `unauthorizedException(ErrorCode): RuntimeException`
- `forbiddenException(ErrorCode): RuntimeException`
- `externalServiceException(String): RuntimeException`
- `configurationException(String): RuntimeException`
- `redirectException(String): RuntimeException`

**Since 11.5.0** — verbatim error-code overloads. The `String errorCode` is reported to the client exactly as
given (rather than mapped from an `ErrorCode`), which is the only way to emit a non-default OAuth `error` value
like `invalid_target` or `invalid_grant`:

- `badRequestException(String errorCode, String errorDescription, boolean exposeErrorDescription): RuntimeException`
- `unauthorizedException(String errorCode, String errorDescription, boolean exposeErrorDescription): RuntimeException`
- `forbiddenException(String errorCode, String errorDescription, boolean exposeErrorDescription): RuntimeException`

The error code must be RFC 6749 §5.2-safe (no spaces, quotes, backslash, control chars) or the call throws.
`exposeErrorDescription=true` returns the description to the client even when the profile does not enable
`expose-detailed-error-messages`; `false` logs it at debug level instead. For a standard OAuth code, pass
`OAuthError.<NAME>.name()`.

**Example Usage**:
```kotlin
try {
    // Perform operation
} catch (e: Exception) {
    logger.error("Error occurred", e)
    throw config.getExceptionFactory().internalServerException(
        ErrorCode.EXTERNAL_SERVICE_ERROR,
        "Authentication error occurred"
    )
}
```

**Important Notes**:
- Always use `ErrorCode` enum for typed error codes
- Provide meaningful error detail messages
- The server will map these to appropriate HTTP responses
- In OAuth contexts, errors map to standard OAuth error codes
- **In token procedures, always _throw_ from `ExceptionFactory`.** Never return an error as a `ResponseModel` —
  whatever `run` returns is used as the *successful* token response, so `problemResponseModel(...)` yields
  HTTP 200 with `{}` and `mapResponseModel(Map.of("error", ...))` yields HTTP 200 with the wrong envelope.
  See `plugin-type-token-procedure.md` §9.

### ErrorCode

**Package**: `se.curity.identityserver.sdk.errors.ErrorCode`

**Type**: Enum

**Common Values**:
- `GENERIC_ERROR` - Generic error (maps to `server_error` in OAuth)
- `EXTERNAL_SERVICE_ERROR` - Error contacting external service
- `CONFIGURATION_ERROR` - Configuration caused an error
- `PLUGIN_ERROR` - Plugin caused an error
- `AUTHENTICATION_FAILED` - Authentication failed (maps to `login_required` in OAuth)
- `INVALID_INPUT` - Input validation failed
- `ACCESS_DENIED` - Access denied
- `MISSING_PARAMETERS` - Required parameter missing

**Usage**:
```kotlin
throw exceptionFactory.internalServerException(
    ErrorCode.EXTERNAL_SERVICE_ERROR,
    "Failed to connect to user database"
)
```

## 3. Session Management

### SessionManager

**Package**: `se.curity.identityserver.sdk.service.SessionManager`

**Purpose**: Store and retrieve data across multiple requests within the same authentication flow.

**Injection**: Add to configuration interface:
```kotlin
@Description("Session manager for storing state")
fun getSessionManager(): SessionManager
```

**Key Methods**:
- `put(Attribute): void` - Store an attribute in the session
- `get(String): Attribute?` - Retrieve an attribute by name (returns null if not found)
- `remove(String): void` - Remove an attribute from the session

**Example Usage**:
```kotlin
val sessionManager = config.getSessionManager()

// Store simple values
sessionManager.put(Attribute.of("otp-session-key", "123456"))
sessionManager.put(Attribute.of("username-session-key", username))

// Store complex objects
val account = accountManager.getByUserName(username)
sessionManager.put(Attribute.of("account-session-key", MapAttributeValue.of(account.toMap)))

// Retrieve values
val otp: String? = sessionManager.get("otp-session-key")?.getAttributeValue()?.getValue() as String?
val username: String? = sessionManager.get("username-session-key")?.getAttributeValue()?.getValue() as String?

// Retrieve attributes for direct reuse
val accountAttribute: Attribute = sessionManager.get("account-session-key")  // Returns Attribute, not unwrapped value

// Clean up session data
sessionManager.remove("otp-session-key")
sessionManager.remove("username-session-key")
sessionManager.remove("account-session-key")
```

**Important Notes**:
- Session is scoped to the authentication flow (survives across screens/handlers)
- Use `Attribute.of()` to create key-value pairs for storage
- Complex objects like `AccountAttributes` must be cast to `AttributeValue`
- `get()` returns the full `Attribute` object, not just the value
- Attributes from session can be reused directly in `SubjectAttributes.of()`
- Always clean up session data when authentication completes or fails

**Best Practices**:
- Store minimal data needed between handlers
- Prefer storing complete objects over IDs to avoid extra database lookups
- Clear sensitive data (like OTPs) from session after use
- Use descriptive attribute names for maintainability

**See also**: `recipes.md` for complete multi-screen flow examples using SessionManager.

## 4. User and Account Management

### AccountManager

**Package**: `se.curity.identityserver.sdk.service.AccountManager`

**Purpose**: Look up user account information including emails, phone numbers, and other attributes.

**Injection**: Add to configuration interface:
```kotlin
@Description("Account lookup service")
fun getAccountManager(): AccountManager
```

**Key Methods**:
- `getByUserName(String): AccountAttributes?` - Retrieve account by username (returns null if not found)
- `getByEmail(String): AccountAttributes?` - Retrieve account by email
- `getById(String): AccountAttributes?` - Retrieve account by account ID

**Example Usage**:
```kotlin
val accountManager = config.getAccountManager()
val account = accountManager.getByUserName(username)
    ?: throw config.getExceptionFactory().internalServerException(
        ErrorCode.EXTERNAL_SERVICE_ERROR,
        "Account not found for user: $username"
    )

// Access account attributes
val email = account.emails?.primaryOrFirst?.value
val phoneNumber = account.phoneNumbers?.primaryOrFirst?.significantValue
val accountId = account.userName
```

**Important Notes**:
- Returns `AccountAttributes` which can be stored in session or added to authentication result
- Account data includes emails, phone numbers, custom attributes
- Cast to `AttributeValue` when storing in session: `account as AttributeValue`
- Use null-safe operators when accessing nested properties (emails, phone numbers may be null)

**See also**: `attributes.md` for AccountAttributes structure, `recipes.md` for complete examples.

## 5. SMS Services

### SmsSender

**Package**: `se.curity.identityserver.sdk.service.sms.SmsSender`

**Purpose**: Send SMS messages to phone numbers.

**Injection**: Add to configuration interface:
```kotlin
@Description("SMS sending service")
fun getSmsSender(): SmsSender
```

**Key Methods**:
- `sendSms(String, String): void` - Send SMS to phone number with message content

**Example Usage**:
```kotlin
val smsSender = config.getSmsSender()
val phoneNumber = "+46701234567"
val message = "Your verification code is: 123456"

smsSender.sendSms(phoneNumber, message)
logger.info("Sent SMS to: {}", phoneNumber)
```

**Important Notes**:
- Phone number should be in E.164 format (e.g., `+46701234567`)
- SMS sending is asynchronous - method returns immediately
- Configure SMS provider in Curity admin console before using
- Handle exceptions for delivery failures

**See also**: `recipes.md` Recipe 2 for SMS OTP flow example.

## 6. Authentication Context

### AuthenticatorInformationProvider

**Package**: `se.curity.identityserver.sdk.service.authentication.AuthenticatorInformationProvider`

**Purpose**: Provide information about the current authenticator context, including URIs for building redirects.

**Injection**: Add to configuration interface:
```kotlin
@Description("Authenticator information provider")
fun getAuthenticatorInformationProvider(): AuthenticatorInformationProvider
```

**Key Methods**:
- `getFullyQualifiedAuthenticationUri(): String` - Get the base URI for this authenticator

**Example Usage**:
```kotlin
val authInfo = config.getAuthenticatorInformationProvider()
val baseUri = authInfo.getFullyQualifiedAuthenticationUri()
// Returns: "https://example.com/authn/authenticate/my-authenticator"

// Build redirect to next screen
val otpUri = baseUri + "/otp"
throw config.getExceptionFactory().redirectException(otpUri)
```

**Important Notes**:
- Returns fully qualified URI including protocol, domain, and path
- Use for building redirects in multi-screen authenticators
- Ensures correct URLs regardless of deployment configuration

**See also**: `recipes.md` Recipe 2 for multi-screen redirect example.

## 7. Authentication Results

### AuthenticationResult

**Package**: `se.curity.identityserver.sdk.authentication.AuthenticationResult`

**Purpose**: Represent the result of a successful authentication.

**Constructors**:
```kotlin
AuthenticationResult(String username)
AuthenticationResult(AuthenticationAttributes attributes)
```

**Example Usage**:
```kotlin
// Simple authentication result with username
return Optional.of(AuthenticationResult(username))

// Authentication result with additional attributes (see attributes.md for details)
val subjectAttributes = SubjectAttributes.of(username)
val authenticationAttributes = AuthenticationAttributes.of(subjectAttributes)
return Optional.of(AuthenticationResult(authenticationAttributes as AuthenticationAttributes))
```

**Important Notes**:
- Return `Optional.empty()` when authentication is not complete (e.g., displaying a form)
- Return `Optional.of(AuthenticationResult(...))` when authentication succeeds
- The username becomes the authenticated subject
- See `attributes.md` for comprehensive guide on SubjectAttributes, AuthenticationAttributes, and ContextAttributes

**See also**: `attributes.md` for complete attribute framework documentation, `recipes.md` for working examples.

## 8. Logging

### SLF4J Logger

**Package**: `org.slf4j.Logger` and `org.slf4j.LoggerFactory`

**Version**: 2.0.12 (provided by server)

**Setup**:
```kotlin
import org.slf4j.Logger
import org.slf4j.LoggerFactory

class MyRequestHandler {
    private val logger: Logger = LoggerFactory.getLogger(MyRequestHandler::class.java)
}
```

**Usage Levels**:
```kotlin
logger.trace("Detailed trace message")
logger.debug("Debug information: {}", value)
logger.info("Informational message: {}", username)
logger.warn("Warning about potential issue")
logger.error("Error occurred during operation: {}", username, exception)
```

**Best Practices**:
- Use parameterized logging with `{}` placeholders
- Log exceptions with stack traces using the exception parameter
- Use appropriate log levels:
  - `debug` - Development debugging information
  - `info` - Normal operation milestones
  - `warn` - Potential problems
  - `error` - Errors requiring attention
- Never log sensitive data (passwords, tokens, etc.)
- Log at entry/exit of important operations for troubleshooting

## 9. HTTP and External Service Calls

### HttpClient

**Package**: `se.curity.identityserver.sdk.service.HttpClient`

**Scope**: `@ConfigurationScope` (safe for ManagedObject use)

**Purpose**: Low-level HTTP client configured by the server admin (scheme, TLS trust, client certificates, proxy).

**Injection**:
```java
@Description("HTTP client for external calls")
HttpClient getHttpClient();
```

**Key Methods**:
- `getScheme(): String` — Returns `"https"` or `"http"`
- `request(URI): HttpRequest.Builder` — Start building a request to the given URI
- `cookieManager(): CookieManager` — Returns the client's cookie manager (nullable)

**Example Usage** (Java):
```java
HttpRequest req = httpClient.request(URI.create(metadataUrl))
        .accept("application/json")
        .timeout(Duration.ofSeconds(10))
        .get();

HttpResponse resp = req.response();
if (resp.statusCode() == 200) {
    String body = resp.body(HttpResponse.asString());
}
```

**Important Notes**:
- `HttpClient` is configured in the Curity admin console with TLS settings, proxy, timeouts
- Use `WebServiceClient` / `WebServiceClientFactory` for higher-level API calls (see below)
- Use `HttpClient` directly when you need full control over URI construction

### WebServiceClientFactory

**Package**: `se.curity.identityserver.sdk.service.WebServiceClientFactory`

**Purpose**: Factory for creating `WebServiceClient` instances.

**Injection**:
```kotlin
@Description("Factory for creating web service clients")
fun getWebServiceClientFactory(): WebServiceClientFactory
```

**Key Methods**:
- `create(URI): WebServiceClient` — Create client from full URI (host, port, path extracted)
- `create(HttpClient): WebServiceClient` — Create client using admin-configured HTTP client
- `create(HttpClient, int): WebServiceClient` — Create client with specific port

**Example Usage** (Kotlin):
```kotlin
val client = webServiceClientFactory.create(httpClient)
    .withHost("api.example.com")
    .withPath("/v1/users")

val response = client.request()
    .accept("application/json")
    .header("Authorization", "Bearer $token")
    .get()
    .response()

val body = response.body(HttpResponse.asString())
```

### WebServiceClient

**Package**: `se.curity.identityserver.sdk.service.WebServiceClient`

**Scope**: `@ConfigurationScope`

**Purpose**: Higher-level HTTP client for making API calls against a configured base URI. Provides fluent API for path, query, and host manipulation.

**Key Methods** (all return new `WebServiceClient`):
- `withPath(String): WebServiceClient` — Append path to base URI
- `withHost(String): WebServiceClient` — Set the target host
- `withQuery(String): WebServiceClient` — Set raw query string
- `withQueries(Map<String, Collection<String>>): WebServiceClient` — Add query parameters
- `withFragment(String): WebServiceClient` — Set URI fragment
- `request(): HttpRequest.Builder` — Start building the HTTP request

**Example Usage** (Java):
```java
WebServiceClient client = webServiceClientFactory.create(httpClient)
        .withHost("api.vipps.no");

// GET with query parameters
HttpResponse getResponse = client
        .withPath("/v2/payments/" + orderId)
        .request()
        .accept("application/json")
        .header("Authorization", "Bearer " + accessToken)
        .get()
        .response();

// POST with JSON body
HttpResponse postResponse = client
        .withPath("/v2/payments")
        .request()
        .contentType("application/json")
        .body(HttpRequest.fromString(json.toJson(requestBody)))
        .post()
        .response();
```

**Important Notes**:
- Each `with*` method returns a **new** instance (immutable builder pattern)
- Prefer `WebServiceClient` over raw `HttpClient` for API integration
- When creating from `HttpClient`, you almost always need `.withHost(...)` afterward

## 10. JSON Serialization

### Json

**Package**: `se.curity.identityserver.sdk.service.Json`

**Scope**: `@ConfigurationScope`

**Purpose**: Serialize and deserialize JSON. Handles Maps, objects, arrays, and SDK Attributes.

**Injection**:
```kotlin
@Description("JSON helper service")
fun getJson(): Json
```

**Key Methods**:
- `toJson(Map<?, ?>): String` — Serialize map to JSON (nulls excluded)
- `toJson(Map<?, ?>, boolean): String` — Serialize with null control
- `toJson(Object): String` — Serialize any object
- `fromJson(String): Map<String, Object>` — Deserialize JSON to map (nulls excluded)
- `fromJson(String, boolean): Map<String, Object>` — Deserialize with null control
- `fromJson(String, Class<T>): T` — Deserialize JSON to typed object (since 7.6)
- `fromJsonArray(String): List<?>` — Deserialize JSON array
- `toAttributes(String): Attributes` — Deserialize JSON to SDK Attributes
- `fromAttributes(Attributes): String` — Serialize SDK Attributes to JSON

**Example Usage** (Java):
```java
Json json = config.getJson();

// Serialize
Map<String, Object> data = Map.of("username", "john", "active", true);
String jsonString = json.toJson(data);

// Deserialize
Map<String, Object> parsed = json.fromJson(responseBody);
String username = (String) parsed.get("username");

// Typed deserialization (Kotlin data class or Java POJO)
MyResponse response = json.fromJson(body, MyResponse.class);
```

**Important Notes**:
- Numbers are narrowed to integers when possible; doubles for decimals, longs for overflow
- `fromJson(String, Class<T>)` does not call constructors — use for request model objects that will be validated
- Throws `Json.JsonException` on syntax errors or serialization failures

## 11. Email Services

### EmailSender

**Package**: `se.curity.identityserver.sdk.service.EmailSender`

**Scope**: `@ConfigurationScope`

**Purpose**: Send emails using the configured email provider.

**Injection**:
```java
@Description("Email provider for sending notifications")
Optional<EmailSender> getEmailSender();
```

**Key Methods**:
- `sendEmail(String recipient, Email email, String resource)` — Send email using a resource template
- `sendEmail(String recipient, Email email, String resource, String templateArea)` — Send with custom template area

**Example Usage** (Java):
```java
EmailSender emailSender = config.getEmailSender()
        .orElseThrow(() -> exceptionFactory.configurationException("Email sender not configured"));

Email email = Email.create("subject-key")
        .withModel(Map.of("username", username, "link", resetLink));

emailSender.sendEmail(recipientAddress, email, "forgot-password/email");
```

**Important Notes**:
- Email provider must be configured in the Curity admin console
- The `resource` parameter identifies the email template (Velocity)
- Declare as `Optional<EmailSender>` when email is not required for all flows
- Email templates follow the same Velocity templating as authenticator views

## 12. Nonce Tokens

### NonceTokenIssuer

**Package**: `se.curity.identityserver.sdk.service.NonceTokenIssuer`

**Purpose**: Issue and introspect single-use nonce tokens. Nonces are cryptographic tokens that can only be consumed once.

**Injection**:
```java
NonceTokenIssuer getNonceTokenIssuer();
```

**Key Methods**:
- `issue(TokenAttributes): String` — Issue a nonce token, returns the token string
- `introspect(String): Optional<TokenAttributes>` — Introspect a token; returns empty if not found, already consumed, or wrong type

**Example Usage** (Java — password reset flow):
```java
NonceTokenIssuer nonceIssuer = config.getNonceTokenIssuer();

// Issue a nonce with custom attributes
TokenAttributes attrs = TokenAttributes.of(
        Attribute.of("subject", username),
        Attribute.of("purpose", "password-reset")
);
String nonce = nonceIssuer.issue(attrs);

// Later — introspect and consume the nonce (single-use)
Optional<TokenAttributes> result = nonceIssuer.introspect(nonce);
if (result.isPresent()) {
    String subject = result.get().get("subject").getValue().toString();
    // Proceed with password reset
} else {
    // Nonce is invalid, expired, or already consumed
}
```

**Important Notes**:
- Nonces are **single-use**: calling `introspect()` consumes the token
- Use for password reset links, email verification, account activation
- The server handles token expiration and storage
- Throws `TokenIssuerException` if issuance fails

## 13. User Preferences

### UserPreferenceManager

**Package**: `se.curity.identityserver.sdk.service.UserPreferenceManager`

**Purpose**: Manage the username cookie and locale preferences. Helps pre-populate login forms.

**Injection**:
```java
UserPreferenceManager getUserPreferenceManager();
```

**Key Methods**:
- `saveUsername(String): void` — Store username in the username cookie
- `getUsername(): String` — Retrieve stored username (nullable)
- `saveLocales(String): void` — Store locale preferences (BCP47 tags, space-separated)
- `getLocales(): String` — Retrieve locale preferences (nullable)
- `hasUserConsentedToNonEssentialCookies(): Boolean` — Check cookie consent (nullable — null = no decision yet)
- `isNonEssentialCookieAllowed(): boolean` — Whether non-essential cookies can be used (considers consent + system settings)
- `clearState(): void` — Clear all persisted user preference state

**Example Usage** (Java):
```java
UserPreferenceManager prefs = config.getUserPreferenceManager();

// Pre-populate username field on GET
String savedUsername = prefs.getUsername();

// Save username after successful login
if (prefs.isNonEssentialCookieAllowed()) {
    prefs.saveUsername(authenticatedUsername);
}
```

## 14. Data Storage

### Bucket

**Package**: `se.curity.identityserver.sdk.service.Bucket`

**Scope**: `@ConfigurationScope`

**Purpose**: Generic key-value store for arbitrary data, organized by subject and purpose. Useful for storing plugin state, counters, or temporary data outside of the session.

**Injection**:
```kotlin
@Description("Data bucket for plugin state")
fun getBucket(): Bucket
```

**Key Methods**:
- `getAttributes(String subject, String purpose): Map<String, Object>` — Get stored attributes (empty map if not found or expired)
- `getBucket(String subject, String purpose): GetBucketResult` — Get with detailed result (Success/Error, since 11.0)
- `storeAttributes(String subject, String purpose, Map<String, Object>): Map<String, Object>` — Store (overwrites existing)
- `storeAttributes(String subject, String purpose, Map<String, Object>, Instant expiresAt): Map<String, Object>` — Store with expiration (since 11.0)
- `clearBucket(String subject, String purpose): boolean` — Delete all attributes for subject/purpose

**Example Usage** (Kotlin):
```kotlin
val bucket = config.getBucket()

// Store data
bucket.storeAttributes("user123", "mfa-state", mapOf(
    "enrollmentComplete" to true,
    "deviceId" to "abc123"
))

// Store with expiration
bucket.storeAttributes("user123", "otp-challenge", mapOf(
    "code" to "123456",
    "attempts" to 0
), Instant.now().plusSeconds(300))

// Retrieve
val data = bucket.getAttributes("user123", "mfa-state")
val enrolled = data["enrollmentComplete"] as? Boolean ?: false

// Clean up
bucket.clearBucket("user123", "otp-challenge")
```

**Important Notes**:
- Unlike `SessionManager`, `Bucket` is not scoped to an authentication flow
- Data persists across sessions and requests
- Subject + purpose together form the key
- Ideal for cross-flow or long-lived state (e.g., device enrollment)

## 15. Rate Limiting

### Throttler

**Package**: `se.curity.identityserver.sdk.service.Throttler`

**Scope**: `@ConfigurationScope`

**Purpose**: Throttle actions based on a key and purpose. Useful for rate-limiting login attempts, OTP sends, or API calls.

**Injection**:
```java
@Description("Rate limiting service")
Throttler getThrottler();
```

**Key Methods**:
- `shouldThrottle(String key, String purpose): ThrottlingResult` — Check and update throttle state
- `shouldThrottle(String key, String purpose, Instant time): ThrottlingResult` — Check at specific time
- `clear(String key, String purpose): boolean` — Reset throttle state

**Result Types** (sealed interface `ThrottlingResult`):
- `ThrottlingResult.Throttled` — Action should be throttled; has `purpose` and `nextAttempt` (nullable Instant)
- `ThrottlingResult.NotThrottled.INSTANCE` — Action is allowed

**Example Usage** (Java):
```java
Throttler throttler = config.getThrottler();

Throttler.ThrottlingResult result = throttler.shouldThrottle(username, "login-attempt");
if (result instanceof Throttler.ThrottlingResult.Throttled throttled) {
    logger.warn("User {} is being throttled until {}", username, throttled.nextAttempt());
    throw exceptionFactory.badRequestException(ErrorCode.ACCESS_DENIED, "Too many attempts. Try again later.");
}

// Proceed with login...

// On successful login, clear the throttle
throttler.clear(username, "login-attempt");
```

## 16. System Information

### SystemInformationProvider

**Package**: `se.curity.identityserver.sdk.service.SystemInformationProvider`

**Scope**: `@ConfigurationScope`

**Purpose**: Provides system-level information about the running Curity instance.

**Injection**:
```java
SystemInformationProvider getSystemInformationProvider();
```

**Key Methods**:
- `getEntityId(): String` — The Entity ID (organization name) from configuration
- `getEnvironmentBaseUri(): URI` — The base URI of the server (nullable if environment name not configured)
- `getZone(): String` — The zone ID the node operates in
- `isNonEssentialCookieOptIn(): boolean` — Whether non-essential cookies require explicit consent

## 17. OAuth Request Context

### OriginalQueryExtractor

**Package**: `se.curity.identityserver.sdk.service.OriginalQueryExtractor`

**Purpose**: Extract query parameters from the original OAuth/authentication request that initiated the current flow.

**Key Methods**:
- `getClientId(): String` — The `client_id` from the original request (nullable)
- `getServiceProviderId(): String` — The service provider ID (nullable)
- `getQueryParameterValue(String): String` — Any arbitrary query parameter value (nullable)
- `getAuthorizationRequestQueryParameterValue(String): String` — From the authorization request (nullable)
- `getAuthenticationRequestQueryParameterValue(String): String` — From the authentication request (nullable)

**Example Usage**:
```java
OriginalQueryExtractor queryExtractor = config.getOriginalQueryExtractor();

String clientId = queryExtractor.getClientId();
String acrValues = queryExtractor.getQueryParameterValue("acr_values");
String loginHint = queryExtractor.getQueryParameterValue("login_hint");
```

### RequestingOAuthClient

**Package**: `se.curity.identityserver.sdk.service.RequestingOAuthClient`

**Purpose**: Provides the OAuth client that initiated the current authentication flow.

**Key Methods**:
- `getClient(): OAuthClient` — The requesting OAuth client (nullable — null if not an OAuth-initiated flow)

**Example Usage**:
```java
RequestingOAuthClient requestingClient = config.getRequestingOAuthClient();
OAuthClient client = requestingClient.getClient();
if (client != null) {
    Set<String> audiences = client.getAudiences();
    Set<String> scopes = client.getScopeNames();
}
```

## 18. Token Issuance and Introspection

These services are primarily used in **Token Procedure** plugins but can also be injected by other plugin types.

### Token Issuers (`se.curity.identityserver.sdk.service.issuer`)

| Interface | Purpose |
|-----------|---------|
| `AccessTokenIssuer` | Issue access tokens |
| `IdTokenIssuer` | Issue OpenID Connect ID tokens |
| `RefreshTokenIssuer` | Issue refresh tokens |
| `DelegationIssuer` | Issue delegation tokens |
| `NonceIssuer` | Issue nonces |
| `DeviceCodeIssuer` | Issue device codes |
| `DefaultJwtAccessTokenIssuerProvider` | Get the default JWT access token issuer |

### Token Introspecters (`se.curity.identityserver.sdk.service.introspecter`)

| Interface | Purpose |
|-----------|---------|
| `AccessTokenIntrospecter` | Introspect access tokens |
| `IdTokenIntrospecter` | Introspect ID tokens |
| `RefreshTokenIntrospecter` | Introspect refresh tokens |
| `NonceIntrospecter` | Introspect nonces |

**Injection** — use `@DefaultService` when the default issuer should be used:
```java
@DefaultService
DefaultJwtAccessTokenIssuerProvider getJwtAccessTokenIssuerProvider();
```

See `plugin-type-token-procedure.md` for complete examples.

## 19. Cryptographic Services (`se.curity.identityserver.sdk.service.crypto`)

These services provide access to keys and cryptographic operations configured in the server.

| Interface | Purpose |
|-----------|---------|
| `AsymmetricSigningCryptoStore` | Sign data with asymmetric keys |
| `AsymmetricSignatureVerificationCryptoStore` | Verify asymmetric signatures |
| `AsymmetricEncryptionCryptoStore` | Encrypt with asymmetric keys |
| `AsymmetricDecryptionCryptoStore` | Decrypt with asymmetric keys |
| `SymmetricSigningCryptoStore` | Sign with symmetric keys |
| `SymmetricSignatureVerificationCryptoStore` | Verify symmetric signatures |
| `SymmetricEncryptionCryptoStore` | Encrypt with symmetric keys |
| `SymmetricDecryptionCryptoStore` | Decrypt with symmetric keys |
| `ServerTrustCryptoStore` | Server trust operations |
| `SignerTrustCryptoStore` | Signer trust operations |
| `ClientKeyCryptoStore` | Client key operations |

## 20. Server-Provided Dependencies

All plugins should mark these as `compileOnly` in Gradle:

```groovy
dependencies {
    // Curity Identity Server SDK (provided at runtime)
    compileOnly 'se.curity.identityserver:identityserver.sdk:11.1.0'
    
    // SLF4J API (provided at runtime)
    compileOnly 'org.slf4j:slf4j-api:2.0.12'
    
    // Kotlin standard library (provided at runtime)
    compileOnly 'org.jetbrains.kotlin:kotlin-stdlib:2.2.0'
    
    // Jakarta Validation (provided at runtime)
    compileOnly 'jakarta.validation:jakarta.validation-api:3.0.0'
}
```

These dependencies are provided by the Curity Identity Server at runtime and should **not** be bundled with your plugin JAR.

**See also**: `build-and-deployment.md` for complete dependency and build configuration.
