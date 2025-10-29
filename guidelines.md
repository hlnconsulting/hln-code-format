# HLN + Junie Spring Boot Guidelines (Merged)

This document merges Junie’s default Spring Boot guidelines with HLN’s house styleguide. Where guidance conflicts, the HLN
styleguide takes precedence. The goal is a single, practical reference for new services and existing codebases.

Contents

- Project layout and build
- Cloud Build (GCP) setup
- .gitignore baseline
- Spring Boot standards (HLN-preferred)
- Configuration and properties
- REST standards
- Database standards
- OO and coding standards
- Testing
- Logging
- Actuator
- Quick checklist for this repository

---

## Project layout and build

Use standard Maven layout. Prefer packaging = jar unless there is a specific need for war.

Example pom.xml (simplified skeleton):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://maven.apache.org/POM/4.0.0"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>gov.nyc.dohmh</groupId>
    <artifactId>report-service</artifactId>
    <version>1.0.0</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.X</version>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <java.version>25</java.version>
        <!-- additional properties here (including any overwrites of parent versioning) -->
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>XXXX.XX.XX</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!-- additional dependency management here -->
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- additional Spring starters here -->

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
        </dependency>

        <!-- additional Spring Cloud starters here -->

        <dependency>
            <groupId>org.apache.httpcomponents.client5</groupId>
            <artifactId>httpclient5</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

        <!-- additional libraries here that rely on the parent pom versioning -->

        <!-- additional libraries with versioning here -->

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>${project.parent.version}</version>
            </plugin>
            <plugin>
                <groupId>io.github.git-commit-id</groupId>
                <artifactId>git-commit-id-maven-plugin</artifactId>
                <version>${git-commit-id-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>get-the-git-infos</id>
                        <goals>
                            <goal>revision</goal>
                        </goals>
                        <phase>initialize</phase>
                    </execution>
                </executions>
                <configuration>
                    <generateGitPropertiesFile>true</generateGitPropertiesFile>
                    <commitIdGenerationMode>full</commitIdGenerationMode>
                </configuration>
            </plugin>
            <!-- additional build plugins here -->
        </plugins>
    </build>
</project>
```

Notes

- Favor constructor injection; avoid field/setter injection.
- Rely on Spring auto-configuration where possible; override defaults only when necessary.

---

## Cloud Build (GCP) setup

Place the Cloud Build configuration at: /.gcp.config/cloudbuild.yaml

Template:

```yaml
substitutions:
  _IMAGE_NAME: XXXXXXXX

steps:
  - name: buildpacksio/pack:latest
    id: build
    entrypoint: pack
    args:
      - "build"
      - "us-central1-docker.pkg.dev/${PROJECT_ID}/containers/${_IMAGE_NAME}:${BRANCH_NAME}"
      - "--tag"
      - "us-central1-docker.pkg.dev/${PROJECT_ID}/containers/${_IMAGE_NAME}:${SHORT_SHA}"
      - "--builder"
      - "paketobuildpacks/builder-jammy-buildpackless-base"
      - "--buildpack"
      - "docker.io/paketobuildpacks/ca-certificates"
      - "--buildpack"
      - "docker.io/paketobuildpacks/adoptium"
      - "--buildpack"
      - "docker.io/paketobuildpacks/java"
      - "--env"
      - "BP_JVM_VERSION=21.*"
      - "--env"
      - "BP_IMAGE_LABELS=git.commit.id=$COMMIT_SHA"
      - "--publish"
```

Notes

- Use buildpackless-full instead of buildpackless-base if the app relies on fonts.
- Add env vars if you want unit tests to run during build.

---

## .gitignore baseline

Minimum entries:

```
target/
*.class

.idea/
*.iml

.vscode/
.DS_Store
```

---

## Spring Boot standards (HLN-preferred)

- Constructor injection for mandatory dependencies. Declare them final; Spring will use the sole constructor automatically.
- Organize configuration with typed properties (@ConfigurationProperties) and validation annotations. Prefer env vars over profiles
  for per-environment values.
- Define clear transaction boundaries using @Transactional; annotate read-only queries with readOnly = true.
- Records are preferred over classes where suitable.
- Favor property/annotation configuration over programmatic configuration.
- Avoid redundant annotation attributes (e.g., omit consumes/produces when defaults are JSON; avoid naming
  @PathVariable/@RequestParam if variable names match).
- Naming: choose variable/method names to eliminate the need for override names in annotations.
- Error handling: Prefer throwing ResponseStatusException for request failures. Use @ControllerAdvice only for cross-cutting
  concerns, external exception mapping, or consistent formatting (e.g., ProblemDetails). Avoid custom exception hierarchies unless
  valuable.
- Response handling: Generally avoid ResponseEntity unless you need to set headers (e.g., Location) or vary status code
  programmatically. Otherwise let Spring map return values and use @ResponseStatus where needed.

---

## Configuration and properties

Use YAML named application.yml. Kebab-case keys. Global config first, then environment-specific sections.

Example structure:

```yaml
spring:
  application:
    name: XXXXX

# security config here (using issuer-uri) if required

management:
  server:
    port: 8099
  endpoint:
    restart:
      access: unrestricted
  endpoints:
    web:
      base-path: /manage
      exposure:
        include: info,health,restart,refresh,prometheus,loggers,httptrace,caches,scheduledtasks

# keycloak config here if required...

# global app config here...
---

spring:
  config:
    activate:
      on-profile: kubernetes
  cloud:
    config:
      enabled: false
---

spring:
  config:
    activate:
      on-profile: default
  cloud:
    config:
      enabled: false

management:
  server:
    port: #{null}

# local keycloak config here if necessary...

# local app config here...
```

Notes

- Prefer environment variables for deployment-time values.
- Keep transactions as small as possible; avoid the Open Session in View antipattern.

---

## REST standards

- Accept and respond with JSON by default. Require application/json for requests and set Content-Type: application/json on
  responses. Honor Accept headers via Spring content negotiation when needed. Use appropriate media types for file upload/download.
- Use nouns (not verbs) in endpoint paths. Collections should be plural nouns. Use kebab-case for path segments (e.g.,
  /api/v1/custom-widgets).
- Version endpoints: /api/v{version}/resources (e.g., /api/v1/orders). Avoid unversioned public APIs.
- Nest resources logically for parent/child relationships (e.g., /articles/{articleId}/comments) but avoid deep nesting beyond 2–3
  levels; include links (URIs) to related resources instead of over-nesting.
- Query parameters should be camelCase for ergonomics (e.g., /custom-widgets?fooBar=true). JSON properties should be consistently
  camelCase or snake_case within a service; choose one and stick with it. JSON payloads must be top-level objects to keep
  extensible.
- POST vs PUT: Use POST when the saved object differs from the payload (server-generated fields, transformations) or when returning
  a body. Use PUT for idempotent full replacements that do not change the representation and typically return no content. PATCH may
  be used for partial updates when appropriate.
- Filtering, sorting, and pagination: Support field-based filtering via query params, sorting via sort=+field,-other conventions,
  and pagination via page/size (or cursor-based where needed). Set sensible defaults and max limits to protect the service.
- Error handling and status codes: Return standard HTTP codes (e.g., 400, 401, 403, 404, 409, 422, 500, 502, 503). Use
  ProblemDetails for consistent error bodies and avoid leaking sensitive info. Prefer throwing ResponseStatusException in
  controllers; use @ControllerAdvice for cross-cutting mapping.
- Caching: Use appropriate caching headers (Cache-Control, ETag/If-None-Match) and server-side caching where it safely improves
  performance. Define cache invalidation/TTL strategies.
- Security: Enforce TLS, authentication, and authorization (principle of least privilege). Validate inputs, avoid overexposing
  data (only return what the caller is allowed to see), and consider rate limiting for public APIs.
- Idempotency: Ensure PUT and DELETE are idempotent. For retryable POST operations (e.g., payments), consider supporting an
  Idempotency-Key header.

---

## Database standards

- Prefer MongoDB when possible (except where CIR-accessed data dictates otherwise). Use Spring Boot to auto-create collections and
  indexes; align naming to Java conventions. Use TTL indexes for auto-cleanup where appropriate.
- For SQL:
    - Tables and columns: snake_case, singular nouns.
    - Primary key: id.
    - Foreign keys: table_column and always indexed.

---

## OO and coding standards

- Clear package and class naming; code organized by package -> class -> method.
- Use restrictive Java modifiers; class member variables should be private.
- Do not create interfaces and impls by default; add them only when there’s a reason.
- HLN standardized formatting should be used. Enable format-on-commit or save actions.
- Enable JetBrains inspections; include LocalCanBeFinal and IncorrectFormatting.
- Class layout order: inner classes/enums/records (prefer static), static fields, static methods, instance fields, instance methods.
- Favor Lombok where appropriate.
- Use modern Java features (streams, lambdas, try-with-resources, pattern matching for switch, java.nio.Path, etc.).
- Avoid excessive temporary variables; compose method calls when it remains readable.
- Avoid repeating method names in log messages without need.
- Single-line statements may omit braces; use judgment for readability and diffs.
- Use Optional primarily for return types, not parameters (Spring autowiring is a notable exception). Prefer map/orElse/etc. over
  ifPresent/get.
- Prefer LocalDate/LocalDateTime unless time zones are required.
- Naming: packages lower case (concatenated or snake case); classes/enums/records PascalCase; variables camelCase; enum constants
  UPPER_SNAKE_CASE. Immutable static finals may use UPPER_SNAKE_CASE.
- Prefer switch over long if-else chains.
- Omit else when the if branch returns/throws; place terminal branches first.
- Prefer declaring variables by interface types (e.g., List<Foo> list = new ArrayList<>()). Use var where it aids readability.

---

## Testing

- Use Testcontainers for integration tests to mirror production dependencies.
- Start Spring Boot tests on a random port:

```text
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
```

---

## Logging

- Use SLF4J with a proper backend; never use System.out.println for application logs.
- Protect sensitive data; never log credentials or personal information.
- Guard expensive log calls. For example:

```text
if (logger.isDebugEnabled()) {
    logger.debug("Detailed state: {}", computeExpensiveDetails());
}

logger.atDebug()
    .setMessage("Detailed state: {}")
    .addArgument(() -> computeExpensiveDetails())
    .log();
```

## JetBrains Inspectopedia (Inspections)

- We use JetBrains Inspectopedia as the baseline for static code analysis in IntelliJ IDEA. Configure at: Settings/Preferences >
  Editor > Inspections. Run Analyze > Inspect Code… on Whole Project before PRs.
- Recommended inspections to enable and treat as at least Warning (Error for critical ones):
    - Dependency injection:
        - Spring: Field injection — discourage @Autowired on fields; prefer constructor injection.
    - Logging:
        - Use of System.out/err or printStackTrace — use SLF4J logger instead.
    - Code quality:
        - Local variable or parameter can be final (LocalCanBeFinal).
        - Redundant modifiers.
        - Redundant nullability annotations and redundant suppression.
        - Unused declaration / unused import.
    - API and design:
        - Optional used as field or parameter — prefer Optional only for return types.
        - Class can be record — prefer records where suitable.
    - Exceptions:
        - Empty catch block.
        - Overly broad catches or caught exception immediately rethrown.
- Formatting:
    - Incorrect formatting — use HLN standardized formatting; enable format on commit or save actions.
- Suppression policy: If you must suppress an inspection, add a brief justification comment on the line or in the PR.
- Team tip: Export your inspection profile (Manage… > Export) and share internally; do not commit .idea to VCS (already ignored).

---

## Actuator

- Base path should be /manage.

---

## Quick checklist for this repository

- Maven project layout with Spring Boot parent and Java 21. ✓
- application.yml present and using YAML with a default-profile section. ✓
- Actuator base-path /manage configured. ✓
- .gitignore includes target, classes, IDE files, and OS files. (Updated to include .vscode/ and .DS_Store.) ✓
- Cloud Build file at /.gcp.config/cloudbuild.yaml with _IMAGE_NAME placeholder. Added in this change; set the correct value per
  service when building. ✓

If any guideline appears to conflict with existing code, prefer the HLN-preferred rules above and open a refactor PR if the change
is non-trivial.
