# Plugin Implementation Recipes

This document provides **end-to-end working examples** for common plugin implementation scenarios. These are copy-paste ready patterns that show complete integration across descriptor, configuration, handlers, and tests.

### Reference Plugins on GitHub

| Plugin Type | Repository |
|-------------|------------|
| Authenticator | [curityio/username-password-authenticator](https://github.com/curityio/username-password-authenticator) |
| Backchannel Authenticator | [Curity-PS/vipps-backchannel](https://github.com/Curity-PS/vipps-backchannel) |
| Authentication Action | [curityio/time-authentication-action](https://github.com/curityio/time-authentication-action) |
| Token Procedure | [curityio/external-idp-token-exchange](https://github.com/curityio/external-idp-token-exchange) |

---

## Basic Authenticators

### Recipe 1: Username/Password Authenticator

A simple single-screen authenticator that verifies username and password credentials.

**Descriptor** (`UsernamePasswordAuthenticatorDescriptor.kt`):
```kotlin
package io.curity.identityserver.plugin.usernamepassword.descriptor

import io.curity.identityserver.plugin.usernamepassword.UsernamePasswordAuthenticatorConfig
import io.curity.identityserver.plugin.usernamepassword.UsernamePasswordRequestHandler
import se.curity.identityserver.sdk.authentication.AuthenticatorRequestHandler
import se.curity.identityserver.sdk.plugin.descriptor.AuthenticatorPluginDescriptor

class UsernamePasswordAuthenticatorDescriptor : AuthenticatorPluginDescriptor<UsernamePasswordAuthenticatorConfig> {
    override fun getPluginImplementationType(): String = "username-password"
    
    override fun getConfigurationType(): Class<out UsernamePasswordAuthenticatorConfig> = 
        UsernamePasswordAuthenticatorConfig::class.java
    
    override fun getAuthenticationRequestHandlerTypes(): Map<String, Class<out AuthenticatorRequestHandler<*>>> {
        return mapOf("index" to UsernamePasswordRequestHandler::class.java)
    }
}
```

**Configuration** (`UsernamePasswordAuthenticatorConfig.kt`):
```kotlin
package io.curity.identityserver.plugin.usernamepassword

import se.curity.identityserver.sdk.config.Configuration
import se.curity.identityserver.sdk.config.annotation.Description
import se.curity.identityserver.sdk.service.ExceptionFactory
import se.curity.identityserver.sdk.service.SessionManager
import se.curity.identityserver.sdk.service.credential.UserCredentialManager

interface UsernamePasswordAuthenticatorConfig : Configuration {
    @Description("Factory for creating SDK exceptions")
    fun getExceptionFactory(): ExceptionFactory
    
    @Description("Manager for session state")
    fun getSessionManager(): SessionManager
    
    @Description("User credential verification service")
    fun getUserCredentialManager(): UserCredentialManager
}
```

**Request Model** (`UsernamePasswordRequestModel.kt`):
```kotlin
package io.curity.identityserver.plugin.usernamepassword

import jakarta.validation.constraints.NotBlank
import se.curity.identityserver.sdk.web.Request

class UsernamePasswordRequestModel(request: Request) {
    @NotBlank(message = "validation.error.username.required")
    val username: String? = request.getFormParameterValues("username").firstOrNull()
    
    @NotBlank(message = "validation.error.password.required")
    val password: String? = request.getFormParameterValues("password").firstOrNull()
    
    val isPostBack: Boolean = request.isPostRequest
}

class UsernamePasswordPostRequestModel(request: Request) : UsernamePasswordRequestModel(request) {
    val validatedUsername: String = username!!
    val validatedPassword: String = password!!
}
```

**Request Handler** (`UsernamePasswordRequestHandler.kt`):
```kotlin
package io.curity.identityserver.plugin.usernamepassword

import org.slf4j.Logger
import org.slf4j.LoggerFactory
import se.curity.identityserver.sdk.attribute.SubjectAttributes
import se.curity.identityserver.sdk.authentication.AuthenticationResult
import se.curity.identityserver.sdk.authentication.AuthenticatorRequestHandler
import se.curity.identityserver.sdk.service.credential.CredentialVerificationResult
import se.curity.identityserver.sdk.web.Request
import se.curity.identityserver.sdk.web.Response
import se.curity.identityserver.sdk.web.ResponseModel
import java.util.Optional

class UsernamePasswordRequestHandler(
    private val config: UsernamePasswordAuthenticatorConfig
) : AuthenticatorRequestHandler<UsernamePasswordRequestModel> {
    
    private val logger: Logger = LoggerFactory.getLogger(UsernamePasswordRequestHandler::class.java)
    
    override fun preProcess(request: Request, response: Response): UsernamePasswordRequestModel {
        return if (request.isPostRequest) {
            UsernamePasswordPostRequestModel(request)
        } else {
            UsernamePasswordRequestModel(request)
        }
    }
    
    override fun get(requestModel: UsernamePasswordRequestModel, response: Response): Optional<AuthenticationResult> {
        logger.debug("Displaying username/password form")
        
        val viewData = emptyMap<String, Any>()
        response.setResponseModel(
            ResponseModel.templateResponseModel(viewData, "index/get"),
            Response.ResponseModelScope.ANY
        )
        
        return Optional.empty()
    }
    
    override fun post(requestModel: UsernamePasswordRequestModel, response: Response): Optional<AuthenticationResult> {
        if (requestModel !is UsernamePasswordPostRequestModel) {
            logger.warn("Expected validated post model")
            return get(requestModel, response)
        }
        
        val username = requestModel.validatedUsername
        val password = requestModel.validatedPassword
        
        logger.debug("Verifying credentials for user: {}", username)
        
        val subjectAttributes = SubjectAttributes.of(username)
        val verificationResult = config.getUserCredentialManager().verify(subjectAttributes, password)
        
        return when (verificationResult) {
            is CredentialVerificationResult.Accepted -> {
                logger.info("Authentication successful for user: {}", username)
                Optional.of(AuthenticationResult(username))
            }
            is CredentialVerificationResult.Rejected -> {
                logger.warn("Authentication failed for user: {}", username)
                val viewData = mapOf("_error" to "Invalid username or password")
                response.setResponseModel(
                    ResponseModel.templateResponseModel(viewData, "index/get"),
                    Response.ResponseModelScope.ANY
                )
                Optional.empty()
            }
        }
    }
}
```

**Template** (`templates/authenticator/username-password/index/get.vm`):
```velocity
#define($_body)
    #if($_error)
        <div class="error">$_error</div>
    #end
    
    <form method="post">
        <div class="form-field">
            <label for="username">Username</label>
            <input type="text" id="username" name="username" class="block full-width mb1 field-light" 
                   autocomplete="username" autofocus />
            <i class="form-field-icon icon ion-ios-person"></i>
        </div>
        
        <div class="form-field">
            <label for="password">Password</label>
            <input type="password" id="password" name="password" class="block full-width mb1 field-light" 
                   autocomplete="current-password" />
            <i class="form-field-icon icon ion-ios-locked"></i>
        </div>
        
        <button type="submit" class="button button-fullwidth mt2">Sign In</button>
    </form>
#end
#parse('layouts/default')
```

---

## Multi-Screen Authenticators

### Recipe 2: SMS OTP Flow (Password → OTP)

Two-screen authenticator: first verifies password, then sends SMS OTP and verifies it.

**Descriptor** (`SmsOtpAuthenticatorDescriptor.kt`):
```kotlin
package io.curity.identityserver.plugin.smsotp.descriptor

import io.curity.identityserver.plugin.smsotp.OtpRequestHandler
import io.curity.identityserver.plugin.smsotp.PasswordRequestHandler
import io.curity.identityserver.plugin.smsotp.SmsOtpAuthenticatorConfig
import se.curity.identityserver.sdk.authentication.AuthenticatorRequestHandler
import se.curity.identityserver.sdk.plugin.descriptor.AuthenticatorPluginDescriptor

class SmsOtpAuthenticatorDescriptor : AuthenticatorPluginDescriptor<SmsOtpAuthenticatorConfig> {
    override fun getPluginImplementationType(): String = "sms-otp"
    
    override fun getConfigurationType(): Class<out SmsOtpAuthenticatorConfig> = 
        SmsOtpAuthenticatorConfig::class.java
    
    override fun getAuthenticationRequestHandlerTypes(): Map<String, Class<out AuthenticatorRequestHandler<*>>> {
        return mapOf(
            "index" to PasswordRequestHandler::class.java,
            "otp" to OtpRequestHandler::class.java
        )
    }
}
```

**Configuration** (`SmsOtpAuthenticatorConfig.kt`):
```kotlin
package io.curity.identityserver.plugin.smsotp

import se.curity.identityserver.sdk.config.Configuration
import se.curity.identityserver.sdk.config.annotation.Description
import se.curity.identityserver.sdk.service.ExceptionFactory
import se.curity.identityserver.sdk.service.SessionManager
import se.curity.identityserver.sdk.service.credential.UserCredentialManager
import se.curity.identityserver.sdk.service.AccountManager
import se.curity.identityserver.sdk.service.SmsSender
import se.curity.identityserver.sdk.service.authentication.AuthenticatorInformationProvider

interface SmsOtpAuthenticatorConfig : Configuration {
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

**Password Handler** (`PasswordRequestHandler.kt`):
```kotlin
package io.curity.identityserver.plugin.smsotp

import org.slf4j.LoggerFactory
import se.curity.identityserver.sdk.attribute.Attribute
import se.curity.identityserver.sdk.attribute.AttributeValue
import se.curity.identityserver.sdk.attribute.MapAttributeValue
import se.curity.identityserver.sdk.attribute.SubjectAttributes
import se.curity.identityserver.sdk.authentication.AuthenticationResult
import se.curity.identityserver.sdk.authentication.AuthenticatorRequestHandler
import se.curity.identityserver.sdk.errors.ErrorCode
import se.curity.identityserver.sdk.service.credential.CredentialVerificationResult
import se.curity.identityserver.sdk.web.Request
import se.curity.identityserver.sdk.web.Response
import se.curity.identityserver.sdk.web.ResponseModel
import java.util.Optional
import kotlin.random.Random

class PasswordRequestHandler(private val config: SmsOtpAuthenticatorConfig) : 
    AuthenticatorRequestHandler<PasswordRequestModel> {
    
    private val logger = LoggerFactory.getLogger(PasswordRequestHandler::class.java)
    
    override fun preProcess(request: Request, response: Response): PasswordRequestModel {
        return if (request.isPostRequest) {
            PasswordPostRequestModel(request)
        } else {
            PasswordRequestModel(request)
        }
    }
    
    override fun get(requestModel: PasswordRequestModel, response: Response): Optional<AuthenticationResult> {
        val viewData = emptyMap<String, Any>()
        response.setResponseModel(
            ResponseModel.templateResponseModel(viewData, "index/get"),
            Response.ResponseModelScope.ANY
        )
        return Optional.empty()
    }
    
    override fun post(requestModel: PasswordRequestModel, response: Response): Optional<AuthenticationResult> {
        if (requestModel !is PasswordPostRequestModel) {
            return get(requestModel, response)
        }
        
        val username = requestModel.validatedUsername
        val password = requestModel.validatedPassword
        
        // Verify password
        val subjectAttributes = SubjectAttributes.of(username)
        val verificationResult = config.getUserCredentialManager().verify(subjectAttributes, password)
        
        if (verificationResult is CredentialVerificationResult.Rejected) {
            logger.warn("Password verification failed for user: {}", username)
            val viewData = mapOf("_error" to "Invalid username or password")
            response.setResponseModel(
                ResponseModel.templateResponseModel(viewData, "index/get"),
                Response.ResponseModelScope.ANY
            )
            return Optional.empty()
        }
        
        // Get account and phone number
        val account = config.getAccountManager().getByUserName(username)
            ?: throw config.getExceptionFactory().internalServerException(
                ErrorCode.EXTERNAL_SERVICE_ERROR,
                "Account not found for user: $username"
            )
        
        val phoneNumber = account.phoneNumbers?.primaryOrFirst?.significantValue
            ?: throw config.getExceptionFactory().internalServerException(
                ErrorCode.CONFIGURATION_ERROR,
                "No phone number configured for user: $username"
            )
        
        // Generate and send OTP
        val otp = String.format("%06d", Random.nextInt(0, 1000000))
        logger.debug("Generated OTP for user: {}", username)
        
        config.getSmsSender().sendSms(phoneNumber, "Your verification code is: $otp")
        logger.info("Sent SMS OTP to user: {}", username)
        
        // Store OTP and account in session
        val sessionManager = config.getSessionManager()
        sessionManager.put(Attribute.of("otp", otp))
        sessionManager.put(Attribute.of("username", username))
        sessionManager.put(Attribute.of("account", MapAttributeValue.of(account.toMap())))
        
        // Redirect to OTP screen
        val otpUrl = config.getAuthenticatorInformationProvider().getFullyQualifiedAuthenticationUri() + "/otp"
        throw config.getExceptionFactory().redirectException(otpUrl)
    }
}
```

**OTP Handler** (`OtpRequestHandler.kt`):
```kotlin
package io.curity.identityserver.plugin.smsotp

import org.slf4j.LoggerFactory
import se.curity.identityserver.sdk.attribute.Attribute
import se.curity.identityserver.sdk.attribute.AuthenticationAttributes
import se.curity.identityserver.sdk.attribute.SubjectAttributes
import se.curity.identityserver.sdk.authentication.AuthenticationResult
import se.curity.identityserver.sdk.authentication.AuthenticatorRequestHandler
import se.curity.identityserver.sdk.errors.ErrorCode
import se.curity.identityserver.sdk.web.Request
import se.curity.identityserver.sdk.web.Response
import se.curity.identityserver.sdk.web.ResponseModel
import java.util.Optional

class OtpRequestHandler(private val config: SmsOtpAuthenticatorConfig) : 
    AuthenticatorRequestHandler<OtpRequestModel> {
    
    private val logger = LoggerFactory.getLogger(OtpRequestHandler::class.java)
    
    override fun preProcess(request: Request, response: Response): OtpRequestModel {
        return if (request.isPostRequest) {
            OtpPostRequestModel(request)
        } else {
            OtpRequestModel(request)
        }
    }
    
    override fun get(requestModel: OtpRequestModel, response: Response): Optional<AuthenticationResult> {
        val viewData = emptyMap<String, Any>()
        response.setResponseModel(
            ResponseModel.templateResponseModel(viewData, "otp/get"),
            Response.ResponseModelScope.ANY
        )
        return Optional.empty()
    }
    
    override fun post(requestModel: OtpRequestModel, response: Response): Optional<AuthenticationResult> {
        if (requestModel !is OtpPostRequestModel) {
            return get(requestModel, response)
        }
        
        val sessionManager = config.getSessionManager()
        
        // Retrieve session data
        val storedOtp = sessionManager.get("otp")?.attributeValue?.value as String?
            ?: throw config.getExceptionFactory().internalServerException(
                ErrorCode.GENERIC_ERROR,
                "Session expired - OTP not found"
            )
        
        val username = sessionManager.get("username")?.attributeValue?.value as String?
            ?: throw config.getExceptionFactory().internalServerException(
                ErrorCode.GENERIC_ERROR,
                "Session expired - username not found"
            )
        
        val accountAttribute = sessionManager.get("account")
        
        // Verify OTP
        if (requestModel.validatedOtp != storedOtp) {
            logger.warn("OTP verification failed for user: {}", username)
            val viewData = mapOf("_error" to "Invalid verification code")
            response.setResponseModel(
                ResponseModel.templateResponseModel(viewData, "otp/get"),
                Response.ResponseModelScope.ANY
            )
            return Optional.empty()
        }
        
        logger.info("OTP verification successful for user: {}", username)
        
        // Clear session
        sessionManager.remove("otp")
        sessionManager.remove("username")
        sessionManager.remove("account")
        
        // Build authentication result with account
        val subjectAttributes = if (accountAttribute != null) {
            SubjectAttributes.of(
                listOf(
                    Attribute.of("subject", username),
                    accountAttribute
                )
            )
        } else {
            SubjectAttributes.of(username)
        }
        
        val authAttributes = AuthenticationAttributes.of(subjectAttributes)
        return Optional.of(AuthenticationResult(authAttributes as se.curity.identityserver.sdk.attribute.AuthenticationAttributes))
    }
}
```

**Key Pattern: Session State Management**
```kotlin
// Store data for next screen
sessionManager.put(Attribute.of("otp", otp))
sessionManager.put(Attribute.of("account", MapAttributeValue.of(account.toMap())))

// Retrieve in next handler
val storedOtp = sessionManager.get("otp")?.attributeValue?.value as String?
val accountAttribute = sessionManager.get("account")

// Reuse attribute directly in SubjectAttributes
SubjectAttributes.of(
    listOf(
        Attribute.of("subject", username),
        accountAttribute  // No unwrapping needed
    )
)

// Clean up
sessionManager.remove("otp")
sessionManager.remove("account")
```

---

## Advanced Patterns

### Recipe 3: Attribute Enrichment

Add custom attributes to the authentication result using AccountManager.

```kotlin
// Get account with all attributes
val account = config.getAccountManager().getByUserName(username)

// Build subject with account data
val subjectAttributes = SubjectAttributes.of(
    listOf(
        Attribute.of("subject", username),
        Attribute.of("email", account.emails?.primaryOrFirst?.value ?: ""),
        Attribute.of("account", account as AttributeValue)
    )
)

val authAttributes = AuthenticationAttributes.of(subjectAttributes)
return Optional.of(AuthenticationResult(authAttributes as se.curity.identityserver.sdk.attribute.AuthenticationAttributes))
```

### Recipe 4: External Service Integration

Pattern for calling external APIs with proper error handling.

```kotlin
import org.slf4j.LoggerFactory
import se.curity.identityserver.sdk.errors.ErrorCode

private val logger = LoggerFactory.getLogger(MyHandler::class.java)

fun callExternalService(username: String): UserData {
    return try {
        logger.debug("Calling external service for user: {}", username)
        
        val response = httpClient.get("https://api.example.com/users/$username")
        
        if (!response.isSuccessful) {
            logger.error("External service returned error: {}", response.statusCode)
            throw config.getExceptionFactory().externalServiceException(
                "External service error: ${response.statusCode}"
            )
        }
        
        response.body
    } catch (e: IOException) {
        logger.error("Failed to connect to external service", e)
        throw config.getExceptionFactory().externalServiceException(
            "Failed to connect to external service: ${e.message}"
        )
    }
}
```

---

## Authentication Actions

### Recipe 6: Conditional Denial Action

An authentication action that denies authentication based on an attribute value.

**Descriptor** (`DenyAuthenticationActionDescriptor.kt`):
```kotlin
package io.curity.identityserver.plugin.deny.descriptor

import io.curity.identityserver.plugin.deny.DenyAuthenticationAction
import io.curity.identityserver.plugin.deny.DenyAuthenticationActionConfiguration
import se.curity.identityserver.sdk.authenticationaction.AuthenticationAction
import se.curity.identityserver.sdk.plugin.descriptor.AuthenticationActionPluginDescriptor

class DenyAuthenticationActionDescriptor : 
    AuthenticationActionPluginDescriptor<DenyAuthenticationActionConfiguration> {
    
    override fun getPluginImplementationType(): Class<out AuthenticationAction> =
        DenyAuthenticationAction::class.java
    
    override fun getConfigurationType(): Class<out DenyAuthenticationActionConfiguration> =
        DenyAuthenticationActionConfiguration::class.java
}
```

**Configuration** (`DenyAuthenticationActionConfiguration.kt`):
```kotlin
package io.curity.identityserver.plugin.deny

import se.curity.identityserver.sdk.authenticationaction.AttributeSource
import se.curity.identityserver.sdk.config.Configuration
import se.curity.identityserver.sdk.config.annotation.DefaultEnum
import se.curity.identityserver.sdk.config.annotation.Description
import se.curity.identityserver.sdk.service.ExceptionFactory
import java.util.Optional

interface DenyAuthenticationActionConfiguration : Configuration {
    
    fun getExceptionFactory(): ExceptionFactory
    
    @Description("Error message to display when authentication is denied")
    fun getError(): Optional<String>
    
    @Description("Mode for determining when to deny authentication")
    fun getMode(): Mode
    
    interface Mode {
        fun getAlways(): Optional<Always>
        fun getAttributeCondition(): Optional<AttributeCondition>
        
        interface Always
        
        interface AttributeCondition {
            @Description("Name of the attribute to check")
            fun getName(): String
            
            @Description("Where to look for the attribute")
            @DefaultEnum("SUBJECT_ATTRIBUTES")
            fun getSource(): AttributeSource
            
            @Description("Expected boolean value to trigger denial")
            fun getExpectedValue(): Boolean
        }
    }
}
```

**Action** (`DenyAuthenticationAction.kt`):
```kotlin
package io.curity.identityserver.plugin.deny

import se.curity.identityserver.sdk.authenticationaction.AuthenticationAction
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionContext
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult.failedResult
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult.successfulResult
import se.curity.identityserver.sdk.errors.ErrorCode

class DenyAuthenticationAction(
    private val config: DenyAuthenticationActionConfiguration
) : AuthenticationAction {
    
    override fun apply(context: AuthenticationActionContext): AuthenticationActionResult {
        val error = config.error.orElse("Access denied")
        
        return when {
            // Always deny mode
            config.mode.always.isPresent -> failedResult(error)
            
            // Conditional denial based on attribute
            config.mode.attributeCondition.isPresent -> {
                val condition = config.mode.attributeCondition.get()
                val attributes = condition.source.getFrom(context)
                val attribute = attributes.get(condition.name)
                val attributeValue = attribute?.value?.toString()?.toBoolean() ?: false
                
                if (attributeValue == condition.expectedValue) {
                    failedResult(error)
                } else {
                    successfulResult(context.authenticationAttributes, context.actionAttributes)
                }
            }
            
            else -> throw config.exceptionFactory.internalServerException(
                ErrorCode.GENERIC_ERROR,
                "Unknown mode: ${config.mode}"
            )
        }
    }
}
```

**Service Descriptor** (`META-INF/services/se.curity.identityserver.sdk.plugin.descriptor.AuthenticationActionPluginDescriptor`):
```
io.curity.identityserver.plugin.deny.descriptor.DenyAuthenticationActionDescriptor
```

**Test** (`DenyAuthenticationActionSpec.groovy`):
```groovy
package io.curity.identityserver.plugin.deny

import se.curity.identityserver.sdk.attribute.Attribute
import se.curity.identityserver.sdk.attribute.AuthenticationAttributes
import se.curity.identityserver.sdk.attribute.SubjectAttributes
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionContext
import se.curity.identityserver.sdk.authenticationaction.AttributeSource
import spock.lang.Specification

class DenyAuthenticationActionSpec extends Specification {
    
    def config
    def action
    
    def setup() {
        config = Mock(DenyAuthenticationActionConfiguration)
        action = new DenyAuthenticationAction(config)
    }
    
    def "should deny when attribute condition is met"() {
        given: "a context with blocked attribute set to true"
        def subjectAttributes = SubjectAttributes.of("testuser")
            .with(Attribute.of("blocked", true))
        def authAttributes = AuthenticationAttributes.of(subjectAttributes)
        def context = Mock(AuthenticationActionContext)
        context.authenticationAttributes >> authAttributes
        
        and: "config checks for blocked attribute"
        def mode = Mock(DenyAuthenticationActionConfiguration.Mode)
        def condition = Mock(DenyAuthenticationActionConfiguration.Mode.AttributeCondition)
        condition.name >> "blocked"
        condition.source >> AttributeSource.SUBJECT_ATTRIBUTES
        condition.expectedValue >> true
        mode.attributeCondition >> Optional.of(condition)
        mode.always >> Optional.empty()
        config.mode >> mode
        config.error >> Optional.of("User is blocked")
        
        when: "applying the action"
        def result = action.apply(context)
        
        then: "authentication is denied"
        !result.isSuccessful()
        result.errorMessage == "User is blocked"
    }
    
    def "should succeed when attribute condition is not met"() {
        given: "a context without blocked attribute"
        def subjectAttributes = SubjectAttributes.of("testuser")
        def authAttributes = AuthenticationAttributes.of(subjectAttributes)
        def context = Mock(AuthenticationActionContext)
        context.authenticationAttributes >> authAttributes
        
        and: "config checks for blocked attribute"
        def mode = Mock(DenyAuthenticationActionConfiguration.Mode)
        def condition = Mock(DenyAuthenticationActionConfiguration.Mode.AttributeCondition)
        condition.name >> "blocked"
        condition.source >> AttributeSource.SUBJECT_ATTRIBUTES
        condition.expectedValue >> true
        mode.attributeCondition >> Optional.of(condition)
        mode.always >> Optional.empty()
        config.mode >> mode
        
        when: "applying the action"
        def result = action.apply(context)
        
        then: "authentication succeeds"
        result.isSuccessful()
    }
}
```

---

## Testing Patterns

### Recipe 5: Basic Handler Test

```groovy
package io.curity.identityserver.plugin.usernamepassword

import se.curity.identityserver.sdk.service.credential.CredentialVerificationResult
import se.curity.identityserver.sdk.web.Response
import spock.lang.Specification

class UsernamePasswordRequestHandlerSpec extends Specification {

    def config
    def userCredentialManager
    def handler
    def response

    def setup() {
        config = Mock(UsernamePasswordAuthenticatorConfig)
        userCredentialManager = Mock()
        response = Mock()
        
        config.getUserCredentialManager() >> userCredentialManager
        
        handler = new UsernamePasswordRequestHandler(config)
    }

    def "POST with valid credentials should authenticate successfully"() {
        given: "a POST request with valid credentials"
        def requestModel = Mock(UsernamePasswordPostRequestModel)
        requestModel.validatedUsername >> "alice"
        requestModel.validatedPassword >> "password123"
        
        and: "credential verification succeeds"
        userCredentialManager.verify(_, "password123") >> CredentialVerificationResult.Accepted.instance

        when: "handling the POST request"
        def result = handler.post(requestModel, response)

        then: "authentication result is returned"
        result.isPresent()
        def authResult = result.get()
        authResult != null
    }
    
    def "POST with invalid credentials should show error"() {
        given: "a POST request with invalid credentials"
        def requestModel = Mock(UsernamePasswordPostRequestModel)
        requestModel.validatedUsername >> "alice"
        requestModel.validatedPassword >> "wrongpassword"
        
        and: "credential verification fails"
        userCredentialManager.verify(_, _) >> CredentialVerificationResult.Rejected.instance

        when: "handling the POST request"
        def result = handler.post(requestModel, response)

        then: "error is displayed"
        1 * response.setResponseModel({ it.data['_error'] != null }, Response.ResponseModelScope.ANY)
        
        and: "result is empty"
        !result.isPresent()
    }
}
```

---

## Event Listeners

### Recipe 7: All-Events Logger

A minimal event listener that logs all events. This is the simplest event listener plugin.

**Descriptor** (`AllEventsEventListenerDescriptor.java`):
```java
package io.curity.identityserver.plugins.eventlistener;

import se.curity.identityserver.sdk.event.EventListener;
import se.curity.identityserver.sdk.event.EventListenerCollection;
import se.curity.identityserver.sdk.plugin.descriptor.EventListenerPluginDescriptor;

import java.util.Collections;
import java.util.Set;

public final class AllEventsEventListenerDescriptor
        implements EventListenerPluginDescriptor<AllEventsEventListenerConfig>
{
    @Override
    public Class<? extends EventListenerCollection> getEventListenerCollection()
    {
        return AllEventsListenerCollection.class;
    }

    @Override
    public String getPluginImplementationType()
    {
        return "all-events";
    }

    @Override
    public Class<? extends AllEventsEventListenerConfig> getConfigurationType()
    {
        return AllEventsEventListenerConfig.class;
    }

    public static final class AllEventsListenerCollection implements EventListenerCollection
    {
        private final Set<EventListener<?>> _listeners;

        public AllEventsListenerCollection(AllEventsEventListenerConfig configuration)
        {
            _listeners = Collections.singleton(new AllEventsEventListener(configuration));
        }

        @Override
        public Set<? extends EventListener<?>> getListeners()
        {
            return Collections.unmodifiableSet(_listeners);
        }
    }
}
```

**Configuration** (`AllEventsEventListenerConfig.java`):
```java
package io.curity.identityserver.plugins.eventlistener;

import se.curity.identityserver.sdk.config.Configuration;

public interface AllEventsEventListenerConfig extends Configuration
{
    // Empty — no additional services or config needed for basic logging
}
```

**Event Listener** (`AllEventsEventListener.java`):
```java
package io.curity.identityserver.plugins.eventlistener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import se.curity.identityserver.sdk.data.events.Event;
import se.curity.identityserver.sdk.event.EventListener;

public final class AllEventsEventListener implements EventListener<Event>
{
    private static final Logger _logger = LoggerFactory.getLogger(AllEventsEventListener.class);
    private final AllEventsEventListenerConfig _configuration;

    public AllEventsEventListener(AllEventsEventListenerConfig configuration)
    {
        _configuration = configuration;
    }

    @Override
    public Class<Event> getEventType()
    {
        return Event.class;  // Listen to ALL event types
    }

    @Override
    public void handle(Event event)
    {
        _logger.info("Event received: {}", event.asMap());
    }
}
```

**Service Descriptor** (`src/main/resources/META-INF/services/se.curity.identityserver.sdk.plugin.descriptor.EventListenerPluginDescriptor`):
```
io.curity.identityserver.plugins.eventlistener.AllEventsEventListenerDescriptor
```

**Key Points**:
- `EventListener<Event>` with `getEventType()` returning `Event.class` receives all events
- Use a specific event subtype (e.g., `IssuedAccessTokenOAuthEvent`) to filter
- Event listeners must be thread-safe
- The `EventListenerCollection` inner class instantiates all listeners from config

---

## Token Procedures

### Recipe 8: External IdP Token Exchange

A token exchange procedure that validates an external JWT, then issues an internal access token. This demonstrates the complete token exchange pattern per RFC 8693.

**Descriptor** (`ExternalIdpTokenProcedureDescriptor.java`):
```java
package io.curity.identityserver.plugins.tokenexchange.descriptor;

import io.curity.identityserver.plugins.tokenexchange.ExternalIdpTokenExchangeProcedure;
import io.curity.identityserver.plugins.tokenexchange.config.ExternalIdpTokenProcedureConfig;
import se.curity.identityserver.sdk.plugin.descriptor.TokenProcedurePluginDescriptor;
import se.curity.identityserver.sdk.procedure.token.OAuthTokenExchangeTokenProcedure;

public final class ExternalIdpTokenProcedureDescriptor
        implements TokenProcedurePluginDescriptor<ExternalIdpTokenProcedureConfig>
{
    @Override
    public Class<? extends OAuthTokenExchangeTokenProcedure> getOAuthTokenEndpointOAuthTokenExchangeTokenProcedure()
    {
        return ExternalIdpTokenExchangeProcedure.class;
    }

    @Override
    public String getPluginImplementationType()
    {
        return "external-idp-token-exchange";
    }

    @Override
    public Class<? extends ExternalIdpTokenProcedureConfig> getConfigurationType()
    {
        return ExternalIdpTokenProcedureConfig.class;
    }
}
```

**Configuration** (`ExternalIdpTokenProcedureConfig.java`):
```java
package io.curity.identityserver.plugins.tokenexchange.config;

import se.curity.identityserver.sdk.config.Configuration;
import se.curity.identityserver.sdk.config.annotation.DefaultLong;
import se.curity.identityserver.sdk.config.annotation.DefaultService;
import se.curity.identityserver.sdk.config.annotation.DefaultString;
import se.curity.identityserver.sdk.config.annotation.Description;
import se.curity.identityserver.sdk.service.ExceptionFactory;
import se.curity.identityserver.sdk.service.HttpClient;
import se.curity.identityserver.sdk.service.issuer.DefaultJwtAccessTokenIssuerProvider;

public interface ExternalIdpTokenProcedureConfig extends Configuration
{
    ExceptionFactory getExceptionFactory();

    @DefaultService
    DefaultJwtAccessTokenIssuerProvider getJwtAccessTokenIssuerProvider();

    @DefaultString("https://example.idp.com/.well-known/openid-configuration")
    @Description("External IdP OIDC discovery URL for JWKS resolution")
    String getMetadataURL();

    @Description("Clock skew tolerance in seconds")
    @DefaultLong(2)
    Long getClockSkew();

    HttpClient getHttpClient();
}
```

**Token Exchange Procedure** (`ExternalIdpTokenExchangeProcedure.java`):
```java
package io.curity.identityserver.plugins.tokenexchange;

import io.curity.identityserver.plugins.tokenexchange.config.ExternalIdpTokenProcedureConfig;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import se.curity.identityserver.sdk.Nullable;
import se.curity.identityserver.sdk.attribute.Attribute;
import se.curity.identityserver.sdk.attribute.ContextAttributes;
import se.curity.identityserver.sdk.attribute.SubjectAttributes;
import se.curity.identityserver.sdk.attribute.token.AccessTokenAttributes;
import se.curity.identityserver.sdk.data.tokens.TokenIssuerException;
import se.curity.identityserver.sdk.errors.ErrorCode;
import se.curity.identityserver.sdk.procedure.token.OAuthTokenExchangeTokenProcedure;
import se.curity.identityserver.sdk.procedure.token.context.OAuthTokenExchangeTokenProcedurePluginContext;
import se.curity.identityserver.sdk.procedure.token.context.OAuthTokenExchangeUnInitializedTokenProcedurePluginContext;
import se.curity.identityserver.sdk.service.issuer.AccessTokenIssuer;
import se.curity.identityserver.sdk.web.ResponseModel;

import java.time.Instant;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

public final class ExternalIdpTokenExchangeProcedure implements OAuthTokenExchangeTokenProcedure
{
    private static final Logger _logger = LoggerFactory.getLogger(ExternalIdpTokenExchangeProcedure.class);
    private final ExternalIdpTokenProcedureConfig _configuration;

    public ExternalIdpTokenExchangeProcedure(ExternalIdpTokenProcedureConfig configuration)
    {
        _configuration = configuration;
    }

    @Override
    public ResponseModel run(OAuthTokenExchangeUnInitializedTokenProcedurePluginContext context)
    {
        // 1. Extract the subject token from the request
        String presentedToken = context.getSubjectTokenValue();
        if (presentedToken == null)
        {
            _logger.debug("No subject token in request");
            throw _configuration.getExceptionFactory()
                    .badRequestException(ErrorCode.TOKEN_ISSUANCE_ERROR, "Invalid subject token");
        }

        // 2. Validate the external token (implement JWT validation with JWKS)
        String subject = validateAndExtractSubject(presentedToken);

        // 3. Determine scopes and audiences from the client configuration
        Set<String> audiences = context.getClient().getAudiences();
        Set<String> scopes = context.getClient().getScopeNames();

        // 4. Initialize the token context
        OAuthTokenExchangeTokenProcedurePluginContext fullContext = context.getInitializedContext(
                SubjectAttributes.of(subject),
                ContextAttributes.empty(),
                audiences,
                scopes
        );

        // 5. Build access token data with optional custom claims
        var tokenData = fullContext.getDefaultAccessTokenData()
                .with(Attribute.of("user_id", subject));

        var delegation = fullContext.getDefaultDelegationData();

        // 6. Issue the token
        try
        {
            @Nullable AccessTokenIssuer issuer = _configuration.getJwtAccessTokenIssuerProvider()
                    .getDefaultJwtAccessTokenIssuer();
            if (issuer == null)
            {
                throw _configuration.getExceptionFactory()
                        .badRequestException(ErrorCode.TOKEN_ISSUANCE_ERROR, "JWT not enabled");
            }

            @Nullable String issuedToken = issuer.issue(
                    AccessTokenAttributes.of(tokenData),
                    fullContext.issueDelegation(delegation)
            );

            if (issuedToken == null)
            {
                throw _configuration.getExceptionFactory()
                        .badRequestException(ErrorCode.TOKEN_ISSUANCE_ERROR, "Token issuance failed");
            }

            // 7. Build the OAuth token response
            var responseData = new HashMap<String, Object>();
            responseData.put("access_token", issuedToken);
            responseData.put("token_type", "bearer");
            responseData.put("scope", tokenData.get("scope").getValue());
            responseData.put("expires_in",
                    Long.parseLong(tokenData.get("exp").getValue().toString()) - Instant.now().getEpochSecond());
            responseData.put("issued_token_type", "urn:ietf:params:oauth:token-type:access_token");

            return ResponseModel.mapResponseModel(responseData);
        }
        catch (TokenIssuerException e)
        {
            return ResponseModel.problemResponseModel("token_issuer_exception", "Could not issue tokens");
        }
    }

    private String validateAndExtractSubject(String token)
    {
        // TODO: Implement JWT validation
        // 1. Fetch OIDC metadata from config.getMetadataURL()
        // 2. Resolve JWKS endpoint
        // 3. Verify JWT signature, expiration, audience, issuer
        // 4. Extract and return the "sub" claim
        throw new UnsupportedOperationException("Implement external token validation");
    }
}
```

**Key Points**:
- Descriptor overrides only `getOAuthTokenEndpointOAuthTokenExchangeTokenProcedure()` — other flows use defaults
- Uses `@DefaultService` on `DefaultJwtAccessTokenIssuerProvider` to use the server's default JWT issuer
- The `UnInitializedContext` → `getInitializedContext()` → `getDefaultAccessTokenData()` flow is the standard pattern
- Custom claims added via `.with(Attribute.of("name", value))`
- JWT validation libraries (jose4j, nimbus) must be bundled as runtime dependencies

**Service Descriptor** (`src/main/resources/META-INF/services/se.curity.identityserver.sdk.plugin.descriptor.TokenProcedurePluginDescriptor`):
```
io.curity.identityserver.plugins.tokenexchange.descriptor.ExternalIdpTokenProcedureDescriptor
```

---

## Quick Reference

**Need to:**
- Verify password? → Recipe 1 (Username/Password)
- Multi-screen flow? → Recipe 2 (SMS OTP)
- Add account attributes? → Recipe 3 (Attribute Enrichment)
- Call external API? → Recipe 4 (External Service Integration)
- Deny authentication conditionally? → Recipe 6 (Conditional Denial Action)
- Write tests? → Recipe 5 (Basic Handler Test)
- Listen to server events? → Recipe 7 (All-Events Logger)
- Token exchange with external IdP? → Recipe 8 (External IdP Token Exchange)

**See also:**
- API reference → `sdk-services.md`
- Request handler lifecycle → `request-handlers.md`
- Template syntax → `templating.md`
- Test framework → `testing.md`
- Event listener guide → `plugin-type-event-listener.md`
- Token procedure guide → `plugin-type-token-procedure.md`
- Configuration system → `configuration.md`
