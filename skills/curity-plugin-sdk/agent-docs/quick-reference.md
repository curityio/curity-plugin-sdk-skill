# Quick Reference – Plugin Development

Task-oriented index for quickly finding documentation when implementing Curity plugins.

---

## Common Implementation Tasks

### Authentication & Credentials
| Task | Location |
|------|----------|
| Verify username/password | `sdk-services.md` → UserCredentialManager |
| Build complete username/password authenticator | `recipes.md` → Recipe 1 |
| Create SubjectAttributes | `attributes.md` → SubjectAttributes |
| Return authentication result | `sdk-services.md` → AuthenticationResult |
| Add account attributes to result | `attributes.md` → AuthenticationAttributes |

### Authentication Actions (Post-Authentication)
| Task | Location |
|------|----------|
| Deny authentication based on conditions | `plugin-type-authentication-action.md` → Pattern 1 |
| Enrich authentication with attributes | `plugin-type-authentication-action.md` → Pattern 2 |
| Prompt user for additional input | `plugin-type-authentication-action.md` → Pattern 3 |
| Access attributes from context | `plugin-type-authentication-action.md` → Accessing Attributes |
| Return success/failed/pending result | `plugin-type-authentication-action.md` → AuthenticationActionResult Types |

### Multi-Screen Flows
| Task | Location |
|------|----------|
| Store data between screens | `sdk-services.md` → SessionManager |
| Redirect to next screen | `sdk-services.md` → ExceptionFactory.redirectException() |
| Build two-screen OTP flow | `recipes.md` → Recipe 2 (SMS OTP) |
| Clear session data | `sdk-services.md` → SessionManager.remove() |

### Request Handling
| Task | Location |
|------|----------|
| Implement GET/POST handlers | `request-handlers.md` → Request handler flow |
| Validate user input | `request-handlers.md` → Request Model Validation |
| Display error messages | `request-handlers.md` → Exception handling |
| Handle form submissions | `request-handlers.md` → Example Handler |

### UI & Templates
| Task | Location |
|------|----------|
| Create template files | `templating.md` → Template path |
| Use Velocity syntax | `templating.md` → Velocity syntax |
| Display error messages | `templating.md` → Template contents |
| Apply Curity CSS styling | `templating.md` → Styling with Curity CSS Classes |
| Add localized messages | `templating.md` → Localization with message keys |

### Configuration & Services
| Task | Location |
|------|----------|
| Inject SDK services | `sdk-services.md` → Accessing SDK Services |
| Define configuration interface | `plugin-system.md` → Section 4 (Configuration) |
| Access configuration values | `plugin-system.md` → Section 4 |

### Error Handling
| Task | Location |
|------|----------|
| Throw redirect exception | `sdk-services.md` → ExceptionFactory.redirectException() |
| Throw internal server error | `sdk-services.md` → ExceptionFactory.internalServerException() |
| Handle external service errors | `recipes.md` → Recipe 4 (External Service Integration) |
| Use error codes | `sdk-services.md` → ErrorCode |

### Logging
| Task | Location |
|------|----------|
| Set up logger | `sdk-services.md` → SLF4J Logger |
| Log authentication events | `sdk-services.md` → SLF4J Logger → Best Practices |
| Log errors with exceptions | `sdk-services.md` → SLF4J Logger → Usage Levels |

### Testing
| Task | Location |
|------|----------|
| Write Spock tests | `testing.md` → Introduction |
| Mock SDK services | `testing.md` → Mocking Dependencies |
| Test request handlers | `recipes.md` → Recipe 5 (Basic Handler Test) |
| Test validation | `testing.md` → Testing Request Models |

### Build & Deployment
| Task | Location |
|------|----------|
| Configure Gradle build | `build-and-deployment.md` → Gradle Configuration |
| Mark server dependencies | `sdk-services.md` → Server-Provided Dependencies |
| Build plugin JAR | `build-and-deployment.md` → Building |
| Deploy to Curity server | `build-and-deployment.md` → Deployment |

---

## Plugin Types

### Which plugin type do I need?
| Requirement | Plugin Type | Documentation |
|-------------|-------------|---------------|
| Authenticate users | Authenticator | `plugin-type-authenticator.md` |
| CIBA / out-of-band authentication | Backchannel Authenticator | `plugin-type-backchannel-authenticator.md` |
| Post-authentication logic | Authentication Action | `plugin-type-authentication-action.md` |
| Listen to server events | Event Listener | `plugin-type-event-listener.md` |
| Customize token issuance / token exchange | Token Procedure | `plugin-type-token-procedure.md` |
| See all types | All types | `plugin-types.md` |

---

### Event Listeners
| Task | Location |
|------|----------|
| Listen to all events | `plugin-type-event-listener.md` → Section 6 |
| Filter by event type | `plugin-type-event-listener.md` → Section 6 (Specific Event Type) |
| Multiple listeners in one plugin | `plugin-type-event-listener.md` → Section 7 |
| Complete event listener example | `recipes.md` → Recipe 7 |

### Token Procedures
| Task | Location |
|------|----------|
| Implement token exchange | `plugin-type-token-procedure.md` → Section 5 |
| Add custom claims to tokens | `plugin-type-token-procedure.md` → Section 7 |
| Token exchange context API | `plugin-type-token-procedure.md` → Section 6 |
| Complete token exchange example | `recipes.md` → Recipe 8 |

### Configuration
| Task | Location |
|------|----------|
| Understand configuration types | `configuration.md` → Section 2 |
| Use default value annotations | `configuration.md` → Section 3 |
| Add validation constraints | `configuration.md` → Section 4 |
| Nested configuration interfaces | `configuration.md` → Section 6 |
| OneOf (sum type) configuration | `configuration.md` → Section 6 |
| EncryptedString for secrets | `configuration.md` → Section 9 |
| ConfigurationScope boundary | `configuration.md` → Section 8 |

---

## SDK Service Quick Lookup

| Service | Package | Purpose | Doc Location |
|---------|---------|---------|--------------|
| UserCredentialManager | `se.curity.identityserver.sdk.service.credential` | Verify passwords | `sdk-services.md` → Section 1 |
| ExceptionFactory | `se.curity.identityserver.sdk.service` | Create exceptions | `sdk-services.md` → Section 2 |
| SessionManager | `se.curity.identityserver.sdk.service` | Store session data | `sdk-services.md` → Section 3 |
| AccountManager | `se.curity.identityserver.sdk.service` | Look up accounts | `sdk-services.md` → Section 4 |
| SmsSender | `se.curity.identityserver.sdk.service.sms` | Send SMS | `sdk-services.md` → Section 5 |
| HttpClient | `se.curity.identityserver.sdk.service` | Low-level HTTP | `sdk-services.md` → Section 9 |
| WebServiceClientFactory | `se.curity.identityserver.sdk.service` | Create HTTP clients | `sdk-services.md` → Section 9 |
| WebServiceClient | `se.curity.identityserver.sdk.service` | API calls | `sdk-services.md` → Section 9 |
| Json | `se.curity.identityserver.sdk.service` | JSON serialization | `sdk-services.md` → Section 10 |
| EmailSender | `se.curity.identityserver.sdk.service` | Send emails | `sdk-services.md` → Section 11 |
| NonceTokenIssuer | `se.curity.identityserver.sdk.service` | Single-use tokens | `sdk-services.md` → Section 12 |
| UserPreferenceManager | `se.curity.identityserver.sdk.service` | Username cookie | `sdk-services.md` → Section 13 |
| Bucket | `se.curity.identityserver.sdk.service` | Key-value storage | `sdk-services.md` → Section 14 |
| Throttler | `se.curity.identityserver.sdk.service` | Rate limiting | `sdk-services.md` → Section 15 |
| SystemInformationProvider | `se.curity.identityserver.sdk.service` | System info | `sdk-services.md` → Section 16 |
| OriginalQueryExtractor | `se.curity.identityserver.sdk.service` | OAuth request params | `sdk-services.md` → Section 17 |
| RequestingOAuthClient | `se.curity.identityserver.sdk.service` | OAuth client info | `sdk-services.md` → Section 17 |
| SLF4J Logger | `org.slf4j` | Logging | `sdk-services.md` → Section 8 |

---

## Attribute Types Quick Lookup

| Type | Purpose | Doc Location |
|------|---------|--------------|
| SubjectAttributes | User identity | `attributes.md` → SubjectAttributes |
| AuthenticationAttributes | Auth result container | `attributes.md` → AuthenticationAttributes |
| ContextAttributes | Auth event metadata | `attributes.md` → ContextAttributes |
| AccountAttributes | User account data | `attributes.md` → AccountAttributes |
| Attribute | Single key-value pair | `attributes.md` → Attribute |

---

## File Structure Quick Lookup

| Component | File Location | Documentation |
|-----------|---------------|---------------|
| Descriptor | `src/main/kotlin/.../descriptor/` | `plugin-type-authenticator.md` → Section 2 |
| Configuration | `src/main/kotlin/.../` | `plugin-system.md` → Section 4 |
| Request Handler | `src/main/kotlin/.../` | `request-handlers.md` |
| Request Model | `src/main/kotlin/.../` | `request-handlers.md` → Validation |
| Templates | `src/main/resources/templates/authenticator/<plugin-name>/` | `templating.md` → Template path |
| Plugin Icon | `src/main/resources/icons/<plugin-type-name>.svg` | `plugin-system.md` → Section 6 |
| Tests | `src/test/groovy/.../` | `testing.md` |
| Build Config | `build.gradle` | `build-and-deployment.md` |

---

## Troubleshooting

| Problem | Check | Documentation |
|---------|-------|---------------|
| Template not found | Template path and file extension (.vm) | `templating.md` → Template path |
| Validation not working | POST request model with annotations | `request-handlers.md` → Validation |
| Session data missing | SessionManager put/get/remove | `sdk-services.md` → SessionManager |
| Compilation errors | Server-provided dependencies marked `compileOnly` | `sdk-services.md` → Section 6 |
| Plugin not loading | Descriptor and configuration setup | `plugin-system.md` → Sections 1-2 |
| Tests failing | Mock setup and expectations | `testing.md` → Mocking |

---

## End-to-End Examples

All examples in `recipes.md`:

1. **Recipe 1**: Username/Password Authenticator (single screen)
2. **Recipe 2**: SMS OTP Flow (multi-screen: password → OTP)
3. **Recipe 3**: Attribute Enrichment (adding account data)
4. **Recipe 4**: External Service Integration (HTTP calls, error handling)
5. **Recipe 5**: Basic Handler Test (Spock framework)
6. **Recipe 6**: Conditional Denial Action
7. **Recipe 7**: All-Events Logger (event listener)
8. **Recipe 8**: External IdP Token Exchange (token procedure)

---

## Document Map

| Document | Primary Purpose | When to Use |
|----------|----------------|-------------|
| `INSTRUCTIONS.md` | Agent instructions | Start here for overview |
| `plugin-system.md` | System fundamentals | Understand plugin lifecycle |
| `plugin-types.md` | Type selector | Choose plugin type |
| `plugin-type-authenticator.md` | Authenticator guide | Implement authenticator |
| `plugin-type-event-listener.md` | Event listener guide | Implement event listener |
| `plugin-type-token-procedure.md` | Token procedure guide | Implement token procedure |
| `request-handlers.md` | Handler patterns | Implement GET/POST logic |
| `sdk-services.md` | Service API reference | Look up service methods |
| `configuration.md` | Configuration deep dive | Annotations, constraints, nesting |
| `attributes.md` | Attribute framework | Work with user/auth data |
| `templating.md` | UI templates | Create user interfaces |
| `testing.md` | Test framework | Write unit tests |
| `recipes.md` | Working examples | Copy complete implementations |
| `build-and-deployment.md` | Build system | Configure Gradle, deploy |
| `quick-reference.md` | This file | Find information fast |
