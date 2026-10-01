# Request handlers

Request handlers contain the core logic of your authenticator. They are responsible for:
- Handling GET and POST requests.
- Displaying views to the user using templates.
- Processing user input and making authentication decisions.
- Returning an `AuthenticationResult` upon success.

Request handler classes extend a plugin specific interface

## Request handler flow

- **`preProcess()`**: Called before `get` or `post`. Creates the request model by extracting data from the request. The request model is automatically validated using Jakarta Validation annotations.
- **`get()`**: Handles GET requests and is typically used to display a form to the user.
- **`post()`**: Handles POST requests, processes user input, and returns an `AuthenticationResult` if authentication is successful. Fields in the request model are already validated.

> **For non-GET/POST verbs**, use `HttpRequestHandler<T>` (an extension of `RequestHandler` with default `put`, `patch`, `delete`, `trace`, `options` methods) — typically in application plugins. Override only the verbs you support and emit a 405 for the rest. See [plugin-type-application.md](plugin-type-application.md#put-patch-delete--one-method-per-verb). `RequestHandler` itself (used by authenticators and actions) exposes only `get()` and `post()`.

### Request Model Validation

Request models should use **Jakarta Validation** annotations for automatic field validation. Validation only applies to POST requests since GET requests typically display empty forms.

**Jakarta Validation Annotations** (`jakarta.validation.constraints.*`):
- `@NotBlank` - String must not be null, empty, or whitespace-only
- `@NotNull` - Field must not be null
- `@NotEmpty` - Collection/array/string must not be empty
- `@Size(min, max)` - String/collection size constraints
- `@Pattern(regexp)` - String must match regex pattern
- `@Email` - Valid email format

**Best practice: Split request models for GET and POST**

```kotlin
import jakarta.validation.constraints.NotBlank
import se.curity.identityserver.sdk.web.Request

/**
 * Base request model for GET requests - no validation
 */
open class UsernamePasswordRequestModel(request: Request)
{
    val username: String? = request.getFormParameterValues("username").firstOrNull()
    val password: String? = request.getFormParameterValues("password").firstOrNull()
}

/**
 * POST request model with validation
 */
class UsernamePasswordPostRequestModel(request: Request) : UsernamePasswordRequestModel(request)
{
    @NotBlank(message = "validation.error.username.required")
    val validatedUsername: String? = username

    @NotBlank(message = "validation.error.password.required")
    val validatedPassword: String? = password
}
```

**In the request handler:**

```kotlin
override fun preProcess(request: Request, response: Response): UsernamePasswordRequestModel
{
    return if (request.isPostRequest) {
        UsernamePasswordPostRequestModel(request)  // Validated
    } else {
        UsernamePasswordRequestModel(request)      // Not validated
    }
}

override fun post(requestModel: UsernamePasswordRequestModel, response: Response): Optional<AuthenticationResult>
{
    // Cast to POST model - validation has already been performed
    val postModel = requestModel as UsernamePasswordPostRequestModel
    val username = postModel.validatedUsername!!
    val password = postModel.validatedPassword!!
    
    // Process authentication...
}
```

**Key points:**
- Validation happens automatically after `preProcess()` returns
- If validation fails, error messages are displayed to the user
- GET requests use the base model (no validation)
- POST requests use the subclass model (with validation)
- Request models should be **immutable** after instantiation
- Use message keys (e.g., `"validation.error.username.required"`) for internationalization
- Extract form parameters using `request.getFormParameterValues("name").firstOrNull()`

**Benefits of validation:**
- Declarative validation - no manual `if` checks needed
- Consistent error handling across all request handlers
- Automatic error message localization
- Type-safe access to validated fields

**See also**: `recipes.md` for complete working examples with validation.

### Example Handler:

```kotlin
// io/curity/example/username/UsernameRequestHandler.kt

import se.curity.identityserver.sdk.authentication.AuthenticationResult
import se.curity.identityserver.sdk.authentication.AuthenticatorRequestHandler
import se.curity.identityserver.sdk.web.Request
import se.curity.identityserver.sdk.web.Response
import se.curity.identityserver.sdk.web.ResponseModel
import java.util.Optional

class UsernameRequestHandler(config: UsernameAuthenticatorConfig) : AuthenticatorRequestHandler<UsernameRequestModel>
{
    override fun preProcess(request: Request, response: Response): UsernameRequestModel
    {
        return UsernameRequestModel(request)
    }

    override fun get(request: UsernameRequestModel, response: Response): Optional<AuthenticationResult>
    {
        // Render the 'get' template
        response.setResponseModel(ResponseModel.templateResponseModel(emptyMap(), "get"),
                                   Response.ResponseModelScope.ANY)
        return Optional.empty() // Authentication is not yet complete
    }

    override fun post(request: UsernameRequestModel, response: Response): Optional<AuthenticationResult>
    {
        // Request model is already validated - username is guaranteed to be non-blank
        val username = request.username!!
        
        // Authentication is successful
        return Optional.of(AuthenticationResult(username))
    }
}
```

## Exception Handling

**CRITICAL: Do not wrap request handler logic in try-catch blocks.**

The Curity SDK uses exceptions as control flow mechanisms. Wrapping them and re-throwing breaks SDK functionality:

```kotlin
// ❌ WRONG - breaks redirects and other SDK control flow
override fun post(requestModel: MyRequestModel, response: Response): Optional<AuthenticationResult>
{
    return try {
        // authentication logic
    } catch (e: Exception) {
        throw config.getExceptionFactory().internalServerException(e.message)
    }
}

// ✅ CORRECT - let SDK exceptions propagate naturally
override fun post(requestModel: MyRequestModel, response: Response): Optional<AuthenticationResult>
{
    // authentication logic - exceptions propagate to the server
    // Only catch specific exceptions that you can handle appropriately
}
```

**When to handle exceptions:**
- Only catch specific exceptions that represent recoverable conditions
- Throw meaningful exceptions for business logic errors (e.g., missing session data)
- Let SDK exceptions (like `redirectException`) propagate without catching

## Multi-Screen Flows and Redirects

For authenticators with multiple screens (e.g., password entry → OTP verification), use `ExceptionFactory.redirectException()` to navigate between routes:

```kotlin
class PasswordRequestHandler(private val config: SmsOtpAuthenticatorConfig) :
    AuthenticatorRequestHandler<PasswordRequestModel>
{
    override fun post(requestModel: PasswordRequestModel, response: Response): Optional<AuthenticationResult>
    {
        val postModel = requestModel as PasswordPostRequestModel
        val username = postModel.validatedUsername!!
        val password = postModel.validatedPassword!!
        
        // Verify password
        val isValid = config.getUserCredentialManager()
            .verify(username, password, SubjectAttributes.of(username))
        
        if (!isValid) {
            throw config.getExceptionFactory()
                .badRequestException(ErrorCode.INVALID_CREDENTIALS)
        }
        
        // Generate OTP and store in session
        val otp = String.format("%06d", Random.nextInt(0, 1000000))
        val sessionManager = config.getSessionManager()
        sessionManager.put(Attribute.of("otp", otp))
        sessionManager.put(Attribute.of("username", username))
        
        // Redirect to the verify-otp route (registered in descriptor)
        val authInfo = config.getAuthenticatorInformationProvider()
        val redirectUrl = "${authInfo.getFullyQualifiedAuthenticationUri()}/verify-otp"
        throw config.getExceptionFactory().redirectException(redirectUrl)
    }
}
```

**Key points:**
- Use `AuthenticatorInformationProvider.getFullyQualifiedAuthenticationUri()` to get the base authenticator URL
- Append the route name (e.g., `/verify-otp`) that matches the descriptor registration
- Throw `redirectException` - don't catch it
- Use `SessionManager` to pass state between screens with `Attribute.of(key, value)`
- Template rendering (`setResponseModel`) ≠ route navigation (use `redirectException`)

## Using SDK Services

Common SDK services are injected through the configuration interface:

### UserCredentialManager

Verify username/password credentials:

```kotlin
val credentialManager = config.getUserCredentialManager()
val isValid = credentialManager.verify(
    username,
    password,
    SubjectAttributes.of(username)
)
```

### SessionManager

Store and retrieve data across requests:

```kotlin
val sessionManager = config.getSessionManager()

// Store data
sessionManager.put(Attribute.of("otp", otpCode))
sessionManager.put(Attribute.of("username", username))

// Retrieve data
val otp = sessionManager.get("otp")?.getAttributeValue()?.getValue() as String?

// Remove data
sessionManager.remove("otp")
```

### ExceptionFactory

Create SDK exceptions for error conditions and control flow:

```kotlin
val exceptionFactory = config.getExceptionFactory()

// Business logic errors
throw exceptionFactory.badRequestException(ErrorCode.INVALID_CREDENTIALS)
throw exceptionFactory.internalServerException("Configuration error")

// Control flow - redirect to another route
throw exceptionFactory.redirectException(targetUrl)
```

### AuthenticatorInformationProvider

Get information about the current authenticator:

```kotlin
val authInfo = config.getAuthenticatorInformationProvider()

// Get the fully qualified URL to this authenticator
val baseUrl = authInfo.getFullyQualifiedAuthenticationUri()
// Example: https://example.com/authn/authenticate/sms-otp

// Construct URLs to other routes in this authenticator
val otpUrl = "${baseUrl}/verify-otp"
```

### AccountManager

Look up user account information:

```kotlin
val accountManager = config.getAccountManager()

// Get account by username
val account = accountManager.getByUserName(username)

// Access account attributes
if (account != null) {
    // Get primary phone number or first available
    val phoneNumber = account.getPhoneNumbers().getPrimaryOrFirst()?.getSignificantValue()
    
    // Get primary email or first available
    val email = account.getEmails().getPrimaryOrFirst()?.getSignificantValue()
}
```

### SmsSender

Send SMS messages to users:

```kotlin
val smsSender = config.getSmsSender()

// Send SMS with a message
val phoneNumber = "+1234567890"
val message = "Your verification code is: 123456"
smsSender.sendSms(phoneNumber, message)
```


