# Curity Plugin System – Agent Reference

## 1. Conceptual Model

- A **plugin** is a component that implements one or more Curity **extension interfaces**.
- Plugins are discovered and loaded by the Curity Identity Server at startup via Java ServiceLoader.
- Each plugin has:
  - A **type** (authentication, token, attribute, etc.)
  - **Configuration** provided via the Configuration interface or configuration files
  - A defined **lifecycle**: construction, initialization, runtime invocation, and shutdown

## 2. Plugin Discovery

The Curity Identity Server discovers plugins at runtime by scanning JAR files for service descriptor files in `META-INF/services/`. **This service descriptor is MANDATORY** - without it, your plugin will not be loaded.

**Service descriptor configuration:**
- Location: `src/main/resources/META-INF/services/`
- Filename: Fully qualified name of the plugin descriptor interface (e.g., `se.curity.identityserver.sdk.plugin.descriptor.AuthenticatorPluginDescriptor`)
- Content: Fully qualified name(s) of your plugin descriptor implementation class(es), one per line

**Example for an authenticator plugin:**

Filename: `src/main/resources/META-INF/services/se.curity.identityserver.sdk.plugin.descriptor.AuthenticatorPluginDescriptor`

Content:
```
io.curity.identityserver.plugin.usernamepassword.descriptor.UsernamePasswordAuthenticatorDescriptor
```

**Common plugin descriptor interfaces:**
- `se.curity.identityserver.sdk.plugin.descriptor.AuthenticatorPluginDescriptor` - for authenticators
- `se.curity.identityserver.sdk.plugin.descriptor.TokenIssuerPluginDescriptor` - for token issuers

**Critical:** Always create the service descriptor file when implementing a new plugin. The filename must exactly match the interface name, and the content must be the fully qualified class name of your descriptor.

## 3. Lifecycle Rules

When generating code for a plugin:

- Do not perform heavy I/O or expensive operations in constructors.
- Prefer to use dedicated initialization methods/hooks provided by the Curity SDK.
- Use SDK context or service objects to:
  - Access configuration
  - Perform logging
  - Resolve other services (e.g. HTTP, crypto)


### Plugin descriptor

The plugin descriptor implements `se.curity.identityserver.sdk.plugin.PluginDescriptor`

```kotlin
class UsernamePasswordAuthenticatorPluginDescriptor : AuthenticatorPluginDescriptor<UsernamePasswordAuthenticatorConfig>
{
    override fun getPluginImplementationType() = "username-password"

    override fun getConfigurationType() = UsernamePasswordAuthenticatorConfig::class.java

    override fun getAuthenticationRequestHandlerTypes(): Map<String, Class<out AuthenticatorRequestHandler<*>>> =
        mapOf("index" to UsernamePasswordRequestHandler::class.java)
}

```

### RequestHandler

See `request-handlers.md`

## 4. Configuration Integration

- Plugin configuration is declared by creating an interface extending `se.curity.identityserver.sdk.config.Configuration`
- The implementation of the interface will be created at runtime
- The configuration may include primitive values and/or objects
- The SDK provides services that can be injected through the Configuration in the `se.curity.identityserver.sdk.service` package

**Common SDK services to inject**:
- `UserCredentialManager` - For password verification (replaces deprecated `CredentialManager`)
- `ExceptionFactory` - For creating properly typed exceptions
- `AuthenticatorInformationProvider` - Server paths and base URLs

See `sdk-services.md` for complete documentation on available SDK services.

**Example**:
```kotlin
interface UsernamePasswordAuthenticatorConfig : Configuration
{
    @Description("Configure a mandatory string to be used in the plugin")
    fun getMandatoryString(): String

    @Description("Optionally, configure a number to be used in the plugin")
    fun getOptionalNumber(): Optional<Number>

    @Description("The user credential manager to use for verifying passwords.")
    fun getUserCredentialManager(): UserCredentialManager

    @Description("Helper object to provide server paths and base urls")
    fun getAuthenticatorInformationProvider(): AuthenticatorInformationProvider
}

```

## 5. Error Handling & Logging

- Use the SLF4J API (version 2.0.12) for all log output
- Do not rely on `System.out`, `System.err`, or ad-hoc logging
- Exceptions should be thrown using `se.curity.identityserver.sdk.service.ExceptionFactory`
- The `ExceptionFactory` can be obtained by adding it to the Configuration interface
- Always use `ErrorCode` enum constants when creating exceptions (e.g., `ErrorCode.EXTERNAL_SERVICE_ERROR`)
- Never use plain strings as error codes

**Example**:
```kotlin
throw config.getExceptionFactory().internalServerException(
    ErrorCode.EXTERNAL_SERVICE_ERROR,
    "Failed to verify credentials"
)
```

See `sdk-services.md` for complete documentation on `ExceptionFactory`, `ErrorCode`, and logging.

## 5. Deployment Considerations

- Plugins are packaged as a JAR file and deployed to the Curity Identity Server
- Create a subfolder in `$IDSVR_HOME/usr/share/plugins/<plugin-name>/`
- Place the plugin JAR and any `implementation` dependency JARs in this folder
- The plugin JAR contains only the plugin code
- Server-provided dependencies (Curity SDK 10.6.1, SLF4J 2.0.12, Kotlin stdlib 2.2.0) should be marked as `compileOnly` and are not deployed
- Only `implementation` dependencies that are not provided by the server need to be deployed as separate JARs 
