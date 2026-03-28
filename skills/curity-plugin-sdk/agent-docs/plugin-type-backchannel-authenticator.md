# Backchannel Authenticator Plugins – Agent Reference

Use this document when generating code for **backchannel authenticator plugins** (CIBA - Client Initiated Backchannel Authentication).

**Reference implementation**: [Curity-PS/vipps-backchannel](https://github.com/Curity-PS/vipps-backchannel) — a polling-based backchannel authenticator integrating with the Vipps API.

## 1. When to Use

Implement a backchannel authenticator plugin when:

- **Clients integrate with Curity using CIBA** (Client Initiated Backchannel Authentication)
- The plugin communicates with external provider using **any backchannel protocol** (polling, push, proprietary API)
- **Cannot show screens or views** to the client - pure API-based flow only
- Authentication happens **out-of-band** (e.g., mobile push notification, separate device)
- The flow follows a **polling** or **push notification** pattern
- Examples: Vipps CIBA, BankID backchannel, Freja eID CIBA mode, custom proprietary backchannel flows

## 2. Core Interfaces and Types

### Plugin Descriptor

A backchannel authenticator plugin descriptor extends `BackchannelAuthenticatorPluginDescriptor`:

```kotlin
import se.curity.identityserver.sdk.plugin.descriptor.BackchannelAuthenticatorPluginDescriptor
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticationHandler

class MyBackchannelAuthenticatorPluginDescriptor : 
    BackchannelAuthenticatorPluginDescriptor<MyAuthenticatorConfig>
{
    override fun getPluginImplementationType(): String = "my-ciba-authenticator"
    
    override fun getBackchannelAuthenticationHandlerType(): Class<out BackchannelAuthenticationHandler> =
        MyBackchannelAuthenticationHandler::class.java
    
    override fun getConfigurationType(): Class<out MyAuthenticatorConfig> =
        MyAuthenticatorConfig::class.java
}
```

### Backchannel Authentication Handler

The handler implements `BackchannelAuthenticationHandler` interface:

```kotlin
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticationHandler
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticationRequest
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticationResult
import se.curity.identityserver.sdk.authentication.BackchannelStartAuthenticationResult
import java.util.Optional

class MyBackchannelAuthenticationHandler(private val config: MyAuthenticatorConfig) :
    BackchannelAuthenticationHandler
{
    override fun startAuthentication(
        authReqId: String,
        authRequest: BackchannelAuthenticationRequest
    ): BackchannelStartAuthenticationResult
    {
        // Initiate authentication with external provider
        // Return ok() or error()
    }
    
    override fun checkAuthenticationStatus(authReqId: String): Optional<BackchannelAuthenticationResult>
    {
        // Poll or check status
        // Return result with state and attributes
    }
    
    override fun cancelAuthenticationRequest(authReqId: String)
    {
        // Cancel and clean up
    }
}
```

## 3. Key Concepts

### Authentication Request

The `BackchannelAuthenticationRequest` contains:

- **subject**: The user identifier (login hint) - phone number, email, user ID, etc.
- **bindingMessage**: Optional message to display to the user for confirmation

```kotlin
override fun startAuthentication(
    authReqId: String,
    authRequest: BackchannelAuthenticationRequest
): BackchannelStartAuthenticationResult
{
    val loginHint = authRequest.subject      // e.g., "+4712345678"
    val bindingMessage = authRequest.bindingMessage  // e.g., "Login to MyApp"
    
    // Initiate with external provider
    val externalAuthRef = externalProvider.startAuth(loginHint, bindingMessage)
    
    // Store reference in session for polling
    config.sessionManager.put(Attribute.of("auth-ref", externalAuthRef))
    
    return BackchannelStartAuthenticationResult.ok()
}
```

### Authentication States

Map external provider responses to `BackchannelAuthenticatorState`:

- **STARTED**: Authentication initiated, waiting for user action
- **SUCCEEDED**: User authenticated successfully
- **FAILED**: User rejected or authentication failed
- **EXPIRED**: Request timed out
- **UNKNOWN**: Unable to determine state

```kotlin
override fun checkAuthenticationStatus(authReqId: String): Optional<BackchannelAuthenticationResult>
{
    val externalAuthRef = config.sessionManager.get("auth-ref")
    val response = externalProvider.checkStatus(externalAuthRef)
    
    return when (response.status) {
        "PENDING" -> Optional.of(
            BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.STARTED)
        )
        "APPROVED" -> {
            val attributes = extractAuthenticationAttributes(response)
            Optional.of(
                BackchannelAuthenticationResult(attributes, BackchannelAuthenticatorState.SUCCEEDED)
            )
        }
        "REJECTED" -> Optional.of(
            BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.FAILED)
        )
        "EXPIRED" -> Optional.of(
            BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.EXPIRED)
        )
        else -> Optional.of(
            BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.UNKNOWN)
        )
    }
}
```

### Authentication Attributes

On successful authentication, return `AuthenticationAttributes`:

```kotlin
private fun extractAuthenticationAttributes(response: ProviderResponse): AuthenticationAttributes
{
    val subject = response.userId  // or extract from ID token
    
    val subjectAttributes = SubjectAttributes.of(
        subject,
        Attributes.of(
            Attribute.of("name", response.name),
            Attribute.of("email", response.email)
        )
    )
    
    return AuthenticationAttributes.of(
        subjectAttributes,
        ContextAttributes.empty()
    )
}
```

## 4. Canonical Skeleton

### Complete Handler Implementation

```kotlin
package com.example.curity.plugin.backchannel

import org.slf4j.Logger
import org.slf4j.LoggerFactory
import se.curity.identityserver.sdk.attribute.Attribute
import se.curity.identityserver.sdk.attribute.AuthenticationAttributes
import se.curity.identityserver.sdk.attribute.ContextAttributes
import se.curity.identityserver.sdk.attribute.SubjectAttributes
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticationHandler
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticationRequest
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticationResult
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticatorState
import se.curity.identityserver.sdk.authentication.BackchannelStartAuthenticationResult
import java.util.Optional

class MyBackchannelAuthenticationHandler(private val config: MyAuthenticatorConfig) :
    BackchannelAuthenticationHandler
{
    private val _logger: Logger = LoggerFactory.getLogger(MyBackchannelAuthenticationHandler::class.java)
    private val _sessionManager = config.sessionManager
    private val _externalClient = MyExternalClient(config)

    override fun startAuthentication(
        authReqId: String,
        authRequest: BackchannelAuthenticationRequest
    ): BackchannelStartAuthenticationResult
    {
        _logger.debug("Starting backchannel authentication for subject: ${authRequest.subject}")

        return try
        {
            // Call external provider
            val externalAuthRef = _externalClient.initiateAuthentication(
                loginHint = authRequest.subject,
                bindingMessage = authRequest.bindingMessage
            )

            // Store reference for polling
            _sessionManager.put(Attribute.of("external-auth-ref", externalAuthRef))
            
            _logger.debug("Authentication started with external reference: $externalAuthRef")
            BackchannelStartAuthenticationResult.ok()
        }
        catch (e: Exception)
        {
            _logger.error("Failed to start authentication: ${e.message}", e)
            BackchannelStartAuthenticationResult.error(
                "server_error",
                "Failed to initiate authentication"
            )
        }
    }

    override fun checkAuthenticationStatus(authReqId: String): Optional<BackchannelAuthenticationResult>
    {
        _logger.trace("Checking authentication status for: $authReqId")

        val externalAuthRef = _sessionManager.get("external-auth-ref")
            ?: return Optional.of(BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.UNKNOWN))

        return try
        {
            val response = _externalClient.pollStatus(externalAuthRef.value.toString())
            Optional.of(mapToAuthenticationResult(response))
        }
        catch (e: Exception)
        {
            _logger.warn("Error checking status: ${e.message}", e)
            Optional.of(BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.UNKNOWN))
        }
    }

    override fun cancelAuthenticationRequest(authReqId: String)
    {
        _logger.debug("Canceling authentication: $authReqId")
        
        val externalAuthRef = _sessionManager.get("external-auth-ref")
        if (externalAuthRef != null) {
            try {
                _externalClient.cancelAuthentication(externalAuthRef.value.toString())
            } catch (e: Exception) {
                _logger.warn("Error canceling authentication: ${e.message}", e)
            }
        }
        
        // Clean up session
        _sessionManager.remove("external-auth-ref")
    }

    private fun mapToAuthenticationResult(response: ExternalResponse): BackchannelAuthenticationResult
    {
        return when (response.status)
        {
            "PENDING", "STARTED" -> {
                BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.STARTED)
            }
            "APPROVED", "COMPLETED" -> {
                val subject = response.userId
                val attributes = AuthenticationAttributes.of(
                    SubjectAttributes.of(subject),
                    ContextAttributes.empty()
                )
                
                // Clean up on success
                _sessionManager.remove("external-auth-ref")
                
                BackchannelAuthenticationResult(attributes, BackchannelAuthenticatorState.SUCCEEDED)
            }
            "REJECTED", "DENIED" -> {
                _sessionManager.remove("external-auth-ref")
                BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.FAILED)
            }
            "EXPIRED", "TIMEOUT" -> {
                _sessionManager.remove("external-auth-ref")
                BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.EXPIRED)
            }
            else -> {
                _logger.warn("Unknown status: ${response.status}")
                BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.UNKNOWN)
            }
        }
    }
}
```

### Configuration Interface

```kotlin
package com.example.curity.plugin.backchannel

import se.curity.identityserver.sdk.config.Configuration
import se.curity.identityserver.sdk.config.annotation.Description
import se.curity.identityserver.sdk.service.ExceptionFactory
import se.curity.identityserver.sdk.service.HttpClient
import se.curity.identityserver.sdk.service.Json
import se.curity.identityserver.sdk.service.SessionManager
import se.curity.identityserver.sdk.service.WebServiceClientFactory
import java.util.Optional

interface MyAuthenticatorConfig : Configuration
{
    @get:Description("HTTP client for external provider communication")
    val httpClient: Optional<HttpClient>

    @get:Description("Client ID for external provider")
    val clientId: String

    @get:Description("Client secret for external provider")
    val clientSecret: String

    val sessionManager: SessionManager
    val exceptionFactory: ExceptionFactory
    val webServiceClientFactory: WebServiceClientFactory
    val json: Json
}
```

## 5. Best Practices

### Session Management

Use `SessionManager` to store state between polling requests:

```kotlin
// Store authentication reference
config.sessionManager.put(Attribute.of("auth-ref", externalAuthRef))

// Retrieve on polling
val authRef = config.sessionManager.get("auth-ref")

// Clean up when done
config.sessionManager.remove("auth-ref")
```

**CRITICAL**: Always clean up session data when authentication completes (success, failure, or expiration).

### Error Handling

Return appropriate error codes in `BackchannelStartAuthenticationResult`:

```kotlin
return try {
    // Start authentication
    BackchannelStartAuthenticationResult.ok()
} catch (e: Exception) {
    when (e) {
        is InvalidUserException -> BackchannelStartAuthenticationResult.error(
            "invalid_request", 
            "User not found"
        )
        is ProviderUnavailableException -> BackchannelStartAuthenticationResult.error(
            "server_error",
            "Provider temporarily unavailable"
        )
        else -> BackchannelStartAuthenticationResult.error(
            "server_error",
            "Authentication failed"
        )
    }
}
```

### Logging

Use appropriate log levels:

- **TRACE**: Detailed polling responses
- **DEBUG**: Start/stop operations, state transitions
- **INFO**: Important state changes (user approved/rejected)
- **WARN**: Recoverable errors, unexpected states
- **ERROR**: Critical failures

```kotlin
_logger.trace("Polling response: $response")
_logger.debug("Starting authentication for user: $subject")
_logger.info("User approved authentication: $authReqId")
_logger.warn("Unknown status received: $status")
_logger.error("Failed to start authentication: ${e.message}", e)
```

### State Mapping

Be explicit when mapping external provider states:

```kotlin
private fun mapExternalState(status: String): BackchannelAuthenticatorState {
    return when (status) {
        // Map all known states explicitly
        "PENDING", "WAITING", "INITIATED" -> BackchannelAuthenticatorState.STARTED
        "COMPLETED", "APPROVED", "SUCCESS" -> BackchannelAuthenticatorState.SUCCEEDED
        "REJECTED", "DENIED", "CANCELLED" -> BackchannelAuthenticatorState.FAILED
        "EXPIRED", "TIMEOUT" -> BackchannelAuthenticatorState.EXPIRED
        else -> {
            _logger.warn("Unmapped external state: $status")
            BackchannelAuthenticatorState.UNKNOWN
        }
    }
}
```

## 6. CIBA Standard Patterns

### Polling Pattern (Most Common)

```kotlin
override fun checkAuthenticationStatus(authReqId: String): Optional<BackchannelAuthenticationResult>
{
    // Poll the external token endpoint
    val authRef = _sessionManager.get("auth-req-id")
    val response = _client.pollTokenEndpoint(authRef)
    
    return when (response.error) {
        null -> {
            // Success - tokens received
            val attributes = extractFromTokens(response.idToken, response.accessToken)
            Optional.of(BackchannelAuthenticationResult(attributes, BackchannelAuthenticatorState.SUCCEEDED))
        }
        "authorization_pending" -> {
            // Still waiting
            Optional.of(BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.STARTED))
        }
        "slow_down" -> {
            // Polling too fast - still waiting
            Optional.of(BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.STARTED))
        }
        "expired_token" -> {
            Optional.of(BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.EXPIRED))
        }
        "access_denied" -> {
            Optional.of(BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.FAILED))
        }
        else -> {
            _logger.warn("Unexpected CIBA error: ${response.error}")
            Optional.of(BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.UNKNOWN))
        }
    }
}
```

### Push Notification Pattern

If the external provider supports push notifications (less common):

```kotlin
// Store callback URL during initialization
override fun startAuthentication(
    authReqId: String,
    authRequest: BackchannelAuthenticationRequest
): BackchannelStartAuthenticationResult
{
    val callbackUrl = config.callbackBaseUrl + "/backchannel-callback"
    
    val externalAuthRef = _client.initiateWithCallback(
        loginHint = authRequest.subject,
        notificationUri = callbackUrl
    )
    
    _sessionManager.put(Attribute.of("auth-ref", externalAuthRef))
    _sessionManager.put(Attribute.of("notification-pending", true))
    
    return BackchannelStartAuthenticationResult.ok()
}

// In checkStatus, return STARTED until notification received
override fun checkAuthenticationStatus(authReqId: String): Optional<BackchannelAuthenticationResult>
{
    val notificationReceived = _sessionManager.get("notification-received")
    
    if (notificationReceived == null) {
        // Still waiting for push notification
        return Optional.of(BackchannelAuthenticationResult(null, BackchannelAuthenticatorState.STARTED))
    }
    
    // Notification received, retrieve final status
    // ... rest of implementation
}
```

## 7. Common Patterns

### Extract Subject from ID Token

```kotlin
private fun extractSubjectFromIdToken(idToken: String): String
{
    // Parse JWT
    val parts = idToken.split(".")
    if (parts.size != 3) {
        throw config.exceptionFactory.internalServerException(
            ErrorCode.EXTERNAL_SERVICE_ERROR,
            "Invalid ID token format"
        )
    }
    
    // Decode payload
    val payload = String(java.util.Base64.getUrlDecoder().decode(parts[1]))
    val claims = config.json.toAttributes(payload)
    
    // Extract subject
    return claims.getOptionalValue("sub")
        ?: throw config.exceptionFactory.internalServerException(
            ErrorCode.EXTERNAL_SERVICE_ERROR,
            "Missing subject in ID token"
        )
}
```

### Multi-Provider Configuration

```kotlin
enum class ProviderEnvironment {
    @Description("Test environment")
    TEST,
    
    @Description("Production environment")
    PRODUCTION;
    
    fun getHost(): String {
        return when (this) {
            TEST -> "test.provider.com"
            PRODUCTION -> "api.provider.com"
        }
    }
    
    fun getBackchannelEndpoint(): String {
        return "/ciba/v1/backchannel/authentication"
    }
    
    fun getTokenEndpoint(): String {
        return "/ciba/v1/token"
    }
}
```

### Access Token Management

```kotlin
private var _accessToken: String? = null
private var _tokenExpiry: Long = 0

private fun getAccessToken(): String {
    val now = System.currentTimeMillis()
    
    if (_accessToken == null || now >= _tokenExpiry) {
        val response = _client.requestAccessToken(
            clientId = config.clientId,
            clientSecret = config.clientSecret
        )
        _accessToken = response.accessToken
        _tokenExpiry = now + (response.expiresIn * 1000) - 60000  // 1 min buffer
    }
    
    return _accessToken!!
}
```

## 8. Testing

### Basic Handler Test with Spock

```groovy
package com.example.curity.plugin.backchannel

import se.curity.identityserver.sdk.attribute.Attribute
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticationRequest
import se.curity.identityserver.sdk.authentication.BackchannelAuthenticatorState
import spock.lang.Specification

class MyBackchannelAuthenticationHandlerSpec extends Specification
{
    MyBackchannelAuthenticationHandler handler
    MyAuthenticatorConfig config
    SessionManager sessionManager

    def setup()
    {
        config = Mock(MyAuthenticatorConfig)
        sessionManager = Mock(SessionManager)
        config.sessionManager >> sessionManager
        
        handler = new MyBackchannelAuthenticationHandler(config)
    }

    def "startAuthentication stores auth reference in session"()
    {
        given: "an authentication request"
        def authReqId = "curity-auth-123"
        def authRequest = Mock(BackchannelAuthenticationRequest)
        authRequest.subject >> "user@example.com"
        authRequest.bindingMessage >> null

        when: "starting authentication"
        def result = handler.startAuthentication(authReqId, authRequest)

        then: "session stores the reference"
        1 * sessionManager.put({ it.name == "external-auth-ref" })
        result.successful
    }

    def "checkAuthenticationStatus returns STARTED when pending"()
    {
        given: "a session with auth reference"
        def authReqId = "curity-auth-123"
        sessionManager.get("external-auth-ref") >> Attribute.of("ref", "external-123")

        when: "checking status"
        def result = handler.checkAuthenticationStatus(authReqId)

        then: "status is STARTED"
        result.isPresent()
        result.get().state == BackchannelAuthenticatorState.STARTED
    }

    def "cancelAuthenticationRequest cleans up session"()
    {
        given: "a handler"
        def authReqId = "curity-auth-123"

        when: "canceling authentication"
        handler.cancelAuthenticationRequest(authReqId)

        then: "session is cleaned up"
        1 * sessionManager.remove("external-auth-ref")
    }
}
```

## 9. Key Differences from Regular Authenticators

| Aspect | Regular Authenticator | Backchannel Authenticator |
|--------|----------------------|---------------------------|
| **Views** | Has templates and request handlers | No views - pure API |
| **User Interaction** | Direct (form submission) | Indirect (mobile app, etc.) |
| **Request Handlers** | `AuthenticatorRequestHandler` | `BackchannelAuthenticationHandler` |
| **Flow** | Synchronous | Asynchronous (polling) |
| **Descriptor** | `AuthenticatorPluginDescriptor` | `BackchannelAuthenticatorPluginDescriptor` |
| **State** | Immediate result | Polled state transitions |
| **Session Use** | Optional | Required (store auth refs) |

## 10. Common Pitfalls

❌ **Forgetting to clean up session data**
```kotlin
// BAD - leaks session data
if (status == "APPROVED") {
    return BackchannelAuthenticationResult(attributes, BackchannelAuthenticatorState.SUCCEEDED)
}
```

✅ **Always clean up**
```kotlin
// GOOD - cleans up session
if (status == "APPROVED") {
    _sessionManager.remove("auth-ref")
    return BackchannelAuthenticationResult(attributes, BackchannelAuthenticatorState.SUCCEEDED)
}
```

❌ **Not handling all external states**
```kotlin
// BAD - unmapped states cause issues
return when (status) {
    "APPROVED" -> BackchannelAuthenticatorState.SUCCEEDED
    else -> BackchannelAuthenticatorState.UNKNOWN
}
```

✅ **Map all known states explicitly**
```kotlin
// GOOD - explicit mapping with logging
return when (status) {
    "PENDING" -> BackchannelAuthenticatorState.STARTED
    "APPROVED" -> BackchannelAuthenticatorState.SUCCEEDED
    "REJECTED" -> BackchannelAuthenticatorState.FAILED
    "EXPIRED" -> BackchannelAuthenticatorState.EXPIRED
    else -> {
        _logger.warn("Unmapped state: $status")
        BackchannelAuthenticatorState.UNKNOWN
    }
}
```

❌ **Wrapping exceptions unnecessarily**
```kotlin
// BAD - breaks SDK control flow
override fun startAuthentication(...): BackchannelStartAuthenticationResult {
    return try {
        // logic
    } catch (e: Exception) {
        throw config.exceptionFactory.internalServerException(e.message)
    }
}
```

✅ **Return error results instead**
```kotlin
// GOOD - use result type for errors
override fun startAuthentication(...): BackchannelStartAuthenticationResult {
    return try {
        // logic
        BackchannelStartAuthenticationResult.ok()
    } catch (e: Exception) {
        _logger.error("Start failed: ${e.message}", e)
        BackchannelStartAuthenticationResult.error("server_error", "Failed to start")
    }
}
```

## 11. See Also

- `plugin-type-authenticator.md` - For frontchannel authenticators
- `sdk-services.md` - For SessionManager and other SDK services
- `testing.md` - For comprehensive testing with Spock
- `recipes.md` - For complete working examples
