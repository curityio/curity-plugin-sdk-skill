# Authenticator Plugins – Agent Reference

Use this document when generating code for **authenticator plugins**.

## 1. When to Use

Implement an authenticator plugin when:

- You need to support a new login mechanism.
- You integrate with a custom or proprietary identity provider.
- The built in authenticators is not enough

## 2. Core Interfaces and Types

### Request Handlers

An authenticator plugin request handler implements the interface `se.curity.identityserver.sdk.authentication.AuthenticatorRequestHandler`

See `request-handlers.md`.


When generating code, always implement the appropriate interface and follow the lifecycle
described in `plugin-system.md`.

## 3. Canonical Skeleton

Below is a **generic skeleton**. Replace the TODO sections and types with the real Curity SDK names.

```kotlin
// TODO: Replace package and imports with plugin-specific ones
package com.example.curity.plugin.auth;

import org.slf4j.Logger
import org.slf4j.LoggerFactory
import se.curity.identityserver.sdk.authentication.AuthenticationResult
import se.curity.identityserver.sdk.authentication.AuthenticatorRequestHandler
import se.curity.identityserver.sdk.web.Request
import se.curity.identityserver.sdk.web.Response
import se.curity.identityserver.sdk.web.ResponseModel
import java.util.Optional

class UsernamePasswordRequestHandler(private val config: UsernamePasswordAuthenticatorConfig) :
    AuthenticatorRequestHandler<UsernamePasswordRequestModel>
{
    private val logger: Logger = LoggerFactory.getLogger(UsernamePasswordRequestHandler::class.java)

    override fun preProcess(request: Request, response: Response): UsernamePasswordRequestModel
    {
        return UsernamePasswordRequestModel(request)
    }

    override fun get(requestModel: UsernamePasswordRequestModel, response: Response): Optional<AuthenticationResult>
    {
        logger.debug("GET request - displaying login form")

        // Adding the template to the response
        val viewData = emptyMap<String, Any>()
        response.setResponseModel(
            ResponseModel.templateResponseModel(viewData, "authenticate/get"),
            Response.ResponseModelScope.ANY
        )

        return Optional.empty()
    }

    override fun post(requestModel: UsernamePasswordRequestModel, response: Response): Optional<AuthenticationResult>
    {
        // Cast to POST model - validation already performed
        val postModel = requestModel as UsernamePasswordPostRequestModel
        val username = postModel.validatedUsername!!
        val password = postModel.validatedPassword!!
        
        // Verify credentials using SDK service
        val isValid = config.getUserCredentialManager()
            .verify(username, password, SubjectAttributes.of(username))
        
        if (!isValid) {
            throw config.getExceptionFactory()
                .badRequestException(ErrorCode.INVALID_CREDENTIALS)
        }
        
        // Return successful authentication
        return Optional.of(AuthenticationResult(username))
        
        // Note: Do NOT wrap this logic in try-catch - let SDK exceptions propagate
    }
}
```

## 4. Multi-Screen Authenticators

For authenticators requiring multiple steps (e.g., password → OTP verification), register multiple routes in the descriptor:

```kotlin
class SmsOtpAuthenticatorPluginDescriptor : AuthenticatorPluginDescriptor<SmsOtpAuthenticatorConfig>
{
    override fun getAuthenticationRequestHandlerTypes(): Map<String, Class<out AuthenticatorRequestHandler<*>>>
    {
        return mapOf(
            "index" to PasswordRequestHandler::class.java,
            "verify-otp" to OtpRequestHandler::class.java
        )
    }
}
```

**Navigate between screens using `redirectException`:**

```kotlin
// In PasswordRequestHandler after password verification:
val sessionManager = config.getSessionManager()
sessionManager.put(Attribute.of("otp", generatedOtp))
sessionManager.put(Attribute.of("username", username))

val authInfo = config.getAuthenticatorInformationProvider()
val redirectUrl = "${authInfo.getFullyQualifiedAuthenticationUri()}/verify-otp"
throw config.getExceptionFactory().redirectException(redirectUrl)
```

See `request-handlers.md` for detailed exception handling and SDK service usage.
