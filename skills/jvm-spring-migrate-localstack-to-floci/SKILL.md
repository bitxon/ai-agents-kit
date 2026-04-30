---
name: jvm-spring-migrate-localstack-to-floci
description: "Migrate a project from LocalStack to Floci AWS emulator. Use this skill whenever the user mentions replacing LocalStack with Floci, migrating AWS local emulator, switching from localstack/localstack to floci/floci (formerly hectorvent/floci), or wants to use Floci for local AWS testing. Also trigger when the user mentions LocalStack cost/licensing issues and wants a free alternative."
---

# Migrate Spring Boot from LocalStack to Floci

Migrate a project from LocalStack to [Floci](https://github.com/floci-io/floci) — a free, MIT-licensed AWS emulator that runs on the same port (`4566`) and works with standard AWS SDKs.

Search path: `$ARGUMENTS` (default: current directory).

## Step 1 — Discover LocalStack usage

Search for all LocalStack references before touching anything:

```
Grep: testcontainers-localstack        → build files
Grep: LocalStackContainer              → Testcontainers config
Grep: localstack/localstack            → docker-compose.yaml
Grep: awslocal                         → init scripts
Grep: path-style-access-enabled        → application config (yaml/properties)
Grep: /etc/localstack/init/ready.d     → init script mount paths (reveals the scripts directory)
```

**Locate init scripts directory:**

From the `/etc/localstack/init/ready.d` grep result (or the `LocalStackContainer` / docker-compose volume definitions), extract the **actual host path** of the init scripts directory. It could be anywhere — `src/test/resources/scripts`, `src/test/resources/aws`, `src/test/resources/localstack`, `docker/init`, etc.

- If init scripts exist: note their directory — you will update them in place (Step 4).
- If no init scripts exist: skip Step 4, and omit the `withCopyToContainer` calls in Step 5.

Record this path as `<init-scripts-dir>` — use it consistently in Steps 5 and 7.

Note every file that needs changing before proceeding.

## Step 2 — Resolve the latest versions

Check the project's Spring Boot version (from `build.gradle` or `build.gradle.kts` or `pom.xml`) to determine which Floci version applies:

| Spring Boot | Spring Cloud AWS | Testcontainers | Floci |
|-------------|------------------|----------------|-------|
| 4.x         | 4.x              | 2.x            | 2.x   |
| 3.x         | 3.x              | 1.x            | 1.x   |

Then fetch the latest releases:
- **Maven artifact:** WebFetch `https://central.sonatype.com/artifact/io.floci/spring-boot-testcontainers-floci/versions` → extract the latest version matching the target major.
- **Docker image:** WebFetch `https://hub.docker.com/r/floci/floci/tags` → extract the latest `<version>-aws` tag matching the same major.

Record both for use in later steps:
- `<floci-lib-version>` — used in Step 3
- `<floci-image-tag>` — used in Steps 5 and 7

## Step 3 — Update build dependency

**Gradle (build.gradle.kts):**
```kotlin
// Remove
testImplementation("org.testcontainers:testcontainers-localstack")

// Add
testImplementation("io.floci:spring-boot-testcontainers-floci:<floci-lib-version>")
```

**Maven (pom.xml):**
```xml
<!-- Remove -->
<dependency>
   <groupId>org.testcontainers</groupId>
   <artifactId>localstack</artifactId>
</dependency>

        <!-- Add -->
<dependency>
<groupId>io.floci</groupId>
<artifactId>spring-boot-testcontainers-floci</artifactId>
<version><floci-lib-version></version>
<scope>test</scope>
</dependency>
```

## Step 4 — Update init scripts

For each script in the init directory:

1. Change shebang: `#!/bin/bash` → `#!/bin/sh`
2. Add env vars block at the top (if not already present):
   ```sh
   export AWS_DEFAULT_REGION=us-east-1
   export AWS_ACCESS_KEY_ID=test
   export AWS_SECRET_ACCESS_KEY=test
   export AWS_ENDPOINT_URL=http://localhost:4566
   ```
3. Replace `awslocal <cmd>` → `aws <cmd>` (the env vars above make `aws` point to Floci automatically)
4. Fix hardcoded LocalStack SQS queue URLs:
   - Before: `http://sqs.us-east-1.localhost.localstack.cloud:4566/000000000000/my-queue`
   - After:   `http://localhost:4566/000000000000/my-queue`

## Step 5 — Update TestcontainersConfig

**Before (LocalStack):**
```kotlin
@Bean
fun localStack(): LocalStackContainer {
   return LocalStackContainer(DockerImageName.parse("localstack/localstack:..."))
      .withServices("dynamodb", "s3", "sqs")
      .withCopyToContainer(MountableFile.forClasspathResource("scripts"), "/etc/localstack/init/ready.d")
}

@Bean
fun propertiesOverride(localstack: LocalStackContainer): DynamicPropertyRegistrar {
   return DynamicPropertyRegistrar { registry ->
      registry.add("spring.cloud.aws.endpoint") { localstack.endpoint }
      registry.add("spring.cloud.aws.region.static") { localstack.region }
      registry.add("spring.cloud.aws.credentials.access-key") { localstack.accessKey }
      registry.add("spring.cloud.aws.credentials.secret-key") { localstack.secretKey }
      registry.add("spring.cloud.aws.s3.path-style-access-enabled") { true }
   }
}
```

**After (Floci):**
```kotlin
@Bean
@ServiceConnection  // auto-configures endpoint, region, credentials, and forcePathStyle
fun floci(): FlociContainer {
   return FlociContainer(DockerImageName.parse("floci/floci:<floci-image-tag>"))
      // <classpath-relative-dir> is the classpath-relative path to the init scripts directory
      // e.g. "scripts", "aws", "localstack" — whatever was discovered in Step 1
      .withCopyToContainer(MountableFile.forClasspathResource("<classpath-relative-dir>"), "/etc/floci/init/start.d")
      // Floci runs init scripts async after HTTP is ready; wait for last script's
      // completion message so all AWS resources exist before Spring context starts
      .waitingFor(Wait.forLogMessage(".*<last script echo message>.*", 1))
}
```

If no init scripts exist, omit the `withCopyToContainer` call and the `waitingFor` — the plain `FlociContainer` with `@ServiceConnection` is sufficient.

Key changes:
- `@ServiceConnection` replaces the entire `DynamicPropertyRegistrar` bean for endpoint/region/credentials/forcePathStyle
- Remove `.withServices(...)` — Floci enables all services by default
- Init dir: `/etc/localstack/init/ready.d` → `/etc/floci/init/start.d`
- Add `waitingFor` for the last script's echo message (see gotchas below)

For the classpath-relative path: if the init scripts are at `src/test/resources/aws/`, the classpath-relative path is `"aws"`. Read the existing `withCopyToContainer` or `MountableFile` call in the original config to find it.

For the wait message: use the `echo` output from the **last** init script alphabetically (Floci runs scripts in filename order). For example, if the last script prints `"SQS queue created."`, use `".*SQS queue created.*"`.

**Required imports:**
```kotlin
import io.floci.testcontainers.FlociContainer
import org.springframework.boot.testcontainers.service.connection.ServiceConnection
import org.testcontainers.containers.wait.strategy.Wait
import org.testcontainers.utility.DockerImageName
import org.testcontainers.utility.MountableFile
```

## Step 6 — Update docker-compose.yaml

Use the **host-relative path** to the init scripts directory discovered in Step 1 (`<init-scripts-dir>`), expressed relative to the docker-compose file location.

```yaml
# Before
services:
   localstack:
      image: localstack/localstack:4.14.0
      ports:
         - "127.0.0.1:4566:4566"
      environment:
         - SERVICES=dynamodb,s3,sqs
         - DOCKER_HOST=unix:///var/run/docker.sock
      volumes:
         - "/var/run/docker.sock:/var/run/docker.sock"
         - "<init-scripts-dir>:/etc/localstack/init/ready.d"
```

```yaml
# After
services:
   floci:
      image: floci/floci:<floci-image-tag>
      ports:
         - "127.0.0.1:4566:4566"
      volumes:
         - "<init-scripts-dir>:/etc/floci/init/start.d"
```

Replace `<init-scripts-dir>` with the actual relative path, e.g. `./src/test/resources/scripts`, `./docker/aws`, etc. If no init scripts exist, omit the volume.

Also update any other services that reference `localstack` by name (e.g., `depends_on`, env vars like `DYNAMO_ENDPOINT=http://localstack:4566`).

Remove:
- `SERVICES` env var — Floci enables all services automatically
- `DOCKER_HOST` env var and docker socket volume — not needed by Floci

## Step 7 — Verify

```bash
# Run tests for the affected module
./gradlew :<module>:build

# Or for Maven
./mvnw test -pl <module>
```

---

## Known Issues / Gotchas

| Symptom                                                                                           | Root cause                                                                                                   | Fix                                                                                  |
|---------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| `/bin/bash: not found` on startup                                                                 | Alpine image has no `/bin/bash`; Floci hardcoded bash ([#241](https://github.com/floci-io/floci/issues/241)) | Upgrade to latest `floci/floci:<version>-aws`                                        |
| `Failed to verify that image 'hectorvent/floci:...' is a compatible substitute for 'floci/floci'` | Floci 2.4.0+ expects the canonical `floci/floci` image name; old `hectorvent/floci` images fail validation   | Use `floci/floci:<version>-aws` (Docker Hub was renamed from `hectorvent/floci`)     |
| `QueueDoesNotExistException` at context startup                                                   | Init scripts run **async** after HTTP ready; Spring context starts before scripts finish                     | Add `.waitingFor(Wait.forLogMessage(...))` for last script's output (Step 5)         |
| `ForcePathStyle configured at both S3Configuration and client/global level`                       | `path-style-access-enabled: true` in config + `@ServiceConnection` both set ForcePathStyle                   | Remove `spring.cloud.aws.s3.path-style-access-enabled: true` from application config |
| `awslocal: command not found` in scripts                                                          | `awslocal` is LocalStack-specific CLI wrapper                                                                | Replace with `aws` and set `AWS_ENDPOINT_URL=http://localhost:4566` (Step 4)         |
| SQS send-message-batch fails with wrong queue URL                                                 | Hardcoded LocalStack hostname in SQS URL                                                                     | Replace `sqs.*.localhost.localstack.cloud:4566` → `localhost:4566` (Step 4)          |