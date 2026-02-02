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
| Issue/validate tokens | Token Handler | `plugin-type-token.md` (when created) |
| See all types | All types | `plugin-types.md` |

---

## SDK Service Quick Lookup

| Service | Package | Purpose | Doc Location |
|---------|---------|---------|--------------|
| UserCredentialManager | `se.curity.identityserver.sdk.service.credential` | Verify passwords | `sdk-services.md` → Section 1 |
| ExceptionFactory | `se.curity.identityserver.sdk.service` | Create exceptions | `sdk-services.md` → Section 2 |
| SessionManager | `se.curity.identityserver.sdk.service` | Store session data | `sdk-services.md` → Section 3 |
| AccountManager | `se.curity.identityserver.sdk.service` | Look up accounts | *(inject via config)* |
| SmsSender | `se.curity.identityserver.sdk.service` | Send SMS | *(inject via config)* |
| SLF4J Logger | `org.slf4j` | Logging | `sdk-services.md` → Section 5 |

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

---

## Document Map

| Document | Primary Purpose | When to Use |
|----------|----------------|-------------|
| `INSTRUCTIONS.md` | Agent instructions | Start here for overview |
| `plugin-system.md` | System fundamentals | Understand plugin lifecycle |
| `plugin-types.md` | Type selector | Choose plugin type |
| `plugin-type-authenticator.md` | Authenticator guide | Implement authenticator |
| `request-handlers.md` | Handler patterns | Implement GET/POST logic |
| `sdk-services.md` | Service API reference | Look up service methods |
| `attributes.md` | Attribute framework | Work with user/auth data |
| `templating.md` | UI templates | Create user interfaces |
| `testing.md` | Test framework | Write unit tests |
| `recipes.md` | Working examples | Copy complete implementations |
| `build-and-deployment.md` | Build system | Configure Gradle, deploy |
| `quick-reference.md` | This file | Find information fast |
