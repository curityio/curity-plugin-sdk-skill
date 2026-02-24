# Authentication Action Plugin Type

**Reference implementation**: [curityio/time-authentication-action](https://github.com/curityio/time-authentication-action) — denies authentication based on time-of-day or date rules.

Authentication Actions run **after authentication** completes but **before** the flow proceeds. They are used to:

- **Enrich** the authentication with additional attributes
- **Deny** authentication based on policies or conditions  
- **Prompt users** for additional input (terms acceptance, attribute collection, etc.)
- **Call external services** to gather context about the authenticated user

> **See also:**
> - `request-handlers.md` - Request handler implementation and validation patterns
> - `templating.md` - Template structure and localization
> - `testing.md` - Unit testing with Spock Framework
> - `sdk-services.md` - Available SDK services
> - `recipes.md` - Complete working examples

## Core Concepts

### 1. AuthenticationAction Interface

The main interface all actions must implement:

```kotlin
interface AuthenticationAction {
    fun apply(context: AuthenticationActionContext): AuthenticationActionResult
}
```

**Parameters:**
- `context: AuthenticationActionContext` - Contains the authenticated user's attributes and session data

**Returns:**
- `AuthenticationActionResult` - One of: `successfulResult`, `failedResult`, or `pendingResult`

### 2. AuthenticationActionContext

The context provided to the action contains:

```kotlin
interface AuthenticationActionContext {
    val authenticationAttributes: AuthenticationAttributes  // Result from authenticator
    val actionAttributes: Attributes                         // Attributes from previous actions
}
```

**Accessing attributes:**
```kotlin
// Get subject attributes (username, email, etc.)
val subjectAttributes = context.authenticationAttributes.subjectAttributes
val username = subjectAttributes.get("subject").get().value

// Get context attributes (IP address, user agent, etc.)
val contextAttributes = context.authenticationAttributes.contextAttributes

// Get attributes from previous actions
val previousActionAttribute = context.actionAttributes.get("some-attribute")
```

### 3. AuthenticationActionResult Types

Three types of results:

#### Success - Continue authentication
```kotlin
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult.successfulResult

// Continue without modification
successfulResult(context.authenticationAttributes, context.actionAttributes)

// Continue with enriched attributes
val enrichedActionAttributes = context.actionAttributes.with(Attribute.of("new-attribute", "value"))
successfulResult(context.authenticationAttributes, enrichedActionAttributes)
```

#### Failed - Deny authentication
```kotlin
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult.failedResult

// Deny with error message
failedResult("Access denied: User does not meet requirements")
```

#### Pending - Prompt user for input
```kotlin
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult.pendingResult
import se.curity.identityserver.sdk.authenticationaction.completions.RequiredActionCompletion.PromptUser.prompt

// Prompt user via registered request handler
pendingResult(prompt())
```

**Example using all three result types:**

```kotlin
class RiskBasedAction(private val config: RiskActionConfiguration) : AuthenticationAction {
    
    override fun apply(context: AuthenticationActionContext): AuthenticationActionResult {
        val ipAddress = context.authenticationAttributes.contextAttributes.get("ip").get().value
        val riskScore = calculateRiskScore(ipAddress)
        
        return when {
            // High risk - deny authentication
            riskScore > config.maxAllowedRisk -> 
                failedResult("Authentication denied due to risk")
            
            // Medium risk - require additional verification
            riskScore > config.mediumRiskThreshold -> 
                pendingResult(prompt())
            
            // Low risk - allow authentication
            else -> 
                successfulResult(context.authenticationAttributes, context.actionAttributes)
        }
    }
}
```

## Plugin Structure

### 1. Descriptor

```kotlin
package io.curity.identityserver.plugin.myaction.descriptor

import se.curity.identityserver.sdk.plugin.descriptor.AuthenticationActionPluginDescriptor

class MyActionDescriptor : AuthenticationActionPluginDescriptor<MyActionConfiguration> {
    override fun getPluginImplementationType(): String = "my-action"
    
    override fun getAuthenticationAction(): Class<out AuthenticationAction> =
        MyActionAuthenticationAction::class.java
    
    override fun getConfigurationType(): Class<out MyActionConfiguration> =
        MyActionConfiguration::class.java
    
    // Optional: Register request handlers for prompting users
    override fun getAuthenticationActionRequestHandlerTypes(): Map<String, Class<out ActionCompletionRequestHandler<*>>> =
        mapOf(
            "index" to MyActionRequestHandler::class.java
        )
}
```

### 2. Configuration Interface

Define configuration properties for your action:

```kotlin
package io.curity.identityserver.plugin.myaction.config

import se.curity.identityserver.sdk.config.Configuration
import se.curity.identityserver.sdk.service.ExceptionFactory
import se.curity.identityserver.sdk.service.SessionManager

interface MyActionConfiguration : Configuration {
    
    // SDK services - see sdk-services.md for available services
    fun getExceptionFactory(): ExceptionFactory
    fun getSessionManager(): SessionManager
    
    // Custom configuration properties
    fun getSomeConfigValue(): String
}
```

> **Note:** For SDK services and their usage, see `sdk-services.md`.

### 3. Service Descriptor File (MANDATORY)

Create `src/main/resources/META-INF/services/se.curity.identityserver.sdk.plugin.descriptor.AuthenticationActionPluginDescriptor`:

```
io.curity.identityserver.plugin.myaction.descriptor.MyActionDescriptor
```

## Implementation Patterns

### Pattern 1: Simple Denial Action

Use case: Deny authentication based on configuration or conditions.

```kotlin
package io.curity.identityserver.plugin.myaction

import se.curity.identityserver.sdk.authenticationaction.AuthenticationAction
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionContext
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult.failedResult
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult.successfulResult

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
                val attribute = condition.source.getFrom(context).get(condition.name)
                val attributeValue = attribute?.value?.toString()?.toBoolean() ?: false
                
                if (attributeValue == condition.expectedValue) {
                    failedResult(error)
                } else {
                    successfulResult(context.authenticationAttributes, context.actionAttributes)
                }
            }
            
            else -> throw config.getExceptionFactory().internalServerException(
                ErrorCode.GENERIC_ERROR, 
                "Unknown mode"
            )
        }
    }
}
```

### Pattern 2: Attribute Enrichment Action

Use case: Call external service and add attributes to the authentication result.

```kotlin
package io.curity.identityserver.plugin.myaction

import org.slf4j.Logger
import org.slf4j.LoggerFactory
import se.curity.identityserver.sdk.attribute.Attribute
import se.curity.identityserver.sdk.authenticationaction.AuthenticationAction
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionContext
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult.successfulResult

class EnrichmentAuthenticationAction(
    private val config: EnrichmentActionConfiguration
) : AuthenticationAction {
    
    private val logger: Logger = LoggerFactory.getLogger(EnrichmentAuthenticationAction::class.java)
    
    override fun apply(context: AuthenticationActionContext): AuthenticationActionResult {
        val username = context.authenticationAttributes.subjectAttributes.subject
        
        logger.debug("Enriching authentication for user: {}", username)
        
        // Call external service or SDK service to get additional data
        val additionalData = fetchUserDataFromExternalService(username)
        
        // Add to action attributes
        val enrichedAttributes = context.actionAttributes.with(
            Attribute.of("department", additionalData.department),
            Attribute.of("employeeId", additionalData.employeeId),
            Attribute.of("roles", additionalData.roles)
        )
        
        return successfulResult(context.authenticationAttributes, enrichedAttributes)
    }
    
    private fun fetchUserDataFromExternalService(username: String): UserData {
        // Implementation using HttpClient or SDK services
        // ...
    }
}
```

### Pattern 3: User Prompt Action (with Request Handler)

Use case: Prompt user for additional input during authentication flow.

**Action implementation:**

```kotlin
package io.curity.identityserver.plugin.myaction

import org.slf4j.Logger
import org.slf4j.LoggerFactory
import se.curity.identityserver.sdk.authenticationaction.AuthenticationAction
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionContext
import se.curity.identityserver.sdk.authenticationaction.AuthenticationActionResult
import se.curity.identityserver.sdk.authenticationaction.completions.RequiredActionCompletion

class SelectorAuthenticationAction(
    private val config: SelectorAuthenticationActionConfiguration
) : AuthenticationAction {
    
    private val session = config.getSessionManager()
    private val logger: Logger = LoggerFactory.getLogger(SelectorAuthenticationAction::class.java)
    
    override fun apply(context: AuthenticationActionContext): AuthenticationActionResult {
        // Check if user already provided input (stored in session by request handler)
        val selectedAttribute = session.get(config.attributeName)
        
        return if (selectedAttribute != null) {
            logger.debug("Selected option is '{}'", selectedAttribute.value)
            session.remove(config.attributeName)
            
            // Add selected value to action attributes
            val enrichedAttributes = context.actionAttributes.with(selectedAttribute)
            
            AuthenticationActionResult.successfulResult(context.authenticationAttributes, enrichedAttributes)
        } else {
            // No input yet - prompt user via request handler
            logger.trace("No selected option; prompting user")
            AuthenticationActionResult.pendingResult(
                SelectorRequestHandler(config)  // Instantiate request handler
            )
        }
    }
}
```

> **Request Handler Implementation:** See `request-handlers.md` for complete request handler patterns, including:
> - Request model validation with Jakarta annotations
> - GET/POST handling
> - Form processing and session management
> - Template response models

> **Template Structure:** See `templating.md` for:
> - Template file location and naming
> - Velocity syntax and variables
> - Localization with message keys
> - Layout integration

## Accessing Attributes from Different Sources

Authentication actions can access attributes from various sources using `AttributeSource`:

```kotlin
import se.curity.identityserver.sdk.config.annotation.DefaultEnum
import se.curity.identityserver.sdk.authenticationaction.AttributeSource

interface MyActionConfiguration : Configuration {
    
    fun getAttributeCondition(): AttributeCondition
    
    interface AttributeCondition {
        fun getName(): String
        
        @DefaultEnum("SUBJECT_ATTRIBUTES")
        fun getSource(): AttributeSource  // SUBJECT_ATTRIBUTES, CONTEXT_ATTRIBUTES, or ACTION_ATTRIBUTES
        
        fun getExpectedValue(): Boolean
    }
}
```

**Usage in action:**

```kotlin
val condition = config.getAttributeCondition()
val attributes = condition.source.getFrom(context)  // Gets correct attribute collection
val attribute = attributes.get(condition.name)
```

## Common Use Cases

### 1. Terms and Conditions Acceptance

```kotlin
class TermsAcceptanceAction(private val config: TermsActionConfiguration) : AuthenticationAction {
    
    override fun apply(context: AuthenticationActionContext): AuthenticationActionResult {
        val session = config.getSessionManager()
        val accepted = session.get("terms-accepted")?.value?.toString()?.toBoolean() ?: false
        
        return if (accepted) {
            session.remove("terms-accepted")
            successfulResult(context.authenticationAttributes, context.actionAttributes)
        } else {
            pendingResult(prompt())  // Show terms screen
        }
    }
}
```

### 2. Risk-Based Authentication

```kotlin
class RiskBasedAction(private val config: RiskActionConfiguration) : AuthenticationAction {
    
    override fun apply(context: AuthenticationActionContext): AuthenticationActionResult {
        val ipAddress = context.authenticationAttributes.contextAttributes.get("ip").get().value
        val riskScore = calculateRiskScore(ipAddress)
        
        return when {
            riskScore > config.maxAllowedRisk -> failedResult("Authentication denied due to risk")
            riskScore > config.mediumRiskThreshold -> pendingResult(prompt())  // Require additional verification
            else -> successfulResult(context.authenticationAttributes, context.actionAttributes)
        }
    }
}
```

### 3. Attribute Collection from External API

```kotlin
class ExternalAttributeAction(private val config: ExternalApiConfiguration) : AuthenticationAction {
    
    override fun apply(context: AuthenticationActionContext): AuthenticationActionResult {
        val username = context.authenticationAttributes.subjectAttributes.subject
        val httpClient = config.getHttpClient()
        
        val apiResponse = httpClient.request(config.apiEndpoint)
            .queryParam("username", username)
            .get()
            .response()
        
        val userData = parseResponse(apiResponse.body(String::class.java))
        
        val enrichedAttributes = context.actionAttributes.with(
            Attribute.of("external-id", userData.id),
            Attribute.of("department", userData.department)
        )
        
        return successfulResult(context.authenticationAttributes, enrichedAttributes)
    }
}
```

## Best Practices

1. **Use session sparingly** - Session storage should be used only for temporary state between action and request handler
2. **Clean up session data** - Always remove session attributes after using them
3. **Handle null safely** - Attributes may not exist; use Kotlin's null safety features
4. **Log appropriately** - Use DEBUG for normal flow, WARN for unexpected conditions
5. **Return appropriate results** - Use `successfulResult` to continue, `failedResult` to deny, `pendingResult` to prompt
6. **Validate configuration** - Check configuration in action constructor or early in `apply()`
7. **Test thoroughly** - Mock context and configuration to test all code paths

## Testing

> **See `testing.md`** for comprehensive testing guide including:
> - Spock Framework setup and configuration
> - Mocking SDK components and configuration
> - Testing action logic with different contexts
> - Testing request handlers and validation
> - Complete test examples

**Quick example:**

```groovy
class MyAuthenticationActionSpec extends Specification {
    
    MyActionConfiguration config
    MyAuthenticationAction action
    
    def setup() {
        config = Mock(MyActionConfiguration)
        action = new MyAuthenticationAction(config)
    }
    
    def "should deny when condition is met"() {
        given: "a context with blocking attribute"
        def subjectAttributes = SubjectAttributes.of("testuser")
            .with(Attribute.of("blocked", true))
        def authAttributes = AuthenticationAttributes.of(subjectAttributes)
        def context = Mock(AuthenticationActionContext)
        context.authenticationAttributes >> authAttributes
        
        when: "applying the action"
        def result = action.apply(context)
        
        then: "authentication is denied"
        !result.isSuccessful()
    }
}
```
