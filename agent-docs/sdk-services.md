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
sessionManager.put(Attribute.of("otp", "123456"))
sessionManager.put(Attribute.of("username", username))

// Store complex objects
val account = accountManager.getByUserName(username)
sessionManager.put(Attribute.of("account", MapAttributeValue.of(account.toMap)))

// Retrieve values
val otp = sessionManager.get("otp")?.getAttributeValue()?.getValue() as String?
val username = sessionManager.get("username")?.getAttributeValue()?.getValue() as String?

// Retrieve attributes for direct reuse
val accountAttribute = sessionManager.get("account")  // Returns Attribute, not unwrapped value

// Clean up session data
sessionManager.remove("otp")
sessionManager.remove("username")
sessionManager.remove("account")
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

## 5. Logging

**Return Type**: Returns `Attributes` which can be cast to `AuthenticationAttributes` for use in `AuthenticationResult` constructor.

**Example Usage**:
```kotlin
// With subject attributes only
val subjectAttributes = SubjectAttributes.of(
    listOf(
        Attribute.of("subject", username),
        Attribute.of("email", email),
        Attribute.of("account", accountAttributes as AttributeValue)
    )
)
val authAttributes = AuthenticationAttributes.of(subjectAttributes)
return Optional.of(AuthenticationResult(authAttributes as AuthenticationAttributes))

// With both subject and context attributes
val contextAttributes = ContextAttributes.of(
    Attribute.of("authenticationMethod", "sms-otp"),
    Attribute.of("authenticationTime", System.currentTimeMillis())
**See also**: `attributes.md` for complete attribute framework documentation, `recipes.md` for working examples.

## 5. Logging

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

## 6. Server-Provided Dependencies

All plugins should mark these as `compileOnly` in Gradle:

```groovy
dependencies {
    // Curity Identity Server SDK (provided at runtime)
    compileOnly 'se.curity.identityserver:identityserver.sdk:10.6.1'
    
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
