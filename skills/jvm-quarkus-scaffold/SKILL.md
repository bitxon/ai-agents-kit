---
name: jvm-quarkus-scaffold
description: "Scaffolds a new Java or Kotlin Quarkus project. Use when the user wants to create, bootstrap, initialize, or start a Quarkus project, microservice, REST API, or JVM backend service with Quarkus — including vague requests like 'set up a Quarkus app', 'create a Quarkus project', 'start a new Quarkus service'. Do NOT use if the user mentions Spring Boot, Ktor, Dropwizard, Helidon, or Micronaut."
---

# Quarkus Scaffold

This skill creates a working Quarkus project using the `code.quarkus.io` generator API, unzips it, and verifies the build compiles cleanly.

## Step 1 — Determine the placement mode

Decide where the project files should land **before** doing anything else. Ask if the user hasn't been clear.

| Signal | Mode | Description |
|---|---|---|
| "in the current folder", "here", "in this directory", "as root project", or no placement cue | **root** (default) | files go directly into the current working directory |
| "as a module", "in a subfolder", "as a subproject" | **subproject** | files go into a new subfolder named after the artifactId |

Before proceeding, check if the target directory already contains a project (look for `build.gradle.kts`, `build.gradle`, `pom.xml`, `gradlew`, or `mvnw`). If found, warn the user and ask for confirmation before continuing.

## Step 2 — Determine artifact identifiers

- **artifactId** — project / service name, lowercase-hyphenated (`demo-service`).
  Derive from task context. If not determinable, ask the user.
- **groupId** — company domain in reverse notation (`com.example`).
  Derive only from what the user explicitly stated in the prompt (e.g. "use com.acme", "for Acme Corp", "domain is acme.com"). Do **not** infer from email addresses, session info, git config, or any other implicit source. If not explicitly provided, ask the user.

## Step 3 — Fetch code.quarkus.io metadata

WebFetch `https://code.quarkus.io/api/streams` and extract:
- **Latest stable Quarkus version**: the entry where `recommended` is `true` and `status` is `FINAL`. Use its `quarkusCoreVersion`.
- **Supported Java versions**: from the recommended stream's `javaCompatibility` object — note the `recommended` Java version.

If the fetch fails, proceed — Step 4 handles the fallback.

## Step 4 — Determine the tech stack

Before generating anything, infer the right choices from the task context. Ask if the user hasn't been clear.

### Language

| Signal | Choice |
|---|---|
| User says "Java" or doesn't specify (default) | `java` |
| User says "Kotlin" explicitly | `kotlin` — automatically add `quarkus-kotlin` to extensions |

### Build tool

| Signal | Choice |
|---|---|
| User says "Gradle" or doesn't specify for Java (default) | `GRADLE` |
| User says "Kotlin" or "Gradle Kotlin DSL" (default for Kotlin) | `GRADLE_KOTLIN_DSL` |
| User says "Maven" explicitly | `MAVEN` |

### Java version

| Metadata available | User specified | Action |
|---|---|---|
| yes | no | use `javaCompatibility.recommended` from the recommended stream |
| yes | yes, supported | use user's value |
| yes | yes, **not** supported | warn and confirm before proceeding |
| no | no | fall back to `21` |
| no | yes | trust user; note at end that version was not validated |

### Quarkus version

| Metadata available | User specified | Action |
|---|---|---|
| yes | no | use `quarkusCoreVersion` from the recommended stream |
| yes | yes | use user's value |
| no | no | fall back to `3.34.6` |
| no | yes | trust user; note at end that version was not validated |

### Extensions

Map the task description to extension IDs. After selecting from the table, verify each ID exists in `https://code.quarkus.io/api/extensions?platformOnly=true` — the fetched JSON is the source of truth. If a selected ID is not found, flag it to the user before proceeding.

| Task / technology mentioned | Extension ID(s) |
|---|---|
| REST API / HTTP endpoints (reactive, default) | `quarkus-rest`, `quarkus-rest-jackson` |
| REST API / HTTP endpoints (classic JAX-RS) | `quarkus-resteasy`, `quarkus-resteasy-jackson` |
| REST API with Kotlin serialization | `quarkus-rest`, `quarkus-rest-kotlin-serialization` |
| REST client (reactive) | `quarkus-rest-client`, `quarkus-rest-client-jackson` |
| Validation (Bean Validation / JSR-380) | `quarkus-hibernate-validator` |
| JPA / relational DB with Panache | `quarkus-hibernate-orm`, `quarkus-hibernate-orm-panache` |
| JPA / Panache (Kotlin) | `quarkus-hibernate-orm-panache-kotlin` |
| Reactive DB / Hibernate Reactive + Panache | `quarkus-hibernate-reactive-panache` |
| Reactive DB / Panache (Kotlin) | `quarkus-hibernate-reactive-panache-kotlin` |
| PostgreSQL | `quarkus-hibernate-orm-panache`, `quarkus-jdbc-postgresql` |
| MySQL | `quarkus-hibernate-orm-panache`, `quarkus-jdbc-mysql` |
| H2 in-memory database | `quarkus-hibernate-orm-panache`, `quarkus-jdbc-h2` |
| Flyway migrations | `quarkus-flyway` |
| Liquibase migrations | `quarkus-liquibase` |
| MongoDB | `quarkus-mongodb-panache` |
| MongoDB (Kotlin) | `quarkus-mongodb-panache-kotlin` |
| Redis cache | `quarkus-redis-client` |
| Kafka / reactive messaging | `quarkus-messaging-kafka` |
| Security / OIDC / OAuth2 resource server | `quarkus-oidc` |
| JWT bearer auth | `quarkus-smallrye-jwt` |
| GraphQL | `quarkus-smallrye-graphql` |
| gRPC | `quarkus-grpc` |
| API docs / OpenAPI / Swagger UI | `quarkus-smallrye-openapi` |
| Health checks / liveness / readiness | `quarkus-smallrye-health` |
| Metrics / Micrometer | `quarkus-micrometer` |
| Prometheus scraping | `quarkus-micrometer`, `quarkus-micrometer-registry-prometheus` |
| Scheduled tasks / cron | `quarkus-scheduler` |
| Application data caching | `quarkus-cache` |
| WebSockets | `quarkus-websockets-next` |
| Kotlin support | `quarkus-kotlin` _(added automatically when language = Kotlin)_ |

For Kotlin projects, always include `quarkus-kotlin` regardless of other selections.

## Step 5 — Build the curl command and download

The `code.quarkus.io/d` endpoint is a GET request with query parameters. The response is a zip file that contains a **wrapping folder** named after the artifactId.

```bash
GROUP_ID="com.example"              # from Step 2
ARTIFACT_ID="demo-service"          # from Step 2
BUILD_TOOL="GRADLE"                 # MAVEN | GRADLE | GRADLE_KOTLIN_DSL
JAVA_VERSION="21"
# Build &e= query segments for each extension (one per extension)
EXTENSIONS="e=quarkus-rest&e=quarkus-rest-jackson&e=quarkus-smallrye-health"

curl -sL \
  "https://code.quarkus.io/d?g=${GROUP_ID}&a=${ARTIFACT_ID}&b=${BUILD_TOOL}&j=${JAVA_VERSION}&${EXTENSIONS}" \
  -o "${ARTIFACT_ID}.zip"
```

Verify the download succeeded: `file "${ARTIFACT_ID}.zip"` — it should report a Zip archive.
If `file` reports HTML or ASCII text, print the file content — the API returned an error. Fix the parameters and retry.

## Step 6 — Unzip

The zip always contains a wrapping folder named after the artifactId.

**Root mode** — extract, then promote contents to the current directory:
```bash
unzip -q "${ARTIFACT_ID}.zip"
cp -a "${ARTIFACT_ID}/." .
rm -rf "${ARTIFACT_ID}"
rm "${ARTIFACT_ID}.zip"
ls -la .
```

**Subproject mode** — the wrapping folder becomes the subproject directly:
```bash
unzip -q "${ARTIFACT_ID}.zip"
rm "${ARTIFACT_ID}.zip"
ls -la "${ARTIFACT_ID}"
```

## Step 7 — Build and verify

**Subproject mode** — use a subshell so the working directory is not changed for subsequent commands:
- Gradle: `(cd "${ARTIFACT_ID}" && chmod +x gradlew && ./gradlew build --no-daemon -x test 2>&1 | tail -30)`
- Maven: `(cd "${ARTIFACT_ID}" && chmod +x mvnw && ./mvnw verify -DskipTests --no-transfer-progress 2>&1 | tail -30)`

**Root mode** — run directly from the current directory:
- Gradle: `chmod +x gradlew && ./gradlew build --no-daemon -x test 2>&1 | tail -30`
- Maven: `chmod +x mvnw && ./mvnw verify -DskipTests --no-transfer-progress 2>&1 | tail -30`

A successful build ends with `BUILD SUCCESSFUL` (Gradle) or `BUILD SUCCESS` (Maven). If the build fails, diagnose from the output.

## Step 8 — Tell the user what was created

After a clean build, report in short form:
- Project location (absolute path)
- Language, Quarkus version, build tool, and Java version used
- Extensions included
- groupId and artifactId used
- How to run dev mode: `./gradlew quarkusDev` (Gradle) or `./mvnw quarkus:dev` (Maven)
- If code.quarkus.io metadata was unavailable and the user provided versions, note that those were used as-is without validation.
