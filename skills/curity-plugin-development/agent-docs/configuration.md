# Configuration – Agent Reference

This document covers the plugin configuration system in depth: type rules, annotations, nested configuration, service injection, and the `@ConfigurationScope` boundary.

## 1. How Configuration Works

Every plugin declares a **Configuration interface** extending `se.curity.identityserver.sdk.config.Configuration`. The server:

1. Parses the interface at startup to discover configurable properties and required services
2. Exposes the properties through admin UI, CLI, and XML configuration
3. Builds an **immutable** implementation of the interface from current configuration state
4. Injects the instance into plugin classes via constructor

If configuration changes at runtime, the server creates a new instance and recreates any plugin classes that depend on it.

### The `id()` Method

All Configuration interfaces inherit:
```java
String id();  // Returns the ID of this plugin instance
```

### Configuration Naming

Method names are automatically converted from camelCase to lisp-case:
- `getServerHost()` → config name `server-host`
- `isValid()` → config name `valid`
- `getOAuthClient()` → config name `o-auth-client`

The `get` and `is` prefixes are stripped. Use `@Name` to override.

## 2. Primitive Configuration Types

| Java Type | Notes |
|-----------|-------|
| `boolean` / `Boolean` | |
| `int` / `Integer` | |
| `long` / `Long` | |
| `double` / `Double` | Precision constrained by `@PrecisionConstraint` (default 10 digits) |
| `String` | Must be non-empty; use `Optional<String>` for nullable |
| `java.net.URI` | |
| Enum types | Any Java enum; use `@DefaultEnum` for defaults |
| `EncryptedString` | Encrypted at rest, decrypted on read (since 9.1) |

### Wrappers

| Wrapper | Notes |
|---------|-------|
| `Optional<T>` | Makes a configuration value optional (nullable). T can be any type except `List` or `Optional` |
| `List<T>` | Ordered list. T can be any type except `List` or `Optional` |

## 3. Default Value Annotations

Apply to configuration methods to provide defaults:

| Annotation | For Type | Example |
|------------|----------|---------|
| `@DefaultString("value")` | `String` | `@DefaultString("https://example.com")` |
| `@DefaultInteger(42)` | `int` | `@DefaultInteger(3600)` |
| `@DefaultLong(100L)` | `long` | `@DefaultLong(86400)` |
| `@DefaultDouble(1.5)` | `double` | `@DefaultDouble(0.95)` |
| `@DefaultBoolean(true)` | `boolean` | `@DefaultBoolean(false)` |
| `@DefaultURI("uri")` | `URI` | `@DefaultURI("https://api.example.com")` |
| `@DefaultEnum("NAME")` | Enum | `@DefaultEnum("SYSTEM_TIME")` |
| `@DefaultService` | Services | Use the server's default service instance |

**`@DefaultService`** is special — it tells the server to use the default instance of a service if one isn't explicitly configured. Supported service types:
- `HttpClient`, `Throttler`
- `AccessTokenIssuer`, `RefreshTokenIssuer`, `IdTokenIssuer`, `NonceIssuer`
- `AccessTokenIntrospecter`, `RefreshTokenIntrospecter`, `IdTokenIntrospecter`, `NonceIntrospecter`

## 4. Constraint Annotations

Apply to configuration methods to validate values:

### `@RangeConstraint(min, max)`
For numeric types (`int`, `long`, `double`):
```java
@RangeConstraint(min = 0, max = 23)
int getHour();

@RangeConstraint(min = 1970, max = 2100)
int getYear();
```

### `@SizeConstraint(min, max)`
For `String` or `List<?>`:
```java
@SizeConstraint(min = 1, max = 255)
String getDisplayName();

@SizeConstraint(min = 1, max = 10)
List<String> getAllowedOrigins();
```

For constraining strings **within** a list:
```java
List<@SizeConstraint(max = 100) String> getNames();
```

### `@PatternConstraint("regex")`
For `String` (uses XML Schema regex syntax, implicitly anchored):
```java
@PatternConstraint("[a-zA-Z0-9_-]+")
String getIdentifier();
```

### `@PrecisionConstraint(digits)`
For `double` (controls decimal precision, default 10):
```java
@PrecisionConstraint(2)
double getRate();
```

## 5. Metadata Annotations

### `@Description("text")`
Provides a description shown in the admin UI/CLI:
```java
@Description("The external IdP's OIDC discovery URL")
String getMetadataURL();
```

Can also annotate the Configuration interface itself:
```java
@Description("A custom username/password authenticator")
public interface MyConfig extends Configuration { ... }
```

### `@Name("config-name")`
Override the auto-generated configuration name:
```java
@Name("oauth-client-id")
String getClientIdentifier();
```

### `@Suggestions({"val1", "val2"})`
Provide suggested values while allowing custom input:
```java
@Suggestions({"openid", "profile", "email"})
String getScope();
```

For optional strings, annotate the type parameter:
```java
Optional<@Suggestions({"red", "blue"}) String> getColor();
```

### `@ListKey`
Mark a field as part of a list item's unique key:
```java
interface ClaimMapping {
    @ListKey
    String getClaimName();

    String getSourceAttribute();
}

List<ClaimMapping> getClaimMappings();
```

## 6. Nested Configuration

Configuration interfaces can contain methods that return other interfaces (not extending `Configuration`). These become nested configuration sections in the admin UI:

```java
public interface MyAuthenticatorConfig extends Configuration
{
    @Description("Time-of-day access restrictions")
    TimeConfiguration getNoAccessBefore();

    @Description("Time-of-day access restrictions")
    TimeConfiguration getNoAccessAfter();
}

// Nested configuration — does NOT extend Configuration
public interface TimeConfiguration
{
    @Description("Hour (0-23)")
    @DefaultInteger(0)
    @RangeConstraint(min = 0, max = 23)
    int getHour();

    @Description("Minute (0-59)")
    @DefaultInteger(0)
    @RangeConstraint(min = 0, max = 59)
    int getMinute();
}
```

### OneOf (Sum Types)

Use `OneOf` when a configuration value can be one of several mutually exclusive types:

```java
import se.curity.identityserver.sdk.config.OneOf;

interface TokenSource extends OneOf {
    Optional<String> staticValue();
    Optional<String> claimName();
}

// In the main config:
TokenSource getTokenSource();

// In the main config, as optional:
Optional<TokenSource> getTokenSource();
```

At runtime, exactly one of the `Optional` methods returns a value.

Use `@DefaultOption` on one method to make it the default choice.

## 7. Service Injection

Any method in a Configuration interface whose return type is a known SDK service will be resolved by the server's dependency injection:

```java
public interface MyPluginConfig extends Configuration
{
    // SDK services — resolved by DI
    ExceptionFactory getExceptionFactory();
    SessionManager getSessionManager();
    AccountManager getAccountManager();
    HttpClient getHttpClient();

    // Optional services — not required
    Optional<EmailSender> getEmailSender();

    // Primitive configuration — resolved from admin config
    String getApiEndpoint();
    @DefaultBoolean(true)
    boolean isEnabled();
}
```

The server distinguishes services from configuration by the return type. SDK service types are injected; primitive/nested types become configurable properties.

## 8. `@ConfigurationScope` Boundary

Services annotated with `@ConfigurationScope` are available to both:
- Regular plugin classes (request handlers, actions, procedures)
- `ManagedObject` instances and `EventListener` instances

Services **without** `@ConfigurationScope` are only available to regular plugin classes.

### Key `@ConfigurationScope` services:
`HttpClient`, `WebServiceClient`, `Json`, `Bucket`, `Throttler`, `SystemInformationProvider`, `EmailSender`

### ManagedObject rules:
- A `ManagedObject` **must NOT cache** service instances in fields
- Always call the Configuration getter each time you use a service
- This is because `ManagedObject` may outlive a service if service configuration changes

```java
// WRONG — do not cache
public class MyManagedObject extends ManagedObject<MyConfig> {
    private final HttpClient httpClient; // BAD

    public MyManagedObject(MyConfig config) {
        super(config);
        this.httpClient = config.getHttpClient(); // BAD — cached reference
    }
}

// CORRECT — always go through config
public class MyManagedObject extends ManagedObject<MyConfig> {
    public MyManagedObject(MyConfig config) {
        super(config);
    }

    public void doSomething() {
        configuration().getHttpClient().request(...); // GOOD — fresh reference
    }
}
```

## 9. EncryptedString

For sensitive configuration values (API keys, secrets) that should be encrypted at rest:

```java
@Description("API secret key")
EncryptedString getApiSecret();
```

Access the decrypted value:
```java
String secret = config.getApiSecret().getValue();
```

## 10. Complete Configuration Example

```java
package com.example.plugin.config;

import se.curity.identityserver.sdk.config.Configuration;
import se.curity.identityserver.sdk.config.EncryptedString;
import se.curity.identityserver.sdk.config.annotation.*;
import se.curity.identityserver.sdk.service.*;
import se.curity.identityserver.sdk.service.authentication.AuthenticatorInformationProvider;
import se.curity.identityserver.sdk.service.credential.UserCredentialManager;

import java.net.URI;
import java.util.List;
import java.util.Optional;

@Description("Example authenticator with comprehensive configuration")
public interface ExampleAuthenticatorConfig extends Configuration
{
    // --- Required services ---
    ExceptionFactory getExceptionFactory();
    SessionManager getSessionManager();
    UserCredentialManager getCredentialManager();
    AuthenticatorInformationProvider getAuthenticatorInformationProvider();

    // --- Optional services ---
    @Description("Email provider for password reset")
    Optional<EmailSender> getEmailSender();

    @Description("HTTP client for external API calls")
    HttpClient getHttpClient();

    // --- Primitive configuration ---
    @Description("External API base URL")
    @DefaultURI("https://api.example.com")
    URI getApiBaseUrl();

    @Description("Maximum login attempts before lockout")
    @DefaultInteger(5)
    @RangeConstraint(min = 1, max = 20)
    int getMaxAttempts();

    @Description("Session timeout in seconds")
    @DefaultLong(3600)
    long getSessionTimeout();

    @Description("Enable debug mode")
    @DefaultBoolean(false)
    boolean isDebugEnabled();

    // --- Enum configuration ---
    @Description("Authentication mode")
    @DefaultEnum("STANDARD")
    AuthMode getAuthMode();

    enum AuthMode { STANDARD, STRICT, LENIENT }

    // --- Encrypted values ---
    @Description("API secret key")
    EncryptedString getApiSecret();

    // --- List configuration ---
    @Description("Allowed redirect domains")
    List<String> getAllowedDomains();

    // --- Nested configuration ---
    @Description("Rate limiting settings")
    RateLimitConfig getRateLimiting();

    interface RateLimitConfig
    {
        @DefaultInteger(10)
        @Description("Max requests per window")
        int getMaxRequests();

        @DefaultInteger(60)
        @Description("Window size in seconds")
        int getWindowSeconds();
    }
}
```
