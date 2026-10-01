# Application Plugins

Application plugins expose custom HTTP endpoints at a configurable mount point under the Curity Identity Server. Use them to implement REST APIs (SCIM, admin APIs, health endpoints, custom webhooks, etc.) that reuse the server's configured services — `AccountManager`, `SessionManager`, `Json`, `ExceptionFactory`, credential managers, and so on.

## When to use

- Implementing a **REST API backed by Curity services** — SCIM, custom user admin endpoints, bespoke integration APIs.
- Exposing **read-only system endpoints** (health, diagnostics) that need access to the server's runtime.
- Building a **multi-route web endpoint** where different paths map to different handlers.

If you need to authenticate users interactively, use an **authenticator** plugin instead. If you need to emit events, use an **event listener**.

## Descriptor

```kotlin
import se.curity.identityserver.sdk.plugin.descriptor.ApplicationPluginDescriptor
import se.curity.identityserver.sdk.web.RequestHandler

class MyAppPluginDescriptor : ApplicationPluginDescriptor<MyAppConfiguration> {

    override fun getPluginImplementationType(): String = "my-app"

    override fun getConfigurationType(): Class<out MyAppConfiguration> =
        MyAppConfiguration::class.java

    override fun getAnonymousRequestHandlerTypes(): Map<String, Class<out RequestHandler<*>>> =
        mapOf(
            "Users"           to UsersCollectionRequestHandler::class.java,
            "Users/:id"       to UserResourceRequestHandler::class.java,
            "Schemas"         to SchemasCollectionRequestHandler::class.java,
            "Schemas/:id"     to SchemaResourceRequestHandler::class.java,
            "ServiceProviderConfig" to ServiceProviderConfigRequestHandler::class.java
        )
}
```

**Routing key syntax:**

- Literal path segments are matched verbatim (`Users`, `Schemas`).
- `:name` is a **path variable** that matches a single path segment. Retrieve its value in the handler with `request.getPathParameterValue("name")`.
- The mount point itself is configured by the server administrator (e.g. `/app/scim-app`), so the full URL for `Users/:id` becomes `https://idsvr.example.com/app/scim-app/Users/abc-123`.
- `getAnonymousRequestHandlerTypes()` exposes handlers that do **not** require authentication. Use `getAuthenticatedRequestHandlerTypes()` if the server should enforce authentication before dispatching.

Register the descriptor via `META-INF/services/se.curity.identityserver.sdk.plugin.descriptor.ApplicationPluginDescriptor`.

## Configuration

Application configurations follow the same pattern as other plugin types — expose injected services through getter methods on the config interface:

```kotlin
interface MyAppConfiguration : Configuration {
    @get:Description("Account manager used to back the /Users endpoints")
    val accountManager: AccountManager

    @get:Description("Base URL used when computing absolute 'location' URLs")
    val resourceBaseUrl: Optional<String>

    @get:DefaultInteger(100)
    val maxPageSize: Int

    fun getJson(): Json
    fun getExceptionFactory(): ExceptionFactory
}
```

Administrators bind a specific `AccountManager` (or any other data-access provider) to the plugin in the config.xml / admin UI, so the plugin code stays decoupled from the storage backend.

## Authenticating a user with `TokenServiceOAuthClient`

**Since SDK 11.4.0. Application plugins only.**

`se.curity.identityserver.sdk.service.oauth.TokenServiceOAuthClient` is a reference to an
OAuth client defined on the OAuth profile linked to the profile hosting the plugin. It runs
an authorization code flow against that local profile, so the plugin never re-declares
endpoints, client secrets or keys — and never hand-rolls an OAuth client.

Declare it on the configuration interface; the administrator selects one of the clients on
the linked OAuth profile:

```kotlin
import se.curity.identityserver.sdk.service.oauth.TokenServiceOAuthClient

interface MyAppConfiguration : Configuration {
    @get:Description("The OAuth client used to authenticate the user")
    val oauthClient: TokenServiceOAuthClient

    @get:DefaultService
    val httpClient: HttpClient

    val sessionManager: SessionManager
    fun getExceptionFactory(): ExceptionFactory
}
```

### Starting the flow

`startAuthorizationCodeFlow` builds the authorization URL and returns a
`TokenServiceOAuthClient.AuthorizationCodeFlow` carrying the state, PKCE verifier, redirect
URI and response mode. It extends `MapAttributeValue`, so it stores into an `Attribute`
with no conversion:

```kotlin
val flow = _config.oauthClient.startAuthorizationCodeFlow(_config.httpClient)
_sessionManager.put(Attribute.of("authz-flow", flow))

throw _exceptionFactory.redirectException(flow.authorizationUrl)
```

There is a second overload taking `Map<String, Collection<String>>` of extra authorization
parameters, appended to the URL as-is — whitelisting them is the plugin's responsibility.

### Handling the callback

```kotlin
import se.curity.identityserver.sdk.service.oauth.TokenServiceOAuthClient.AuthorizationCodeFlow

val attribute = _sessionManager.remove("authz-flow")
    ?: throw _exceptionFactory.badRequestException(
        ErrorCode.INVALID_INPUT, "No in-flight authorization code flow")
val flowValue = attribute.attributeValue as? MapAttributeValue
    ?: throw _exceptionFactory.internalServerException(
        ErrorCode.GENERIC_ERROR, "Malformed authorization code flow in session")

val tokens = _config.oauthClient.handleAuthorizationCodeFlowCallback(
    AuthorizationCodeFlow.of(flowValue),
    callbackParameters,   // "code" + "state", or a JARM "response" JWT; may include "iss"
    _config.httpClient
)

val claims = tokens.idTokenClaims()
    ?: throw _exceptionFactory.internalServerException(
        ErrorCode.EXTERNAL_SERVICE_ERROR, "No id_token in token response")
val subject = claims["sub"]?.toString() ?: /* ... */
```

`handleAuthorizationCodeFlowCallback` validates `state` and `iss` against the started flow,
exchanges the code at the linked profile's token endpoint, and validates the returned ID
token's signature, expiry, issuer and audience. When the linked client uses JARM, the
`response` JWT is validated before `code`/`state`/`iss` are read out of it.

### Configuring the referenced OAuth client

The client on the linked OAuth profile must authenticate with a **symmetric key**, not the
hashed `<secret>` that config-backed clients normally carry. The plugin authenticates *as*
that client against the token endpoint, and a one-way hash cannot be replayed. A client with
only a `<secret>` fails at request time, not at startup — the endpoint returns
`{"error_code":"configuration_error"}` with a 500, and the server log carries the real reason:

```
Client with id=my-client must be configured with a symmetric key for authentication
```

The `symmetric-key` leaf in turn requires symmetrically-signed-JWT client authentication to be
enabled on the profile. Without it the server refuses to boot:

```
CDB boot error: Init transaction failed to validate:
/base:profiles/profile{token-service as:oauth-service}/settings/as:authorization-server/
client-store/config-backed/client{my-client}/symmetric-key: Symmetrically signed JWT Client
authentication must be enabled in the profile.
```

Both pieces together, on the OAuth profile the plugin's profile links to:

```xml
<authorization-server xmlns="https://curity.se/ns/conf/profile/oauth">
    <client-authentication>
        <symmetrically-signed-jwt>
            <signature-algorithm>HS256</signature-algorithm>
        </symmetrically-signed-jwt>
    </client-authentication>
    <client-store>
        <config-backed>
            <client>
                <id>my-client</id>
                <symmetric-key>data:text/plain;aes,v:S...</symmetric-key>
                <redirect-uris>https://localhost:8443/apps/my-app/callback</redirect-uris>
                <scope>openid</scope>
                <user-authentication/>
                <capabilities>
                    <code/>
                </capabilities>
            </client>
        </config-backed>
    </client-store>
</authorization-server>
```

The redirect URI registered on the client must be the one the SDK derives — observed to be the
plugin's configured base URL plus the callback handler's path, e.g.
`https://localhost:8443/apps/my-app/callback` for a handler registered under `callback`. Log
`flow.getRedirectUri()` at debug and assert on the `redirect_uri` in an integration test rather
than assuming it.

### Things that surprise people

- **PKCE is always used, and the SDK owns `state` and the redirect URI.** Do not generate
  your own `state` or compute your own callback URL — the SDK's values are what get
  validated. Read `flow.redirectUri` if you need to know what it chose (logging it at debug
  is worth it: the OAuth client on the linked profile must have exactly that URI registered,
  and it is not necessarily `${baseUrl}/callback`).
- **`OAuthTokens` is a record with nullable components, not `Optional`s.** Accessors are
  `accessToken()`, `accessTokenExpiresIn()`, `refreshToken()` and `idTokenClaims()`; all but
  the first are `@Nullable`. `idTokenClaims()` is a `Map<String, Object>` and is `null` when
  no ID token came back.
- **Build the callback parameter map from whatever is present**, rather than requiring
  `code` and `state`. A request model that throws when they are missing also throws on a
  legitimate `error=access_denied` callback, before the handler can report it — and it
  precludes JARM, where everything arrives in a single `response` JWT.
- The whole flow needs a session that survives the redirect, so persist the
  `AuthorizationCodeFlow` via `SessionManager`, an encrypted cookie or a database.

## Request handlers

Application plugins use `HttpRequestHandler<Request>` — it extends the plain `RequestHandler<Request>` and adds default (no-op) implementations for `put`, `patch`, `delete`, `trace`, and `options`. Override only the verbs the endpoint supports; reject the rest explicitly:

```kotlin
@Produces(Produces.ContentType.JSON)
class UserResourceRequestHandler(
    private val configuration: MyAppConfiguration,
) : HttpRequestHandler<Request> {

    override fun preProcess(request: Request, response: Response): Request = request

    override fun get(requestModel: Request, response: Response)   { /* ... */ }
    override fun put(request: Request, response: Response)        { /* ... */ }
    override fun patch(requestModel: Request, response: Response) { /* ... */ }
    override fun delete(request: Request, response: Response)     { /* ... */ }

    override fun post(requestModel: Request, response: Response) =
        writeMethodNotAllowed(response, "POST to this resource is not allowed.")
}
```

> **Use `HttpRequestHandler`, not plain `RequestHandler`.** `RequestHandler` only exposes `get()` and `post()`, so PUT/PATCH/DELETE would silently fall through to nothing. `HttpRequestHandler` gives you one method per verb and makes the endpoint's contract explicit. The defaults return `null` (an empty 200 body) — explicitly override rejected verbs to emit a proper 405 in your API's error format.

### `@Produces` content type

Annotate every handler with `@Produces(Produces.ContentType.JSON)` (or `HTML`) — the framework uses this to negotiate content type and pick the right response writer.

### Path variables

```kotlin
override fun get(requestModel: Request, response: Response) {
    val id = requestModel.getPathParameterValue("id")  // matches "Users/:id"
    if (id.isNullOrBlank()) throw exceptionFactory.notFoundException()
    // ...
}
```

`getPathParameterValue(name)` returns the raw (url-decoded) value of the named path segment, or `null` if the route didn't bind it.

### Query parameters

```kotlin
val filter = requestModel.getQueryParameterValues("filter").firstOrNull()
val startIndex = requestModel.getQueryParameterValues("startIndex").firstOrNull()?.toIntOrNull() ?: 1
```

### PUT, PATCH, DELETE — one method per verb

With `HttpRequestHandler` each verb has its own override. The signatures are:

```kotlin
override fun get(requestModel: Request, response: Response): Any?
override fun post(requestModel: Request, response: Response): Any?
override fun put(request: Request, response: Response): Any?
override fun patch(requestModel: Request, response: Response): Any?
override fun delete(request: Request, response: Response): Any?
override fun options(request: Request, response: Response): Any?
override fun trace(requestModel: Request, response: Response): Any?
```

All are `default` and return `null` — if you don't override, the server returns 200 with no body. Always override disallowed verbs with a 405 response so API clients see a deterministic error.

**Inspecting the method string.** `request.getMethod()` returns the uppercase HTTP verb as a `String`; `HttpMethod.of(method)` converts it to the typed enum. `Request` exposes `isGetRequest()`, `isPostRequest()`, and `isHeadRequest()` helpers — but no `isPutRequest()` / `isDeleteRequest()`, so with `HttpRequestHandler` you rarely need to branch on the method yourself.

**Legacy fallback.** On older SDKs without `HttpRequestHandler`, or on authenticator `RequestHandler` subtypes that don't expose verb-specific overrides, you can still dispatch from `post()`:

```kotlin
override fun post(requestModel: Request, response: Response) {
    when (requestModel.method.uppercase()) {
        "PUT"    -> handlePut(requestModel, response)
        "DELETE" -> handleDelete(requestModel, response)
        else     -> throw exceptionFactory.methodNotAllowed()
    }
}
```

Prefer the `HttpRequestHandler` overrides for application plugins.

## Writing JSON responses

Application plugins usually return JSON bodies rather than render templates. The pattern is:

```kotlin
import se.curity.identityserver.sdk.http.HttpStatus
import se.curity.identityserver.sdk.web.Response
import se.curity.identityserver.sdk.web.ResponseModel

fun writeJson(response: Response, status: HttpStatus, body: Map<String, Any?>) {
    response.setHttpStatus(status)
    response.setResponseModel(
        ResponseModel.mapResponseModel(body),
        Response.ResponseModelScope.ANY
    )
}
```

- `ResponseModel.mapResponseModel(Map)` wraps a JSON-serializable map.
- `Response.ResponseModelScope.ANY` applies the model to any status code; use `.SUCCESS` / `.FAILURE` to scope it. The two-arg overload `setResponseModel(ResponseModel, HttpStatus, HttpStatus...)` scopes by specific status codes.
- `response.setHttpStatus(status)` sets the HTTP status code independently from the model.

**Do not** go through `ExceptionFactory` for domain-specific error envelopes (e.g. SCIM `urn:...:Error`) — `ExceptionFactory` emits the server's generic error templates. Write the error envelope yourself via `setResponseModel` and pair it with the desired `HttpStatus`.

## Response headers

Mutate response headers through the builder returned by `response.getHeaders()`:

```kotlin
response.headers.header("Location", "https://idsvr.example.com/app/my-app/Users/$id")
response.headers.header("X-Request-Id", requestId)
response.headers.remove("Some-Header")
```

The builder is mutable — calls like `header(name, value)` update the response in place. `HttpHeaders.Builder.header()` accepts either a single `String` value or a `Collection<String>`.

## AccountManager cheatsheet

`AccountManager` is the canonical account storage abstraction. Key methods for application plugins:

| Method | Purpose |
|---|---|
| `getById(String): AccountAttributes?` | Lookup by account id (returns `null` if missing). |
| `getByUserName(String): AccountAttributes?` | Lookup by userName. |
| `getByEmail(String): AccountAttributes?` | Lookup by any registered email. |
| `getByPhone(String): AccountAttributes?` | Lookup by any registered phone number. |
| `createAccount(AccountAttributes): AccountAttributes` | Persist a new account, returns the stored representation (with server-assigned id). |
| `updateAccount(AccountAttributes): Boolean` | Replace the account by id; returns `true` on success. |
| `deleteAccount(AccountAttributes): Unit` | Remove the account. |

**There is no listing API.** `AccountManager` intentionally exposes only the four equality lookups above — you cannot iterate or page through accounts. When building a list-style endpoint, reject unsupported filters at the edge and only support filters that map to one of these lookups (e.g. SCIM `userName eq "..."`, `emails eq "..."`, `phoneNumbers eq "..."`, `id eq "..."`).

## AccountAttributes is already SCIM 2.0

`AccountAttributes` is the server's internal representation of a SCIM 2.0 user, so you rarely need a hand-rolled DTO:

```kotlin
// Inbound: client JSON → AccountAttributes
val account = AccountAttributes.fromMap(clientJson)

// Outbound: AccountAttributes → client JSON
val body = account.asMap()
```

**Important gotchas:**

- `AccountAttributes.asMap()` **always** includes `"schemas": ["urn:ietf:params:scim:schemas:core:2.0:User"]` — you cannot strip it, the attribute is intrinsic to the type.
- `AccountAttributes.fromMap()` silently ignores unknown / server-controlled fields you don't want to accept (e.g. `meta`) only if you remove them first. Sanitize the inbound map before calling `fromMap`.
- `AccountAttributes.PASSWORD`, `USER_NAME`, `EMAILS`, `PHONE_NUMBERS`, etc. expose the canonical attribute keys as `String` constants — use them instead of hardcoding.
- Mutations produce a new instance: `account.with(Attribute.of("id", newId))`, `account.withEmails(...)`, `account.addEmail(...)`.
- `AccountAttributes.of(userName, email, password)` is a convenience factory for minimal user construction.

## Minimal end-to-end example

```kotlin
@Produces(Produces.ContentType.JSON)
class UserResourceRequestHandler(
    private val configuration: MyAppConfiguration,
) : HttpRequestHandler<Request> {

    override fun preProcess(request: Request, response: Response): Request = request

    override fun get(requestModel: Request, response: Response) {
        val id = requireId(requestModel, response) ?: return
        val account = configuration.accountManager.getById(id)
        if (account == null) {
            writeError(response, HttpStatus.NOT_FOUND, "User '$id' not found")
            return
        }
        writeJson(response, HttpStatus.OK, account.asMap())
    }

    override fun put(request: Request, response: Response)    { /* replace by id */ }
    override fun delete(request: Request, response: Response) { /* delete by id */ }

    override fun post(requestModel: Request, response: Response) =
        writeMethodNotAllowed(response, "POST to this resource is not allowed.")

    override fun patch(requestModel: Request, response: Response) =
        writeMethodNotAllowed(response, "PATCH is not yet implemented.")
}
```

## Testing

Application handlers mock cleanly with Spock:

```groovy
def "GET returns the account when found"() {
    given:
    AccountManager accountManager = Mock()
    def config = Mock(MyAppConfiguration) {
        getAccountManager() >> accountManager
    }
    def handler = new UserResourceRequestHandler(config, Mock(ExceptionFactory))
    def request = Mock(Request) {
        getPathParameterValue("id") >> "abc-123"
    }
    def response = Mock(Response)
    ResponseModel captured = null

    when:
    handler.get(request, response)

    then:
    1 * accountManager.getById("abc-123") >> AccountAttributes.fromMap([id: "abc-123", userName: "alice"])
    1 * response.setHttpStatus(HttpStatus.OK)
    1 * response.setResponseModel(_, _) >> { ResponseModel m, _ -> captured = m }

    and:
    captured.getViewData()["userName"] == "alice"
}
```

Key testing patterns:

- `response.setResponseModel(_, _)` is easy to capture with `>> { ResponseModel m, _ -> captured = m }`; read the payload back via `captured.getViewData()`.
- Mock `Response.getHeaders()` to return a `Mock(HttpHeaders.Builder)` when asserting `Location` headers.
- For body parsing, stub `Json.fromJson(_ as String)` with `new JsonSlurper().parseText(it)` so the handler sees real parsed maps.
- Spock 2.4 on Groovy 4.0 does **not** bundle `groovy.json` — add `testImplementation 'org.apache.groovy:groovy-json:4.0.22'` before using `JsonSlurper`/`JsonOutput`.
