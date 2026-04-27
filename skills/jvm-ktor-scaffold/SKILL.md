---
name: jvm-ktor-scaffold
description: "Scaffolds a new Kotlin Ktor project. Use when the user wants to create, bootstrap, initialize, or start a Ktor project, microservice, REST API, or Kotlin backend service — including vague requests like 'set up a Ktor app', 'create a Ktor project', 'start a new Ktor service', or 'build a Kotlin web service with Ktor'. Do NOT use if the user mentions Spring Boot, Dropwizard, Quarkus, Helidon, or Micronaut."
---

# Ktor Scaffold

This skill creates a working Ktor project using the `start.ktor.io` generator API, unzips it, and verifies the build compiles cleanly.

## Step 1 — Determine the placement mode

Decide where the project files should land **before** doing anything else. Ask if the user hasn't been clear.

| Signal | Mode | Description |
|---|---|---|
| "in the current folder", "here", "in this directory", "as root project", or no placement cue | **root** (default) | files go directly into the current working directory |
| "as a module", "in a subfolder", "as a subproject" | **subproject** | files go into a new subfolder named after the project |

Before proceeding, check if the target directory already contains a project (look for `build.gradle.kts`, `build.gradle`, `pom.xml`, `gradlew`, or `mvnw`). If found, warn the user and ask for confirmation before continuing.

## Step 2 — Determine artifact identifiers

- **artifactId** — project / service name, lowercase-hyphenated (`demo-service`).
  Derive from task context. If not determinable, ask the user.
- **groupId** — company domain in reverse notation (`com.example`).
  Derive only from what the user explicitly stated in the prompt (e.g. "use com.acme", "for Acme Corp", "domain is acme.com"). Do **not** infer from email addresses, session info, git config, or any other implicit source. If not explicitly provided, ask the user.

`packageName` = `groupId` (reversed domain only, nothing appended).

## Step 3 — Fetch start.ktor.io metadata

Make two requests:

**Settings** — `GET https://start.ktor.io/project/settings`

Extract:
- **Latest Ktor version**: `ktor_version.default_id`
- **Latest Kotlin version**: `kotlin_version.default_id`
- **Available engines**: `engine.options[].id` (e.g. `NETTY`, `JETTY`, `CIO`, `TOMCAT`)
- **Available build systems**: `build_system.options[].id`
- **Configuration options**: `configuration_in.options[].id`

**Features** — `GET https://start.ktor.io/features/{ktorVersion}`

Each item has `xmlId` (used as the plugin ID in the generate request), `name`, `group`, and `requiredFeatures`.

If the fetches fail, proceed — Step 4 handles the fallback.

## Step 4 — Determine the tech stack

Before generating anything, infer the right choices from the task context. Ask if the user hasn't been clear.

### Build system

| Signal | Choice |
|---|---|
| User says "Gradle" or doesn't specify (default) | `GRADLE_KTS` |
| User says "Maven" explicitly | `MAVEN` |
| User says "Amper" explicitly | `AMPER` |

### Engine

| Signal | Choice |
|---|---|
| No preference specified (default) | `NETTY` |
| User says "Jetty" | `JETTY` |
| User says "CIO" or "coroutine" | `CIO` |
| User says "Tomcat" | `TOMCAT` |

### Configuration format

| Signal | Choice |
|---|---|
| No preference specified (default) | `YAML` |
| User says "HOCON" or "application.conf" | `HOCON` |
| User says "code" or "programmatic" | `CODE` |

### Ktor version

| Metadata available | User specified | Action |
|---|---|---|
| yes | no | use `ktor_version.default_id` from settings |
| yes | yes | use user's value |
| no | no | fall back to `3.4.2` |
| no | yes | trust user; note at end that version was not validated |

### Kotlin version

| Metadata available | User specified | Action |
|---|---|---|
| yes | no | use `kotlin_version.default_id` from settings |
| no | no | fall back to `2.3.20` |

### Plugins (features)

Map the task description to plugin `xmlId` values. Include required dependencies automatically (check `requiredFeatures`).

After selecting plugins from the table, verify each `xmlId` exists verbatim in the features JSON fetched in Step 3. The table is a guide for intent mapping — the fetched JSON is the source of truth for spelling. If a selected `xmlId` is not found in the fetched list, flag it to the user before proceeding.

| Task / technology mentioned | Plugin xmlId(s) |
|---|---|
| REST API / HTTP endpoints | `io.ktor/server-content-negotiation`, `io.ktor/server-kotlinx-serialization` |
| Routing only | _(routing is always included by default — no extra plugin needed)_ |
| JSON serialization | `io.ktor/server-content-negotiation`, `io.ktor/server-kotlinx-serialization` |
| Jackson serialization | `io.ktor/server-content-negotiation`, `io.ktor/server-jackson` |
| GSON serialization | `io.ktor/server-content-negotiation`, `io.ktor/server-gson` |
| Authentication (general) | `io.ktor/server-auth` |
| OAuth2 resource server — validate incoming Bearer/JWT tokens issued by an external auth server (Keycloak, Auth0, etc.) | `io.ktor/server-auth`, `io.ktor/server-auth-jwt` |
| OAuth2 client — redirect users to an external provider for login (Google, GitHub, Keycloak authorization code flow) | `io.ktor/server-auth`, `io.ktor/server-auth-oauth` |
| Basic auth | `io.ktor/server-auth`, `io.ktor/server-auth-basic` |
| Sessions / cookies | `io.ktor/server-sessions` |
| CORS | `io.ktor/server-cors` |
| CSRF protection | `io.ktor/server-csrf` |
| HTTPS redirect / HSTS | `io.ktor/server-https-redirect`, `io.ktor/server-hsts` |
| Compression / gzip | `io.ktor/server-compression` |
| Caching headers | `io.ktor/server-caching-headers` |
| Static files / content | `io.ktor/server-static-content` |
| WebSockets | `io.ktor/server-websockets` |
| Server-Sent Events | `io.ktor/server-sse` |
| Request validation | `io.ktor/server-request-validation` |
| Status pages / error handling | `io.ktor/server-status-pages` |
| Request logging / call log | `io.ktor/server-call-logging` |
| Call ID / tracing | `io.ktor/server-callid`, `io.ktor/server-call-logging` |
| Metrics / Micrometer | `io.ktor/server-metrics-micrometer` |
| OpenTelemetry | `io.opentelemetry.instrumentation/server-open-telemetry` |
| OpenAPI / Swagger UI | `io.ktor/server-openapi`, `io.ktor/server-swagger` |
| Resources (typed routes) | `io.ktor/server-resources` |
| Auto HEAD response | `io.ktor/server-auto-head-response` |
| Default headers | `io.ktor/server-default-headers` |
| Forwarded headers (proxy) | `io.ktor/server-forwarded-header-support` |
| PostgreSQL | `org.jetbrains/server-postgres` |
| MongoDB | `org.jetbrains/server-mongodb` |
| Exposed ORM | `org.jetbrains/server-exposed` |
| Kafka | `io.github.flaxoos/server-kafka` |
| RabbitMQ | `io.github.damirdenis-tudor/server-rabbitmq` |
| Dependency injection | `io.ktor/server-di` |
| Koin DI | `io.insert-koin/server-koin` |
| HTML DSL / server-side HTML (Kotlin-native) | `org.jetbrains/server-html-dsl` |
| gRPC | `org.jetbrains/kotlinx-rpc-grpc` |
| Rate limiting | `io.github.flaxoos/server-rate-limiting` |

## Step 5 — Build the curl command and download

The API accepts a JSON POST body. The response is a zip file with **no wrapping folder** — files unzip directly into the current directory.

Translate artifact identifiers from Step 2 to Ktor API fields:
- `artifactId` → `project_name`
- `groupId` (`com.example`) → `company_website` (reversed: `example.com`)

```bash
PROJECT_NAME="demo-service"       # artifactId from Step 2
KTOR_VERSION="3.4.2"
KOTLIN_VERSION="2.3.20"
ENGINE="NETTY"                    # NETTY | JETTY | CIO | TOMCAT
BUILD_SYSTEM="GRADLE_KTS"         # GRADLE_KTS | MAVEN | AMPER
CONFIG_OPTION="YAML"              # YAML | HOCON | CODE
COMPANY_WEBSITE="example.com"     # groupId reversed from Step 2
# Features: space-separated list of xmlId values to include in the JSON array
FEATURES='"io.ktor/server-content-negotiation","io.ktor/server-kotlinx-serialization","io.ktor/server-cors"'

curl -s "https://start.ktor.io/project/generate" \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d "{\"settings\":{\"project_name\":\"${PROJECT_NAME}\",\"company_website\":\"${COMPANY_WEBSITE}\",\"ktor_version\":\"${KTOR_VERSION}\",\"kotlin_version\":\"${KOTLIN_VERSION}\",\"build_system\":\"${BUILD_SYSTEM}\",\"build_system_args\":null,\"engine\":\"${ENGINE}\"},\"features\":[${FEATURES}],\"configurationOption\":\"${CONFIG_OPTION}\",\"addDefaultRoutes\":true,\"addWrapper\":true}" \
  -o "${PROJECT_NAME}.zip"
```

Verify the download succeeded: `file "${PROJECT_NAME}.zip"` — it should report a Zip archive.
If `file` reports HTML or ASCII text, print the content — the API returned an error. Fix the parameters and retry.

## Step 6 — Unzip

The zip has **no wrapping folder**, so the extraction target must be set up beforehand.

**Root mode** — extract directly into the current directory:
```bash
unzip -q "${PROJECT_NAME}.zip"
rm "${PROJECT_NAME}.zip"
ls -la .
```

**Subproject mode** — create the subfolder first, then extract into it:
```bash
mkdir -p "${PROJECT_NAME}"
unzip -q "${PROJECT_NAME}.zip" -d "${PROJECT_NAME}"
rm "${PROJECT_NAME}.zip"
ls -la "${PROJECT_NAME}"
```

## Step 7 — Build and verify

**Subproject mode** — use a subshell so the working directory is not changed for subsequent commands:
- Gradle: `(cd "${PROJECT_NAME}" && chmod +x gradlew && ./gradlew build --no-daemon -x test 2>&1 | tail -30)`
- Maven: `(cd "${PROJECT_NAME}" && chmod +x mvnw && ./mvnw verify -DskipTests --no-transfer-progress 2>&1 | tail -30)`

**Root mode** — run directly from the current directory:
- Gradle: `chmod +x gradlew && ./gradlew build --no-daemon -x test 2>&1 | tail -30`
- Maven: `chmod +x mvnw && ./mvnw verify -DskipTests --no-transfer-progress 2>&1 | tail -30`

A successful build ends with `BUILD SUCCESSFUL` (Gradle) or `BUILD SUCCESS` (Maven). If the build fails, diagnose from the output.

## Step 8 — Tell the user what was created

After a clean build, report in short form:
- Project location (absolute path)
- Ktor version, Kotlin version, build system, and engine used
- Plugins included
- groupId, artifactId, and generated package used
- How to run: `./gradlew run` (Gradle) or `./mvnw ktor:run` (Maven)
- Configuration format (YAML / HOCON / Code)
- If metadata was unavailable and the user provided versions, note that those were used as-is without validation.
