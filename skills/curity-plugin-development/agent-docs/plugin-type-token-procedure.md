# Token Procedure Plugin – Agent Reference

**Reference implementation**: [curityio/external-idp-token-exchange](https://github.com/curityio/external-idp-token-exchange) — validates an external IdP JWT and issues a Curity access token via RFC 8693 token exchange.

## 1. When to Use

Use a **Token Procedure Plugin** when you need to:

- **Customize token issuance** for a specific OAuth flow (authorization code, client credentials, token exchange, etc.)
- **Add custom claims** to access tokens or ID tokens
- **Validate incoming tokens** from external identity providers (token exchange)
- **Control delegation** and scope behavior during token issuance
- **Implement custom token exchange** between external and internal tokens

Token procedures replace or extend the default token issuance behavior for one or more OAuth grant types.

## 2. Key Interfaces

| Interface | Package | Purpose |
|-----------|---------|---------|
| `TokenProcedurePluginDescriptor<C>` | `se.curity.identityserver.sdk.plugin.descriptor` | Plugin entry point |
| `OAuthTokenExchangeTokenProcedure` | `se.curity.identityserver.sdk.procedure.token` | Token exchange flow |
| `AuthorizationCodeTokenProcedure` | `se.curity.identityserver.sdk.procedure.token` | Authorization code flow |
| `ClientCredentialsTokenProcedure` | `se.curity.identityserver.sdk.procedure.token` | Client credentials flow |
| `RefreshTokenProcedure` | `se.curity.identityserver.sdk.procedure.token` | Refresh token flow |
| `RopcTokenProcedure` | `se.curity.identityserver.sdk.procedure.token` | Resource Owner Password Credentials |
| `AssistedTokenProcedure` | `se.curity.identityserver.sdk.procedure.token` | Assisted token flow |
| `DeviceCodeTokenProcedure` | `se.curity.identityserver.sdk.procedure.token` | Device code flow |

## 3. Architecture

```
TokenProcedurePluginDescriptor
  ├── getConfigurationType()
  ├── getPluginImplementationType()
  └── getOAuthTokenEndpoint*Procedure()  ← one method per supported flow
        └── TokenProcedure.run(Context) → ResponseModel
```

The descriptor returns the procedure class for each OAuth flow it handles. The server calls `run()` with a context object providing access to request data, token issuers, and delegation management.

## 4. Supported Flow Types

The `TokenProcedurePluginDescriptor` interface has methods for each OAuth flow. Override only the flows your plugin handles:

| Descriptor Method | Procedure Interface | OAuth Grant |
|-------------------|-------------------|-------------|
| `getOAuthTokenEndpointOAuthTokenExchangeTokenProcedure()` | `OAuthTokenExchangeTokenProcedure` | Token Exchange (RFC 8693) |
| `getOAuthTokenEndpointAuthorizationCodeTokenProcedure()` | `AuthorizationCodeTokenProcedure` | Authorization Code |
| `getOAuthTokenEndpointClientCredentialsTokenProcedure()` | `ClientCredentialsTokenProcedure` | Client Credentials |
| `getOAuthTokenEndpointRefreshTokenProcedure()` | `RefreshTokenProcedure` | Refresh Token |
| `getOAuthTokenEndpointRopcTokenProcedure()` | `RopcTokenProcedure` | Resource Owner Password |
| `getOAuthTokenEndpointDeviceCodeTokenProcedure()` | `DeviceCodeTokenProcedure` | Device Authorization |
| `getOAuthTokenEndpointAssistedTokenProcedure()` | `AssistedTokenProcedure` | Assisted Token |

## 5. Token Exchange — Complete Example

This is the most common custom token procedure. It validates an external token and issues a new internal token.

### Descriptor (Java)

```java
package com.example.plugin.descriptor;

import com.example.plugin.MyTokenExchangeProcedure;
import com.example.plugin.config.MyTokenProcedureConfig;
import se.curity.identityserver.sdk.plugin.descriptor.TokenProcedurePluginDescriptor;
import se.curity.identityserver.sdk.procedure.token.OAuthTokenExchangeTokenProcedure;

public final class MyTokenProcedureDescriptor
        implements TokenProcedurePluginDescriptor<MyTokenProcedureConfig>
{
    @Override
    public Class<? extends OAuthTokenExchangeTokenProcedure> getOAuthTokenEndpointOAuthTokenExchangeTokenProcedure()
    {
        return MyTokenExchangeProcedure.class;
    }

    @Override
    public String getPluginImplementationType()
    {
        return "my-token-exchange";
    }

    @Override
    public Class<? extends MyTokenProcedureConfig> getConfigurationType()
    {
        return MyTokenProcedureConfig.class;
    }
}
```

### Configuration (Java)

```java
package com.example.plugin.config;

import se.curity.identityserver.sdk.config.Configuration;
import se.curity.identityserver.sdk.config.annotation.DefaultLong;
import se.curity.identityserver.sdk.config.annotation.DefaultService;
import se.curity.identityserver.sdk.config.annotation.DefaultString;
import se.curity.identityserver.sdk.config.annotation.Description;
import se.curity.identityserver.sdk.service.ExceptionFactory;
import se.curity.identityserver.sdk.service.HttpClient;
import se.curity.identityserver.sdk.service.issuer.DefaultJwtAccessTokenIssuerProvider;

public interface MyTokenProcedureConfig extends Configuration
{
    ExceptionFactory getExceptionFactory();

    @DefaultService
    DefaultJwtAccessTokenIssuerProvider getJwtAccessTokenIssuerProvider();

    @DefaultString("https://external-idp.example.com/.well-known/openid-configuration")
    @Description("The external IdP's OIDC discovery URL for JWKS resolution")
    String getMetadataURL();

    @Description("Clock skew tolerance in seconds")
    @DefaultLong(2)
    Long getClockSkew();

    HttpClient getHttpClient();
}
```

### Token Exchange Procedure (Java)

```java
package com.example.plugin;

import com.example.plugin.config.MyTokenProcedureConfig;
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

public final class MyTokenExchangeProcedure implements OAuthTokenExchangeTokenProcedure
{
    private static final Logger _logger = LoggerFactory.getLogger(MyTokenExchangeProcedure.class);
    private final MyTokenProcedureConfig _configuration;

    public MyTokenExchangeProcedure(MyTokenProcedureConfig configuration)
    {
        _configuration = configuration;
    }

    @Override
    public ResponseModel run(OAuthTokenExchangeUnInitializedTokenProcedurePluginContext context)
    {
        // 1. Get the presented subject token
        String presentedToken = context.getSubjectTokenValue();
        if (presentedToken == null)
        {
            throw _configuration.getExceptionFactory()
                    .badRequestException(ErrorCode.TOKEN_ISSUANCE_ERROR, "Missing subject token");
        }

        // 2. Validate the external token (implement your own validation)
        //    This typically involves JWKS fetch, signature verification, claims validation
        String subject = validateExternalToken(presentedToken);

        // 3. Initialize the context with subject, audiences, and scopes
        Set<String> audiences = context.getClient().getAudiences();
        Set<String> scopes = context.getClient().getScopeNames();

        OAuthTokenExchangeTokenProcedurePluginContext fullContext = context.getInitializedContext(
                SubjectAttributes.of(subject),
                ContextAttributes.empty(),
                audiences,
                scopes
        );

        // 4. Build token data (can add custom claims)
        var tokenData = fullContext.getDefaultAccessTokenData()
                .with(Attribute.of("external_subject", subject));

        var delegation = fullContext.getDefaultDelegationData();

        // 5. Issue the access token
        try
        {
            @Nullable AccessTokenIssuer issuer = _configuration.getJwtAccessTokenIssuerProvider()
                    .getDefaultJwtAccessTokenIssuer();
            if (issuer == null)
            {
                throw _configuration.getExceptionFactory()
                        .badRequestException(ErrorCode.TOKEN_ISSUANCE_ERROR, "JWT issuer not configured");
            }

            @Nullable String accessToken = issuer.issue(
                    AccessTokenAttributes.of(tokenData),
                    fullContext.issueDelegation(delegation)
            );

            if (accessToken == null)
            {
                throw _configuration.getExceptionFactory()
                        .badRequestException(ErrorCode.TOKEN_ISSUANCE_ERROR, "Failed to issue access token");
            }

            // 6. Build the response
            Map<String, Object> responseData = new HashMap<>();
            responseData.put("access_token", accessToken);
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

    private String validateExternalToken(String token)
    {
        // Implement JWT validation: fetch JWKS, verify signature, check claims
        // Throw exception if invalid
        // Return the subject claim
        throw new UnsupportedOperationException("Implement token validation");
    }
}
```

### Service Descriptor File

`src/main/resources/META-INF/services/se.curity.identityserver.sdk.plugin.descriptor.TokenProcedurePluginDescriptor`:
```
com.example.plugin.descriptor.MyTokenProcedureDescriptor
```

## 6. Token Exchange Context API

The `OAuthTokenExchangeUnInitializedTokenProcedurePluginContext` provides:

| Method | Purpose |
|--------|---------|
| `getSubjectTokenValue()` | The incoming subject token string |
| `getClient()` | The OAuth client making the request (`getAudiences()`, `getScopeNames()`) |
| `getRequest()` | The HTTP request (access form parameters, headers) |
| `getInitializedContext(SubjectAttributes, ContextAttributes, Set<String> audiences, Set<String> scopes)` | Initialize for token issuance |

The `OAuthTokenExchangeTokenProcedurePluginContext` (after initialization) provides:

| Method | Purpose |
|--------|---------|
| `getDefaultAccessTokenData()` | Default access token attributes (can add/modify with `.with()`) |
| `getDefaultDelegationData()` | Default delegation data |
| `issueDelegation(DelegationData)` | Issue a delegation and get the delegation object |

## 7. Adding Custom Claims

Use `.with()` on the default token data to add claims:

```java
var tokenData = fullContext.getDefaultAccessTokenData()
        .with(Attribute.of("user_id", subject))
        .with(Attribute.of("org_id", organizationId))
        .with(Attribute.of("roles", String.join(" ", userRoles)));
```

## 8. Bundled Dependencies

Token procedures often need JWT libraries. These must be bundled with the plugin JAR since they are not provided by the server:

```groovy
plugins {
    id 'java'
}

dependencies {
    compileOnly 'se.curity.identityserver:identityserver.sdk:10.6.1'
    compileOnly 'org.slf4j:slf4j-api:2.0.12'

    // Bundled (NOT compileOnly) — these are packaged with the plugin
    implementation 'org.bitbucket.b_c:jose4j:0.9.6'
    implementation 'org.json:json:20250517'
}

// Copy runtime dependencies to output directory alongside the plugin JAR
tasks.register('createDeployDir', Copy) {
    from jar
    from configurations.runtimeClasspath
    into "${layout.buildDirectory.get()}/deploy/${project.name}"
}
```

## 9. Error Handling

Token procedures should return error responses using either:

1. **ExceptionFactory** — for immediate error responses:
```java
throw _configuration.getExceptionFactory()
        .badRequestException(ErrorCode.TOKEN_ISSUANCE_ERROR, "Invalid subject token");
```

2. **ResponseModel.problemResponseModel** — for structured error responses:
```java
return ResponseModel.problemResponseModel("invalid_grant", "The subject token is expired");
```

## 10. Testing

```groovy
import spock.lang.Specification
import se.curity.identityserver.sdk.procedure.token.context.OAuthTokenExchangeUnInitializedTokenProcedurePluginContext
import se.curity.identityserver.sdk.service.ExceptionFactory

class MyTokenExchangeProcedureSpec extends Specification {

    def "run throws when subject token is null"() {
        given:
        def config = Mock(MyTokenProcedureConfig) {
            getExceptionFactory() >> Mock(ExceptionFactory) {
                badRequestException(_, _) >> new RuntimeException("bad request")
            }
        }
        def context = Mock(OAuthTokenExchangeUnInitializedTokenProcedurePluginContext) {
            getSubjectTokenValue() >> null
        }
        def procedure = new MyTokenExchangeProcedure(config)

        when:
        procedure.run(context)

        then:
        thrown(RuntimeException)
    }
}
```
