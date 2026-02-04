# Unit Testing with Spock

Curity plugin projects use **Spock Framework** for unit testing, written in **Groovy**. Spock provides powerful mocking, assertions, and expressive test syntax.

## Setup

### Dependencies

Add to `build.gradle`:

```gradle
plugins {
    id 'org.jetbrains.kotlin.jvm' version '2.2.0'
    id 'groovy'  // Required for Spock tests
}

dependencies {
    // Test dependencies
    testImplementation 'org.spockframework:spock-core:2.3-groovy-4.0'
    testImplementation 'org.apache.groovy:groovy-all:4.0.15'
    testImplementation 'se.curity.identityserver:identityserver.sdk:10.6.1'
    testImplementation 'org.slf4j:slf4j-api:2.0.12'
    testImplementation 'org.jetbrains.kotlin:kotlin-stdlib:2.2.0'
    testImplementation 'jakarta.validation:jakarta.validation-api:3.0.0'
}
```

### Directory Structure

```
src/
├── main/
│   └── kotlin/
│       └── io/curity/identityserver/plugin/yourplugin/
│           ├── YourRequestHandler.kt
│           └── YourConfig.kt
└── test/
    └── groovy/
        └── io/curity/identityserver/plugin/yourplugin/
            ├── YourRequestHandlerSpec.groovy
            └── OtherSpec.groovy
```

**Conventions**:
- Test files end with `Spec.groovy`
- Located in `src/test/groovy/` matching the package structure
- Test class names match source class names with `Spec` suffix

## Spock Basics

### Test Structure

Spock uses a BDD-style structure with labeled blocks:

```groovy
class MyHandlerSpec extends Specification {

    def "descriptive test name with spaces"() {
        given: "setup conditions"
        def handler = new MyHandler(config)
        
        when: "action is performed"
        def result = handler.doSomething()
        
        then: "verify outcomes"
        result != null
        result.success == true
    }
}
```

**Block Types**:
- `given:` (or `setup:`) - Set up test fixtures
- `when:` - Execute the action being tested
- `then:` - Verify expected outcomes
- `expect:` - Combined when/then for simple tests
- `where:` - Data-driven test parameters
- `cleanup:` - Clean up resources (optional)

### Mocking with Spock

Spock has built-in mocking capabilities:

```groovy
def "test with mocks"() {
    given: "mocked dependencies"
    def config = Mock(MyConfig)
    def service = Mock(MyService)
    
    // Configure mock behavior
    config.getService() >> service
    service.doSomething("input") >> "output"
    
    when: "using the mocks"
    def result = service.doSomething("input")
    
    then: "verify interactions"
    1 * service.doSomething("input")  // Verify called once
    result == "output"
}
```

**Mock Types**:
- `Mock()` - Standard mock with behavior stubbing
- `Stub()` - Lighter mock, only stubbing (no interaction verification)
- `Spy()` - Partial mock (calls real methods unless stubbed)

**Stubbing Syntax**:
- `method() >> returnValue` - Return value
- `method() >>> [val1, val2]` - Return sequence
- `method() >> { args -> ... }` - Closure for complex logic
- `method() >> { throw new Exception() }` - Throw exception

**Verification Syntax**:
- `1 * method()` - Called exactly once
- `0 * method()` - Never called
- `(1.._) * method()` - At least once
- `_ * method()` - Any number of times
- `1 * method({ it.contains("test") })` - With argument constraint

### Assertions

Spock assertions are simple boolean expressions:

```groovy
then:
result != null              // Basic assertion
result.size() == 3         // Equality
result.contains("test")    // Method calls
result instanceof String   // Type checks
!result.isEmpty()          // Negation
```

**Power Assertions**: When assertions fail, Spock shows detailed output:

```
Condition not satisfied:

result.size() == 3
|      |      |
|      2      false
["a", "b"]
```

### Exception Testing

```groovy
def "test that exception is thrown"() {
    given:
    def handler = new MyHandler(config)
    
    when:
    handler.doSomethingDangerous()
    
    then:
    def exception = thrown(RuntimeException)
    exception.message == "expected message"
}

def "test that no exception is thrown"() {
    when:
    handler.doSomethingSafe()
    
    then:
    notThrown(Exception)
}
```

## Testing Request Handlers

### Basic Pattern

```groovy
class MyRequestHandlerSpec extends Specification {

    MyConfig config
    def exceptionFactory
    def sessionManager
    MyRequestHandler handler
    def response

    def setup() {
        // Create mocks (use def for SDK types to avoid import issues)
        config = Mock(MyConfig)
        exceptionFactory = Mock()
        sessionManager = Mock()
        response = Mock()

        // Configure config mock to return dependencies
        config.getExceptionFactory() >> exceptionFactory
        config.getSessionManager() >> sessionManager

        // Create handler
        handler = new MyRequestHandler(config)
    }

    def "GET request should display form"() {
        given: "a GET request model"
        def requestModel = Mock(MyRequestModel)

        when: "handling the GET request"
        def result = handler.get(requestModel, response)

        then: "response model is set"
        1 * response.setResponseModel(_, _)

        and: "result is empty"
        !result.isPresent()
    }
}
```

### Testing POST Handlers

```groovy
def "POST with valid input should succeed"() {
    given: "a POST request"
    def requestModel = Mock(MyPostRequestModel)
    requestModel.validatedField >> "value"

    and: "dependencies are configured"
    def service = Mock()
    config.getService() >> service
    service.process("value") >> "success"

    when: "handling the POST request"
    def result = handler.post(requestModel, response)

    then: "service is called"
    1 * service.process("value")

    and: "authentication succeeds"
    result.isPresent()
    result.get() != null
}
```

### Testing Session Interactions

```groovy
def "should store data in session"() {
    given:
    def requestModel = Mock(MyPostRequestModel)
    
    when:
    handler.post(requestModel, response)
    
    then: "data is stored in session"
    1 * sessionManager.put({ 
        it.name == "key" && 
        it.value.value == "expectedValue" 
    })
}

def "should retrieve data from session"() {
    given: "session contains data"
    def attr = Attribute.of("key", "value")
    sessionManager.get("key") >> attr
    
    when:
    def result = handler.post(requestModel, response)
    
    then: "data is used correctly"
    result.isPresent()
}
```

### Testing Error Cases

```groovy
def "should show error on invalid input"() {
    given:
    def requestModel = Mock(MyPostRequestModel)
    requestModel.validatedField >> "invalid"
    
    when:
    def result = handler.post(requestModel, response)
    
    then: "error is displayed"
    1 * response.setResponseModel({ 
        it.data['_error'] != null 
    }, _)
    
    and: "result is empty"
    !result.isPresent()
}

def "should throw exception on system error"() {
    given:
    def requestModel = Mock(MyPostRequestModel)
    def systemException = new RuntimeException("error")
    exceptionFactory.internalServerException(_, _) >> systemException
    
    and: "service fails"
    def service = Mock()
    config.getService() >> service
    service.process(_) >> { throw new Exception() }
    
    when:
    handler.post(requestModel, response)
    
    then:
    def exception = thrown(RuntimeException)
    exception == systemException
}
```

## Testing with Attributes Framework

### Testing MapAttributeValue

```groovy
def "should store account in session as MapAttributeValue"() {
    given:
    def requestModel = Mock(MyPostRequestModel)
    def account = Mock()
    
    when:
    handler.post(requestModel, response)
    
    then: "account is wrapped in MapAttributeValue"
    1 * sessionManager.put({ 
        it.name == "account" && 
        it.value instanceof MapAttributeValue 
    })
}
```

### Testing SubjectAttributes

```groovy
def "should create subject with account attribute"() {
    given:
    def accountAttribute = Attribute.of("account", MapAttributeValue.of([:]))
    sessionManager.get("account") >> accountAttribute
    
    when:
    def result = handler.post(requestModel, response)
    
    then:
    result.isPresent()
    // SubjectAttributes is tested implicitly through AuthenticationResult
}
```

## Data-Driven Tests

Test multiple scenarios with `where` block:

```groovy
def "should validate OTP format: #scenario"() {
    given:
    def requestModel = Mock(OtpPostRequestModel)
    requestModel.validatedOtp >> otp
    
    when:
    def result = handler.validateOtp(requestModel)
    
    then:
    result == expected
    
    where:
    scenario          | otp      | expected
    "valid 6 digits"  | "123456" | true
    "too short"       | "123"    | false
    "too long"        | "1234567"| false
    "non-numeric"     | "abc123" | false
}
```

## Best Practices

### 1. Use Descriptive Test Names

```groovy
// Good - describes behavior
def "POST with valid password should verify credentials and redirect to OTP screen"()

// Bad - too vague
def "test post"()
```

### 2. Use `def` for SDK Mock Types

Avoid importing SDK classes in tests - use `def` to prevent compilation issues:

```groovy
// Good - no import needed
def accountManager = Mock()
def account = Mock()

// Bad - requires importing SDK classes
AccountManager accountManager = Mock(AccountManager)
AccountAttributes account = Mock(AccountAttributes)
```

### 3. Separate Test Blocks

Use `and:` to organize complex tests:

```groovy
then: "password is verified"
1 * userCredentialManager.verify(_, _)

and: "SMS is sent"
1 * smsSender.sendSms(_, _)

and: "session is updated"
1 * sessionManager.put(_)
```

### 4. Mock Only What's Needed

Don't over-mock - only stub methods that are actually called:

```groovy
// Good - only stub what's used
config.getSessionManager() >> sessionManager

// Bad - unnecessary stubbing
config.getSessionManager() >> sessionManager
config.getUnusedService() >> Mock()
```

### 5. Test Both Success and Failure Paths

Always test:
- Happy path (valid input, successful processing)
- Error cases (invalid input, missing data)
- Edge cases (null values, empty strings)
- Exception scenarios (service failures, session expiration)

### 6. Keep Tests Independent

Each test should be self-contained:

```groovy
def setup() {
    // Reset mocks for each test
    config = Mock(MyConfig)
    handler = new MyRequestHandler(config)
}
```

### 7. Use Meaningful Assertion Messages

For complex conditions, add explanatory labels:

```groovy
then: "authentication result contains subject and account"
result.isPresent()
def authResult = result.get()
authResult != null
```

## Running Tests

```bash
# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests MyHandlerSpec

# Run specific test method
./gradlew test --tests "MyHandlerSpec.POST with valid input should succeed"

# Run tests with detailed output
./gradlew test --info

# Generate HTML test report
./gradlew test
# Report: build/reports/tests/test/index.html
```

## Complete Example

```groovy
package io.curity.identityserver.plugin.myauth

import se.curity.identityserver.sdk.attribute.Attribute
import se.curity.identityserver.sdk.service.credential.CredentialVerificationResult
import se.curity.identityserver.sdk.web.Response
import spock.lang.Specification

class PasswordHandlerSpec extends Specification {

    MyConfig config
    def userCredentialManager
    def sessionManager
    def exceptionFactory
    PasswordHandler handler
    def response

    def setup() {
        config = Mock(MyConfig)
        userCredentialManager = Mock()
        sessionManager = Mock()
        exceptionFactory = Mock()
        response = Mock()

        config.getUserCredentialManager() >> userCredentialManager
        config.getSessionManager() >> sessionManager
        config.getExceptionFactory() >> exceptionFactory

        handler = new PasswordHandler(config)
    }

    def "POST with valid password should authenticate"() {
        given: "valid credentials"
        def requestModel = Mock(PasswordPostRequestModel)
        requestModel.validatedUsername >> "alice"
        requestModel.validatedPassword >> "secret"

        and: "verification succeeds"
        userCredentialManager.verify(_, "secret") >> 
            CredentialVerificationResult.Accepted.instance

        when: "handling POST"
        def result = handler.post(requestModel, response)

        then: "credentials are verified"
        1 * userCredentialManager.verify(_, "secret")

        and: "authentication succeeds"
        result.isPresent()
    }

    def "POST with invalid password should show error"() {
        given: "invalid credentials"
        def requestModel = Mock(PasswordPostRequestModel)
        requestModel.validatedUsername >> "alice"
        requestModel.validatedPassword >> "wrong"

        and: "verification fails"
        userCredentialManager.verify(_, "wrong") >> 
            CredentialVerificationResult.Rejected.instance

        when: "handling POST"
        def result = handler.post(requestModel, response)

        then: "error is shown"
        1 * response.setResponseModel({ 
            it.data['_error'] != null 
        }, _)

        and: "authentication fails"
        !result.isPresent()
    }
}
```

## Additional Resources

- [Spock Framework Documentation](https://spockframework.org/spock/docs/2.3/all_in_one.html)
- [Groovy Documentation](https://groovy-lang.org/documentation.html)
- Curity SDK JavaDoc (included with SDK)
