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

Use `OneOf` when a configuration value can be one of several mutually exclusive types.
`OneOf` is a bare marker interface; each option is an `Optional` getter:

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

At runtime, exactly one of the `Optional` methods returns a value. Read a `OneOf` by
trying each option rather than inferring the choice from which fields are populated:

```java
var source = config.getTokenSource();
var staticValue = source.staticValue().orElse(null);
if (staticValue != null) { /* ... */ }
```

Options may be nested configuration interfaces, not just scalars — that is the usual
way to model "either this set of settings, or that one".

#### A OneOf never produces its own element

A `OneOf` **never** generates an element of its own, at any nesting level. The name of the
`OneOf`-typed property does not appear in the configuration at all; the selected option's
element sits directly where the property would have been. The property name only ever exists
as the accessor in code, so it never has to be a name an administrator would recognise.

For a `getSource()` returning a `OneOf` with a `staticConfiguration` option, nested inside a
`service-providers` list:

```xml
<!-- CORRECT — the option element is a direct child; there is no <source> wrapper -->
<service-providers>
    <realm>urn:test:sp</realm>
    <static-configuration>
        <reply-url>https://sp.example.com/callback</reply-url>
    </static-configuration>
</service-providers>
```

The names that matter in the configuration are therefore the **option** names, not the
property name — choose those carefully.

#### `@DefaultOption` requires a fully defaultable option

`@DefaultOption` marks one option as the default choice, and only one option may carry
it. The annotated option must be creatable **from defaults alone**: every value inside it
needs its own default annotation (`@DefaultString`, `@DefaultInteger`, `@DefaultEnum`, …).

An option containing any value the administrator must supply — a required URL, an
`EncryptedString` secret, a nested list — therefore cannot be the default option, and
there is no default-value annotation that applies to a nested configuration interface.
When no option qualifies, simply omit `@DefaultOption`; the administrator then picks an
option explicitly, which is usually the honest outcome.

Getting this wrong fails at plugin load, not at compile time:

```
se.curity.identityserver.prebooter.PluginBuildException: Failed to update plugin group
'my-plugin' due to Plugin Configuration error: Default value option must be annotated
with both @DefaultOption and the appropriate annotation providing a default value, but
only one annotation was used.
```

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

`ManagedObject` exposes **no `configuration()` accessor** — its public API is the
`ManagedObject(C configuration)` constructor plus `close()` / `close(boolean)`. Keep your
own reference to the configuration and read services off it per call:

```java
// WRONG — do not cache the service
public class MyManagedObject extends ManagedObject<MyConfig> {
    private final HttpClient httpClient; // BAD

    public MyManagedObject(MyConfig config) {
        super(config);
        this.httpClient = config.getHttpClient(); // BAD — cached reference
    }
}

// CORRECT — hold the configuration, resolve the service on each use
public class MyManagedObject extends ManagedObject<MyConfig> {
    private final MyConfig _configuration;

    public MyManagedObject(MyConfig configuration) {
        super(configuration);
        _configuration = configuration;
    }

    public void doSomething() {
        _configuration.getHttpClient().request(...); // GOOD — fresh reference
    }
}
```

Only `@ConfigurationScope` services are reachable this way. Calling a configuration getter
that returns a narrower-scoped service (`ExceptionFactory`, `SessionManager`, …) throws
`UnsupportedOperationException` at runtime.

### ManagedObject lifecycle

- The server creates a new instance **every time the configuration changes**, including at
  startup, and closes the previous instance first. State held in a `ManagedObject` is
  therefore scoped to one configuration generation — which is what makes it the right home
  for a cache of fetched remote documents.
- Return it from `PluginDescriptor.createManagedObject`:

```java
@Override
public Optional<? extends ManagedObject<MyConfig>> createManagedObject(MyConfig configuration) {
    return Optional.of(new MyMetadataCache(configuration));
}
```

- Plugin types receive it by **constructor injection, the same way they receive the
  Configuration** — no `SdkPluginComposer` binding is needed for it.
- The plugin is unavailable until the constructor returns; if it throws, the plugin never
  becomes active. Keep constructors cheap and do not fetch anything remote in them.
- Implement **either** `close()` **or** `close(boolean hasReplacement)`, never both — the
  plugin system calls `close(boolean)`, whose default implementation delegates to `close()`.
- Implementations must be thread-safe (the class implements `ThreadSafe`); `close()` and the
  next instance's constructor may run on different threads.

#### Testing a ManagedObject

In Kotlin, classes and methods are final unless marked `open`, so a `ManagedObject`
subclass cannot be mocked by Spock. Construct the real object with a stubbed configuration
instead — the SDK's HTTP types are all interfaces, so the whole call chain stubs cleanly:

```groovy
HttpClient httpClient = Mock()
MyConfig config = Stub() { getHttpClient() >> httpClient }
def cache = new MyMetadataCache(config)

when:
def result = cache.get("https://sp.example.com/metadata", Duration.ofHours(1))

then:
1 * httpClient.request(_) >> Stub(HttpRequest.Builder) {
    get() >> Stub(HttpRequest) {
        response() >> Stub(HttpResponse) {
            statusCode() >> 200
            body(_) >> documentBody
        }
    }
}
```

Two Groovy traps when stubbing these interfaces:
- Do not name a helper parameter after the method being stubbed. In
  `Stub(HttpResponse) { body(_) >> body }` Groovy resolves `body` to the parameter and the
  stub silently fails to match. Name it `documentBody`.
- Interactions declared in a `then:` block only cover the preceding `when:`. A call made in
  `given:` to pre-populate state has no interaction registered and returns `null`. Use
  successive `when:`/`then:` pairs instead.

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
