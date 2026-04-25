---
name: jvm-spring-scaffold
description: "Scaffolds a new Kotlin or Java Spring Boot project. Use when the user wants to create, bootstrap, initialize, or start a Spring Boot project, microservice, REST API, or JVM backend service — including vague requests like 'set up a Spring app', 'create a Spring project', or 'start a new Kotlin/Java service'. Also the default for any generic JVM backend request where no framework is specified. Do NOT use if the user mentions Dropwizard, Quarkus, Ktor, Helidon, or Micronaut."
---

# Spring Boot Scaffold

This skill creates a working Spring Boot project using Spring Initializr `https://start.spring.io`, unzips it, and verifies the build compiles cleanly.

## Step 1 — Determine the placement mode

Decide where the project files should land **before** doing anything else. Ask if the user hasn't been clear.

| Signal | Mode | Description |
|---|---|---|
| "in the current folder", "here", "in this directory", "as root project", or no placement cue | **root** (default) | files go directly into the current working directory |
| "as a module", "in a subfolder", "as a subproject" | **subproject** | files go into a new subfolder named after the artifactId |

Before proceeding, check if the target directory already contains a project (look for `build.gradle.kts`, `build.gradle`, `pom.xml`, `gradlew`, or `mvnw`). If found, warn the user and ask for confirmation before continuing.

## Step 2 — Fetch start.spring.io metadata

WebFetch `https://start.spring.io/metadata/client` and extract:
- **Latest stable Spring Boot version**: first entry in `bootVersion.values` whose `id` does not contain `SNAPSHOT`, `RC`, or `M` (milestone). Normalize version (Remove `.RELEASE`) `4.0.6.RELEASE` becomes `4.0.6`
- **Supported Java versions**: full list of `id` values from `javaVersion.values`

If the fetch fails, proceed — Step 3 handles the fallback.

## Step 3 — Determine the tech stack

Before generating anything, infer the right choices from the task context. Ask if the user hasn't been clear.

### Language
| Signal | Choice |
|---|---|
| User says "Kotlin" or doesn't specify (default) | `kotlin` |
| User says "Java" explicitly | `java` |

### Build tool
| Signal | Choice |
|---|---|
| User says "Gradle" or doesn't specify (default) | `gradle-project-kotlin` |
| User says "Maven" explicitly | `maven-project` |
| User says "Groovy DSL" explicitly | `gradle-project` |

### Java version

| Metadata available | User specified | Action |
|---|---|---|
| yes | no | for `java` fallback to `25`; for `kotlin` fallback to `21` |
| yes | yes, in `javaVersion.values` | use user's value |
| yes | yes, **not in** `javaVersion.values` | warn the user and confirm before proceeding |
| no | no | for `java` fallback to `25`; for `kotlin` fallback to `21` |
| no | yes | trust user; note at end that version was not validated |

### Spring Boot version

| Metadata available | User specified | Action |
|---|---|---|
| yes | no | use the latest stable version extracted in Step 2 |
| yes | yes | use user's value |
| no | no | fall back to `4.0.6` |
| no | yes | trust user; note at end that version was not validated |

### Dependencies
Map the task description to starter IDs. Common mappings:

| Task / technology mentioned | Dependency IDs |
|---|---|
| REST API, web endpoints | `web` |
| Reactive / WebFlux / non-blocking | `webflux` |
| Validation (JSR-303) | `validation` |
| JPA / relational DB (ORM) | `data-jpa` |
| JDBC (no ORM) | `data-jdbc` |
| PostgreSQL | `data-jpa,postgresql` |
| MySQL | `data-jpa,mysql` |
| H2 in-memory database | `data-jpa,h2` |
| Flyway migrations | `flyway` |
| Liquibase migrations | `liquibase` |
| MongoDB | `data-mongodb` |
| Redis cache/session | `data-redis` |
| Kafka | `kafka` |
| RabbitMQ | `amqp` |
| Security / JWT / auth | `security` |
| OAuth2 resource server | `oauth2-resource-server` |
| OAuth2 client | `oauth2-client` |
| GraphQL | `graphql` |
| gRPC server | `spring-grpc-server` |
| gRPC client | `spring-grpc-client` |
| API docs / OpenAPI / Swagger | `springdoc-openapi` |
| Metrics / Prometheus scraping | `prometheus` |
| Observability / health checks | `actuator` |
| Testcontainers | `testcontainers` |
| Hot reload during development | `devtools` |
| Lombok | `lombok` |

For Kotlin projects, **do not** add `lombok` — Kotlin data classes replace it.

## Step 4 — Build the curl command and download

Construct the parameters from Step 3, then download `starter.zip`.
- `baseDir`: depends on placement mode:
  - **Subproject mode**: include `baseDir` — the zip will unpack into a named subfolder.
  - **Root mode**: omit `baseDir` entirely — the zip extracts files directly at the root level, no wrapping folder.
- `groupId`: derive from task context (company name), use reverse-domain notation (e.g. `com.example`).
- `artifactId`: derive from task context (service name). Keep artifactId lowercase-hyphenated (`demo-service`).
- `packageName`: derived from groupId + artifactId, dots only, no hyphens. Strip common redundant suffixes (`-service`, `-application`, `-app`) from the artifactId first, then collapse remaining hyphens — `demo-service` → `com.example.demo`, `user-auth-service` → `com.example.userauth`, `payment-app` → `com.example.payment`.

```bash
TYPE="gradle-project-kotlin"            # or maven-project, gradle-project
LANGUAGE="kotlin"                       # or java
BOOT_VERSION="4.0.6"
JAVA_VERSION="21"
GROUP_ID="com.example"
ARTIFACT_ID="demo-service"
BASE_DIR="demo-service"                 # subproject mode only; omit variable and -d line below for root mode
PACKAGE_NAME="com.example.demo"
DEPENDENCIES="web,actuator,validation"

curl -s "https://start.spring.io/starter.zip" \
  -d type="$TYPE" \
  -d language="$LANGUAGE" \
  -d bootVersion="$BOOT_VERSION" \
  -d baseDir="$BASE_DIR" \
  -d groupId="$GROUP_ID" \
  -d artifactId="$ARTIFACT_ID" \
  -d name="$ARTIFACT_ID" \
  -d packageName="$PACKAGE_NAME" \
  -d dependencies="$DEPENDENCIES" \
  -d packaging=jar \
  -d javaVersion="$JAVA_VERSION" \
  -o "${ARTIFACT_ID}.zip"
```

Verify the download succeeded using shell: `file "${ARTIFACT_ID}.zip"` (the file should be a zip, not an error page).
If `file` reports it as HTML or ASCII, print the file content — the API returned an error. Fix the parameters and retry.

## Step 5 — Unzip

Both modes use the same commands — the zip structure already matches the target layout.

```bash
unzip -q "${ARTIFACT_ID}.zip"
rm "${ARTIFACT_ID}.zip"
ls -la .
```

## Step 6 — Build and verify

**Subproject mode** — use a subshell so the working directory is not changed for subsequent commands:
- Gradle: `(cd "$BASE_DIR" && chmod +x gradlew && ./gradlew build --no-daemon -x test 2>&1 | tail -30)`
- Maven: `(cd "$BASE_DIR" && chmod +x mvnw && ./mvnw verify -DskipTests --no-transfer-progress 2>&1 | tail -30)`

**Root mode** — run directly from the current directory:
- Gradle: `chmod +x gradlew && ./gradlew build --no-daemon -x test 2>&1 | tail -30`
- Maven: `chmod +x mvnw && ./mvnw verify -DskipTests --no-transfer-progress 2>&1 | tail -30`

A successful build ends with `BUILD SUCCESSFUL` (Gradle) or `BUILD SUCCESS` (Maven). If the build fails, diagnose from the output

## Step 7 — Tell the user what was created

After a clean build, report in short form:
- Project location (absolute path)
- Language, Spring Boot version and build tool used
- Dependencies included
- How to run: `./gradlew bootRun` or `./mvnw spring-boot:run`
- If start.spring.io metadata was unavailable and the user provided versions, note that those were used as-is without validation against the current supported versions list.
