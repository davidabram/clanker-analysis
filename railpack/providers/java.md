# Java Build Process

Complete documentation of how Railpack detects, builds, and runs Java
applications.

**Location**: `core/providers/java/`

## Detection

The Java provider is detected if **any** of these exist (`java.go:16-18`):

- `pom.xml`, `pom.atom`, `pom.clj`, `pom.groovy`, `pom.rb`, `pom.scala`,
  `pom.yaml`, `pom.yml` (Maven)
- `gradlew` (Gradle wrapper)

**Provider Priority**: Java is checked 3rd in the detection order (after Go).

## Build Tool Detection

- **Gradle**: If `gradlew` exists
- **Maven**: If `pom.xml` exists (or variants)

## JDK Version Resolution

**Location**: `java.go:105-114`, `jdk.go`

Priority order:

1. **`JAVA_VERSION` environment variable**
2. **`.sdkmanrc`** - Java version from SDK manager
3. **`pom.xml`** - Maven compiler source/target version
4. **mise.toml or .tool-versions**
5. **Default**: Determined by build tool

Maven example (`pom.xml`):

```xml
<properties>
  <maven.compiler.source>21</maven.compiler.source>
  <maven.compiler.target>21</maven.compiler.target>
</properties>
```

## Build Process

### Gradle Build

**Location**: `java.go:35-46`, `gradle.go`

**Process**:

```bash
# Make gradlew executable if needed
chmod +x gradlew

# Build
./gradlew clean build -x check -x test -Pproduction
```

**Flags**:

- `clean` - Remove previous build artifacts
- `build` - Build the project
- `-x check` - Skip checks
- `-x test` - Skip tests
- `-Pproduction` - Production profile

**Cache**: Gradle cache directory

**Output**: JAR files in `build/libs/` or `*/build/libs/`

### Maven Build

**Location**: `java.go:48-58`, `maven.go`

**Process**:

```bash
# Make mvnw executable if needed
chmod +x mvnw

# Build
./mvnw -DoutputFile=target/mvn-dependency-list.log -B -DskipTests clean dependency:list install -Pproduction
# Or: mvn <same flags> if mvnw doesn't exist
```

**Flags**:

- `-B` - Batch mode (non-interactive)
- `-DskipTests` - Skip tests
- `clean` - Remove previous build
- `dependency:list` - List dependencies
- `install` - Install to local repository
- `-Pproduction` - Production profile

**Cache**: Maven cache directory (`~/.m2/repository`)

**Output**: JAR files in `target/`

## Start Command

**Location**: `java.go:83-93`

### Gradle

```bash
java $JAVA_OPTS -jar <port-config> */build/libs/*jar | grep -v plain
```

**Port Config**: Extracted from `build.gradle` if Spring Boot detected

**Note**: Excludes `*-plain.jar` files

### Maven

```bash
java <port-config> $JAVA_OPTS -jar target/*jar
```

**Port Config**: Extracted from `pom.xml` if Spring Boot detected

## Framework Detection

### Spring Boot

**Location**: `java.go:112-116`

**Detected if**:

- `**/spring-boot*.jar` exists, OR
- `**/spring-boot*.class` exists, OR
- `**/org/springframework/boot/**` directory exists

**Metadata**: Sets `javaFramework=spring-boot`

### Port Configuration

**Location**: `maven.go`, `gradle.go`

For Spring Boot apps, attempts to read port from:

- Maven: `pom.xml` properties
- Gradle: `build.gradle` or `build.gradle.kts`

Injects port configuration into start command.

## Deploy

**Location**: `java.go:69-77`

**Deploy Includes**:

- JDK runtime (for running JAR)
- Built JAR files (`target/` or `build/libs/`)

**Output Directories**:

- Gradle: Current directory (`.`)
- Maven: `target/.`

## Complete Build Flow Example

### Spring Boot with Maven

```
Project structure:
  pom.xml
  src/
    main/
      java/
        com/app/
          Application.java
```

**Build Process**:

1. **Detection**: ✅ Has `pom.xml` → Java detected
2. **Build Tool**: Maven
3. **JDK Version**: Read from `pom.xml` or default
4. **Build**:
   ```bash
   mvn clean dependency:list install -DskipTests -B -Pproduction
   ```
5. **Start Command**:
   ```bash
   java $JAVA_OPTS -jar target/*jar
   ```

## Environment Variable Overrides

- `JAVA_VERSION` - Override JDK version
- `JAVA_OPTS` - JVM options (e.g., `-Xmx512m`)

## Best Practices

1. **Use wrapper scripts** - Include `mvnw` or `gradlew` in repository
2. **Specify Java version** - In `pom.xml` or `build.gradle`
3. **Production profile** - Configure for optimized builds
4. **Set `JAVA_OPTS`** - For memory limits and JVM tuning
5. **Single JAR** - Use fat JAR / uber JAR packaging

## Common Issues

### Multiple JARs Built

- Gradle creates both regular and `-plain.jar` files
- Railpack excludes `*-plain.jar` automatically
- For Maven, ensure single JAR output

### Wrong JDK Version

- Specify version explicitly in `pom.xml` or `build.gradle`
- Or use `JAVA_VERSION` environment variable

### OutOfMemoryError

- Set `JAVA_OPTS` with appropriate memory limits
- Example: `JAVA_OPTS=-Xmx1g -Xms512m`

## Files Reference

- `core/providers/java/java.go` - Main provider logic
- `core/providers/java/maven.go` - Maven-specific logic
- `core/providers/java/gradle.go` - Gradle-specific logic
- `core/providers/java/jdk.go` - JDK version resolution
- `examples/java-*` - Example Java projects
