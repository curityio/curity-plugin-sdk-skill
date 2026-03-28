# Event Listener Plugin – Agent Reference

## 1. When to Use

Use an **Event Listener Plugin** when you need to:

- **Audit logging** — record authentication events, token issuance, account changes
- **External notifications** — send events to SIEM systems, webhooks, or message queues
- **Analytics** — track login patterns, failure rates, or usage metrics
- **Side effects** — trigger downstream actions when specific events occur

Event listeners are **passive observers** — they cannot modify the flow or block operations.

## 2. Key Interfaces

| Interface | Package | Purpose |
|-----------|---------|---------|
| `EventListenerPluginDescriptor<C>` | `se.curity.identityserver.sdk.plugin.descriptor` | Plugin entry point |
| `EventListenerCollection` | `se.curity.identityserver.sdk.event` | Container for listener instances |
| `EventListener<T extends Event>` | `se.curity.identityserver.sdk.event` | Handles events of type T |
| `Event` | `se.curity.identityserver.sdk.data.events` | Base type for all events |

## 3. Architecture

```
EventListenerPluginDescriptor
  └── getEventListenerCollection() → EventListenerCollection
        └── getListeners() → Set<EventListener<?>>
              └── EventListener<T>
                    ├── getEventType() → Class<T>
                    └── handle(T event)
```

The server calls `handle()` whenever an event matching `getEventType()` occurs. If `getEventType()` returns `Event.class`, the listener receives **all** events.

## 4. Lifecycle and Scope

Event listeners have **`@ConfigurationScope`** access only. This means:

- They are created when configuration loads (not per-request)
- They receive only services marked with `@ConfigurationScope`
- They must be **thread-safe** — multiple events may be handled concurrently
- They do **not** have access to per-request services (e.g., request objects)

## 5. Event Types

The base `Event` interface provides:
- `asMap(): Map<String, Object>` — all event data as a map

Specific event subtypes exist in `se.curity.identityserver.sdk.data.events` for different categories (OAuth events, authentication events, account events, etc.). Use a specific subtype in `getEventType()` to filter which events your listener receives.

## 6. Code Skeleton

### Descriptor (Java)

```java
package com.example.plugin;

import se.curity.identityserver.sdk.event.EventListener;
import se.curity.identityserver.sdk.event.EventListenerCollection;
import se.curity.identityserver.sdk.plugin.descriptor.EventListenerPluginDescriptor;

import java.util.Collections;
import java.util.Set;

public final class MyEventListenerDescriptor
        implements EventListenerPluginDescriptor<MyEventListenerConfig>
{
    @Override
    public Class<? extends EventListenerCollection> getEventListenerCollection()
    {
        return MyListenerCollection.class;
    }

    @Override
    public String getPluginImplementationType()
    {
        return "my-event-listener";
    }

    @Override
    public Class<? extends MyEventListenerConfig> getConfigurationType()
    {
        return MyEventListenerConfig.class;
    }

    public static final class MyListenerCollection implements EventListenerCollection
    {
        private final Set<EventListener<?>> _listeners;

        public MyListenerCollection(MyEventListenerConfig configuration)
        {
            _listeners = Collections.singleton(new MyEventListener(configuration));
        }

        @Override
        public Set<? extends EventListener<?>> getListeners()
        {
            return Collections.unmodifiableSet(_listeners);
        }
    }
}
```

### Descriptor (Kotlin)

```kotlin
package com.example.plugin

import se.curity.identityserver.sdk.event.EventListener
import se.curity.identityserver.sdk.event.EventListenerCollection
import se.curity.identityserver.sdk.plugin.descriptor.EventListenerPluginDescriptor

class MyEventListenerDescriptor : EventListenerPluginDescriptor<MyEventListenerConfig> {

    override fun getEventListenerCollection() = MyListenerCollection::class.java
    override fun getPluginImplementationType() = "my-event-listener"
    override fun getConfigurationType() = MyEventListenerConfig::class.java

    class MyListenerCollection(configuration: MyEventListenerConfig) : EventListenerCollection {
        private val listeners: Set<EventListener<*>> = setOf(MyEventListener(configuration))
        override fun getListeners(): Set<EventListener<*>> = listeners
    }
}
```

### Configuration

```java
package com.example.plugin;

import se.curity.identityserver.sdk.config.Configuration;

public interface MyEventListenerConfig extends Configuration
{
    // Add services as needed. Only @ConfigurationScope services are available.
    // Example: HttpClient getHttpClient();
}
```

### Event Listener — Listen to All Events (Java)

```java
package com.example.plugin;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import se.curity.identityserver.sdk.data.events.Event;
import se.curity.identityserver.sdk.event.EventListener;

public final class MyEventListener implements EventListener<Event>
{
    private static final Logger _logger = LoggerFactory.getLogger(MyEventListener.class);
    private final MyEventListenerConfig _configuration;

    public MyEventListener(MyEventListenerConfig configuration)
    {
        _configuration = configuration;
    }

    @Override
    public Class<Event> getEventType()
    {
        return Event.class;  // Receives ALL events
    }

    @Override
    public void handle(Event event)
    {
        _logger.info("Event received: {}", event.asMap());
    }
}
```

### Event Listener — Listen to Specific Event Type

To listen only to a specific event type, change the generic parameter and `getEventType()`:

```java
import se.curity.identityserver.sdk.data.events.IssuedAccessTokenOAuthEvent;

public final class AccessTokenListener implements EventListener<IssuedAccessTokenOAuthEvent>
{
    @Override
    public Class<IssuedAccessTokenOAuthEvent> getEventType()
    {
        return IssuedAccessTokenOAuthEvent.class;
    }

    @Override
    public void handle(IssuedAccessTokenOAuthEvent event)
    {
        // Only called for access token issuance events
        _logger.info("Access token issued: {}", event.asMap());
    }
}
```

### Service Descriptor File

`src/main/resources/META-INF/services/se.curity.identityserver.sdk.plugin.descriptor.EventListenerPluginDescriptor`:
```
com.example.plugin.MyEventListenerDescriptor
```

## 7. Multiple Listeners in One Plugin

A single plugin can provide multiple listeners for different event types:

```java
public static final class MyListenerCollection implements EventListenerCollection
{
    private final Set<EventListener<?>> _listeners;

    public MyListenerCollection(MyEventListenerConfig configuration)
    {
        _listeners = Set.of(
                new AuthenticationEventListener(configuration),
                new TokenEventListener(configuration),
                new AccountEventListener(configuration)
        );
    }

    @Override
    public Set<? extends EventListener<?>> getListeners()
    {
        return _listeners;
    }
}
```

## 8. Gradle Build

```groovy
plugins {
    id 'java'
}

group = 'com.example'
version = '1.0.0-SNAPSHOT'

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

dependencies {
    compileOnly 'se.curity.identityserver:identityserver.sdk:11.1.0'
    compileOnly 'org.slf4j:slf4j-api:2.0.12'
}
```

## 9. Testing

```groovy
import spock.lang.Specification
import se.curity.identityserver.sdk.data.events.Event

class MyEventListenerSpec extends Specification {

    def "handle logs event data"() {
        given:
        def config = Mock(MyEventListenerConfig)
        def listener = new MyEventListener(config)
        def event = Mock(Event) {
            asMap() >> [type: "auth_success", subject: "user123"]
        }

        when:
        listener.handle(event)

        then:
        noExceptionThrown()
    }

    def "getEventType returns Event class"() {
        given:
        def listener = new MyEventListener(Mock(MyEventListenerConfig))

        expect:
        listener.getEventType() == Event
    }
}
```

## 10. Key Differences from Other Plugin Types

| Aspect | Event Listener | Authenticator / Action |
|--------|---------------|----------------------|
| Scope | `@ConfigurationScope` | Per-request scope |
| Lifecycle | Created on config load | Created per request |
| Threading | Must be thread-safe | Single-threaded per request |
| Services | Only `@ConfigurationScope` services | All services |
| User interaction | None (passive observer) | Can render views, redirect |
| Flow control | Cannot modify flow | Can allow/deny/redirect |
