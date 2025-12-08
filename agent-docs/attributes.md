# Attributes Framework

The Attributes framework provides a type-safe way to work with JSON-like data structures in Curity plugins.

## Overview

The Attributes framework consists of immutable classes that represent structured data:

- **`Attributes`** - Top-level container holding multiple attributes
- **`Attribute`** - A single key-value pair
- **`AttributeValue`** - The value part, which can be primitive, list, or map

All classes in the framework are **immutable**. They provide:
- `of()` factory methods to create instances
- `with()` methods to create modified copies with additional data

## Core Concepts

### Attributes

**Package**: `se.curity.identityserver.sdk.attribute.Attributes`

**Purpose**: Container for multiple `Attribute` objects, similar to a JSON object.

**Factory Methods**:
- `Attributes.of(Attribute...): Attributes` - Create from one or more attributes
- `Attributes.of(Collection<Attribute>): Attributes` - Create from attribute collection

**Modification Methods**:
- `with(Attribute): Attributes` - Returns new `Attributes` with the additional attribute
- `with(String, AttributeValue): Attributes` - Returns new `Attributes` with the additional key-value pair

**Access Methods**:
- `get(String): Attribute?` - Get attribute by name (using `[]` operator)
- `getAttributeValue(String): AttributeValue?` - Get value directly by name

### Attribute

**Package**: `se.curity.identityserver.sdk.attribute.Attribute`

**Purpose**: Represents a single key-value pair, similar to a JSON property.

**Factory Methods**:
- `Attribute.of(String, String): Attribute` - Create with string value
- `Attribute.of(String, Number): Attribute` - Create with numeric value
- `Attribute.of(String, Boolean): Attribute` - Create with boolean value
- `Attribute.of(String, AttributeValue): Attribute` - Create with complex value (list, map, nested attributes)

**Properties**:
- `name: String` - The attribute name
- `value: AttributeValue` - The attribute value

### AttributeValue

**Package**: `se.curity.identityserver.sdk.attribute.AttributeValue`

**Purpose**: Base interface for all attribute values.

**Implementations**:
- **Primitives**: String, Number, Boolean (automatically wrapped)
- **`ListAttributeValue`**: Ordered collection of values
- **`MapAttributeValue`**: Key-value map of values
- **Complex types**: `AccountAttributes`, `SubjectAttributes`, etc. (extend `Attributes`)

### ListAttributeValue

**Package**: `se.curity.identityserver.sdk.attribute.ListAttributeValue`

**Purpose**: Represents an ordered list of values (like JSON array).

**Factory Methods**:
- `ListAttributeValue.of(List<?>): ListAttributeValue` - Create from list

**Access Methods**:
- `asList(): List<AttributeValue>` - Get as list of attribute values

**Example**:
```kotlin
val listValue = ListAttributeValue.of(listOf("one", "two", "three"))
val attribute = Attribute.of("tags", listValue)
```

### MapAttributeValue

**Package**: `se.curity.identityserver.sdk.attribute.MapAttributeValue`

**Purpose**: Represents a key-value map (like JSON object).

**Factory Methods**:
- `MapAttributeValue.of(Map<String, ?>): MapAttributeValue` - Create from map
- `MapAttributeValue.of(Attributes): MapAttributeValue` - Create from Attributes

**Access Methods**:
- `asMap(): Map<String, AttributeValue>` - Get as map
- `get(String): AttributeValue?` - Get value by key

**Example**:
```kotlin
val mapValue = MapAttributeValue.of(
    mapOf(
        "string" to "a-string",
        "number" to 3
    )
)
val attribute = Attribute.of("metadata", mapValue)
```

## JSON Equivalence

The Attributes framework maps to JSON structures:

**Kotlin Code**:
```kotlin
val attributes = Attributes.of(
    Attribute.of("boolean", true),
    Attribute.of("string", "my-good-string"),
    Attribute.of("list", ListAttributeValue.of(listOf("one", "two", "three"))),
    Attribute.of("map", MapAttributeValue.of(
        mapOf(
            "string" to "a-string",
            "number" to 3
        )
    ))
)
```

**Equivalent JSON**:
```json
{
    "boolean": true,
    "string": "my-good-string",
    "list": ["one", "two", "three"],
    "map": {
        "string": "a-string",
        "number": 3
    }
}
```

## Specialized Attributes Classes

Several domain-specific classes extend `Attributes`:

### SubjectAttributes

**Purpose**: Represents a subject (user) with attributes.

**Critical Requirement**: Must always contain a `subject` attribute.

**Factory Methods**:
- `SubjectAttributes.of(String): SubjectAttributes` - Create with subject identifier
- `SubjectAttributes.of(Attribute): SubjectAttributes` - Create from single attribute
- `SubjectAttributes.of(Collection<Attribute>): SubjectAttributes` - Create from multiple attributes

**Modification Methods**:
- `with(Attribute): SubjectAttributes` - Returns new instance with additional attribute

**Properties**:
- `subject: String` - The subject identifier (convenience accessor)

**Example**:
```kotlin
// Simple subject
val subject = SubjectAttributes.of("teddie")

// Add attributes using chaining
val subjectWithEmail = subject
    .with(Attribute.of("email", "teddie@example.com"))
    .with(Attribute.of("role", "admin"))

// Or create with all attributes at once
val subjectFromList = SubjectAttributes.of(
    listOf(
        Attribute.of("subject", "teddie"),
        Attribute.of("email", "teddie@example.com")
    )
)
```

### AccountAttributes

**Purpose**: Represents user account information from the account manager.

**Factory Methods**:
- `AccountAttributes.of(String, String): AccountAttributes` - Create with ID and username
- `AccountAttributes.fromMap(Map<String, AttributeValue>): AccountAttributes` - Deserialize from map

**Properties**:
- `id: String` - Account ID
- `userName: String` - Username
- Plus methods like `getEmails()`, `getPhoneNumbers()`, etc.

**Example**:
```kotlin
// Create account attributes
val account = AccountAttributes.of("123", "teddie")

// Store as attribute value (wrap in MapAttributeValue)
val accountAttr = Attribute.of("account", MapAttributeValue.of(account))

// Add to subject
val subject = SubjectAttributes.of("teddie").with(accountAttr)

// Extract account later
val accountFromSubject = subject["account"]?.value?.let { value ->
    val mapValue = value as MapAttributeValue
    AccountAttributes.fromMap(mapValue.asMap())
}
```

### AuthenticationAttributes

**Purpose**: Container for authentication result attributes.

**Factory Methods**:
- `AuthenticationAttributes.of(SubjectAttributes): AuthenticationAttributes` - With subject only
- `AuthenticationAttributes.of(SubjectAttributes, ContextAttributes): AuthenticationAttributes` - With subject and context

**Example**:
```kotlin
val subject = SubjectAttributes.of("teddie")
    .with(Attribute.of("email", "teddie@example.com"))

val authAttrs = AuthenticationAttributes.of(subject)
```

### ContextAttributes

**Purpose**: Represents context information about the authentication event.

**Factory Methods**:
- `ContextAttributes.of(Attribute): ContextAttributes` - Create from single attribute
- `ContextAttributes.of(Collection<Attribute>): ContextAttributes` - Create from multiple attributes

**Example**:
```kotlin
val context = ContextAttributes.of(
    listOf(
        Attribute.of("authenticationMethod", "sms-otp"),
        Attribute.of("authenticationTime", System.currentTimeMillis())
    )
)
```

## Working with Immutability

Since all classes are immutable, use chaining or collection building:

### Chaining Pattern

```kotlin
val subject = SubjectAttributes.of(username)
    .with(Attribute.of("email", email))
    .with(Attribute.of("role", role))
    .with(Attribute.of("department", department))
```

### Collection Building Pattern

```kotlin
val attributes = mutableListOf(
    Attribute.of("subject", username)
)

if (email != null) {
    attributes.add(Attribute.of("email", email))
}

if (accountAttribute != null) {
    attributes.add(accountAttribute)
}

val subject = SubjectAttributes.of(attributes)
```

## Common Patterns

### Storing Complex Objects in Session

```kotlin
// Store account in session
val account = accountManager.getByUserName(username)
sessionManager.put(Attribute.of("account", MapAttributeValue.of(account)))

// Retrieve account from session
val accountAttr = sessionManager.get("account")
// Use directly in SubjectAttributes
val subject = SubjectAttributes.of(username).with(accountAttr)
```

### Reusing Session Attributes

```kotlin
// In first handler: store data
sessionManager.put(Attribute.of("otp", "123456"))
sessionManager.put(Attribute.of("account", MapAttributeValue.of(account)))

// In second handler: retrieve and reuse
val accountAttribute = sessionManager.get("account")
val subject = if (accountAttribute != null) {
    SubjectAttributes.of(
        listOf(
            Attribute.of("subject", username),
            accountAttribute  // Reuse directly - no unwrapping needed
        )
    )
} else {
    SubjectAttributes.of(username)
}
```

### Extracting Values

```kotlin
// Simple values
val email = subject["email"]?.value?.toString()

// Complex values - Lists
val tagsAttr = attributes["tags"]?.value as? ListAttributeValue
val tags = tagsAttr?.asList()?.map { it.toString() }

// Complex values - Maps
val metadataAttr = attributes["metadata"]?.value as? MapAttributeValue
val metadata = metadataAttr?.asMap()

// Complex values - AccountAttributes
val accountAttr = subject["account"]?.value as? MapAttributeValue
val account = accountAttr?.let { AccountAttributes.fromMap(it.asMap()) }
```

## Best Practices

1. **Use Factory Methods**: Always use `of()` to create instances
2. **Leverage Chaining**: Use `with()` methods for building up attributes
3. **Type Safety**: Use `MapAttributeValue.of()` when storing complex objects
4. **Session Efficiency**: Store complete `Attribute` objects in session to reuse them
5. **Null Safety**: Use Kotlin's null-safe operators when extracting values
6. **Immutability**: Remember that `with()` returns a new instance - assign the result
7. **Domain Classes**: Use specialized classes (`SubjectAttributes`, `AccountAttributes`) over raw `Attributes`

## Complete Example

```kotlin
import se.curity.identityserver.sdk.attribute.AccountAttributes
import se.curity.identityserver.sdk.attribute.Attribute
import se.curity.identityserver.sdk.attribute.MapAttributeValue
import se.curity.identityserver.sdk.attribute.SubjectAttributes

// Create account
val userName = "teddie"
val accountId = "123"
val account = AccountAttributes.of(accountId, userName)

// Wrap account in attribute
val accountAttribute = Attribute.of("account", MapAttributeValue.of(account))

// Create subject using chaining
val subjectChained = SubjectAttributes.of(userName).with(accountAttribute)

// Or create from list
val subjectFromList = SubjectAttributes.of(
    listOf(
        Attribute.of("subject", userName),
        accountAttribute
    )
)

// Both approaches produce the same result
assert(subjectChained.subject == userName)
assert(subjectFromList.subject == userName)

// Extract account from subject
val accountFromSubject = subjectChained["account"]?.value?.let { value ->
    val mapValue = value as MapAttributeValue
    AccountAttributes.fromMap(mapValue.asMap())
}

assert(accountFromSubject?.id == accountId)
assert(accountFromSubject?.userName == userName)
```
