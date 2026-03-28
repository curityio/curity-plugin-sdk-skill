# Build and Deployment

## 1. Repository Configuration

**IMPORTANT:** Always use **ONLY** `mavenCentral()` for repositories.

The Curity SDK and all required dependencies are available from Maven Central. Do not add any other repositories (including `repo.curity.io`).

```gradle
repositories {
    mavenCentral()
}
```

## 2. Build System
Curity plugins use **Gradle with Groovy DSL** for build configuration.

**Key build settings:**
- Java: Version 21
- Kotlin: Version 2.2.0 with JVM toolchain 21
- All SDK and server-provided dependencies: marked as `compileOnly`

Example `build.gradle`:
```gradle
plugins {
    id 'org.jetbrains.kotlin.jvm' version '2.2.0'
}

group = 'io.curity.identityserver.plugin'
version = '1.0.0'

repositories {
    mavenCentral()
}

dependencies {
    // Curity Identity Server SDK (provided at runtime)
    compileOnly 'se.curity.identityserver:identityserver.sdk:11.1.0'

    // SLF4J API (provided at runtime)
    compileOnly 'org.slf4j:slf4j-api:2.0.12'

    // Kotlin standard library (provided at runtime)
    compileOnly 'org.jetbrains.kotlin:kotlin-stdlib:2.2.0'

    // Jakarta Validation (provided at runtime)
    compileOnly 'jakarta.validation:jakarta.validation-api:3.0.0'
}

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}

kotlin {
    jvmToolchain(21)
}
```

## 3. Server-Provided Dependencies
The Curity Identity Server provides these dependencies at runtime. They must be declared as `compileOnly`:

- **SDK**: `se.curity.identityserver:identityserver.sdk:11.1.0`
- **SLF4J**: `org.slf4j:slf4j-api:2.0.12`
- **Kotlin stdlib**: `org.jetbrains.kotlin:kotlin-stdlib:2.2.0`
- **Jakarta Validation**: `jakarta.validation:jakarta.validation-api:3.0.0`

**Important:** Never include these as `implementation` or `runtimeOnly` dependencies, as they are already present in the server classpath.

## 4. Deployment Structure
Plugins are deployed to: `$IDSVR_HOME/usr/share/plugins/<plugin-name>/`

The deployment directory should contain:
- The plugin JAR file
- Any additional runtime dependencies (JARs not provided by the server)

All JARs must be in the **same directory** (no subdirectories).

## 5. Gradle Tasks

### `createDeployDir`
Prepares the plugin for deployment by creating a build directory with all necessary files.

```gradle
tasks.register('createDeployDir', Sync) {
    dependsOn jar

    destinationDir = file("$buildDir/deploy/${project.name}")

    // Copy plugin JAR
    from(jar)

    // Copy runtime dependencies (excluding provided dependencies)
    from(configurations.runtimeClasspath)
}
```

**Usage:** `./gradlew createDeployDir`

**Output:** `build/deploy/<plugin-name>/` containing the plugin JAR and any runtime dependencies

### `deployToLocal`
Automatically installs the plugin to a local Curity Identity Server instance.

```gradle
tasks.register('deployToLocal', Sync) {
    dependsOn createDeployDir

    doFirst {
        def idsvr_home = System.getenv('IDSVR_HOME')
        if (!idsvr_home) {
            throw new GradleException(
                "IDSVR_HOME environment variable is not set.\n" +
                "Please set it to your Curity Identity Server installation directory:\n" +
                "  export IDSVR_HOME=/path/to/idsvr"
            )
        }
    }

    def idsvr_home = System.getenv('IDSVR_HOME')

    from createDeployDir
    into file("$idsvr_home/usr/share/plugins/")
}
```

**Prerequisites:**
- `IDSVR_HOME` environment variable must be set
- Example: `export IDSVR_HOME=/opt/idsvr`

**Usage:** `./gradlew deployToLocal`

## 6. Development Workflow

1. **Initialize Gradle wrapper (if not present):** `gradle wrapper`
2. **Build the plugin:** `./gradlew build`
3. **Prepare for deployment:** `./gradlew createDeployDir`
4. **Install to local server:** `./gradlew deployToLocal` (requires `IDSVR_HOME`)
5. **Restart the Curity Identity Server** to load the plugin

**Note:** Always install the Gradle wrapper before building. If the wrapper is not present in your project, run `gradle wrapper` to generate it. This ensures consistent builds across different environments.

## 7. Best Practices

- Always use `compileOnly` for server-provided dependencies
- Keep the plugin JAR small by not bundling server-provided libraries
- Use semantic versioning for plugin versions
- Test plugins locally before finishing your task
- Document any custom runtime dependencies your plugin requires
