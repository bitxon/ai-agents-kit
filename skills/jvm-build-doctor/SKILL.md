---
name: jvm-build-doctor
description: >
  Diagnoses and guides resolution of Maven and Gradle build failures for JVM projects.
  Invoke this skill whenever the user pastes or describes a build error — including
  dependency resolution failures ("Could not find", "Could not resolve"), version
  conflicts/convergence errors, compilation errors ("cannot find symbol", "incompatible
  types"), plugin failures ("Execution failed for task"), or JVM/toolchain version
  mismatches ("class file has wrong version"). Auto-trigger on any Maven or Gradle
  error output, even a partial paste. Use this skill proactively whenever the user
  says their build is broken, asks why a dependency isn't found, or shares a
  pom.xml/build.gradle asking why it fails — even without an explicit stack trace.
---

# JVM Build Doctor

When a user shares a build error, diagnose the likely cause and recommend a resolution path. The goal is to be **token-efficient**: identify the error category quickly, apply the most targeted lookup strategy first, and avoid wasteful requests to sites that block automated access.

> **Command convention**: prefer the wrapper (`./gradlew`, `./mvnw`) when it exists in the project. Fall back to the system installation (`gradle`, `mvn`) if the wrapper is missing or broken. All examples below use the short form — apply the wrapper prefix as appropriate.

## Error Categories

| Category | Typical signals |
|---|---|
| **Dependency resolution** | "Could not find", "Could not resolve", artifact/version missing |
| **Version conflict** | Convergence failure, "Conflict with dependency", multiple versions of same artifact |
| **Compilation error** | "cannot find symbol", "incompatible types", "method does not apply" |
| **Plugin failure** | "Execution failed for task", plugin class not found, task configuration error |
| **JVM/toolchain mismatch** | "class file has wrong version", source/target incompatibility, JAVA_HOME issues |

---

### Dependency resolution

Start with an exact lookup on Maven Central (see [Hints › Maven Central Lookups](#maven-central-lookups)) to confirm whether the artifact and version exist.

**If the artifact doesn't exist at that groupId/artifactId** — don't shotgun WebFetch calls. Do one targeted WebSearch for the groupId + artifactId together. Common causes:
- GroupId was renamed (e.g. `tools.jackson` → `com.fasterxml.jackson.core`)
- ArtifactId was renamed (e.g. `org.testcontainers:postgresql` → `org.testcontainers:testcontainers-postgresql`)
- Extra or missing segment in groupId
- Typo in artifactId
- Library absorbed into another artifact

**If the artifact exists but the requested version is missing** — `maven-metadata.xml` lists all published versions. Compare `<latest>`/`<release>` to what the build requested. Also check the POM of the nearest existing version for a `<relocation>` element — this signals the artifact moved to a new coordinate.

---

### Version conflict

When two dependencies pull different versions of the same artifact, build tools report convergence failures. The fix is always the same — pick a version and declare it authoritatively — but the mechanism differs:

- **Gradle**: `gradle dependencies --configuration compileClasspath` shows the full tree with `->` indicating forced resolutions. Use `resolutionStrategy.force` or a platform/BOM + `constraints` block to pin the version.
- **Maven**: `mvn dependency:tree` reveals the tree. Use `<dependencyManagement>` to declare the winning version without introducing a direct dependency.

If it's unclear which version is safe, use the Maven Central POM lookup (see [Hints › Maven Central Lookups](#maven-central-lookups)) to inspect the conflicting artifact's own dependencies and understand what introduced the pull.

---

### Compilation error

"Cannot find symbol" and "method does not apply" after a version bump almost always mean an API change — method renamed, moved to a different class, or removed entirely. Rather than guessing:

1. Identify the class/method that's missing from the error message.
2. Do a WebSearch for `<library> <version> migration` or `<class-name> removed/renamed <library>` to find the changelog or migration guide.
3. Check whether the import path changed (common in major version bumps, e.g. `javax.*` → `jakarta.*`).

If the error appears on fresh code (no recent version bump), it's more likely a missing dependency or wrong scope.

---

### Plugin failure

"Execution failed for task" with a plugin error usually points to:
- Plugin version incompatible with the current Gradle/Maven version
- Plugin version incompatible with the Java version being used
- Missing plugin configuration (required properties not set)

Check `gradle --version` or `mvn --version`, then WebSearch `<plugin-name> compatibility <tool-version>` to find the supported matrix. Check if a newer plugin version resolves it before changing the build tool version.

---

### JVM/toolchain mismatch

#### JVM/toolchain mismatch — Gradle

**Step 1 — Check active Java and Gradle versions**
```bash
java -version
gradle --version
```

**Step 2 — Check compatibility matrix**

| Java | Min Gradle |
|---|---|
| 17 | 7.3 |
| 21 | 8.5 |
| 25 | 9.1 |

Full matrix: https://docs.gradle.org/current/userguide/compatibility.html

> This step is critical — if Java is too new for Gradle itself, the build crashes before toolchain resolution even starts (e.g. `Unsupported class file major version`). Toolchain config cannot fix this; upgrade Gradle instead (see [Hints › Upgrade Gradle Wrapper](#upgrade-gradle-wrapper)).

**Step 3 — Check what Java version the build expects**

In `build.gradle` / `build.gradle.kts`:
- Toolchain: `java { toolchain { languageVersion = X } }` → run `gradle -q javaToolchains` to see available versions
- Effective version: `java { sourceCompatibility = X }`

**Step 4 — Analyze and fix**

| Situation | Fix |
|---|---|
| Java too new for Gradle | Upgrade Gradle wrapper (see [Hints › Upgrade Gradle Wrapper](#upgrade-gradle-wrapper)) |
| Toolchain version not found | Install the required JDK or add a toolchain resolver plugin to auto-provision |
| Wrong JDK used (no toolchain declared) | Declare a toolchain or set `JAVA_HOME` explicitly |
| Stale daemon | `gradle --stop`, then rebuild |
| Bytecode too new for runtime | Lower `sourceCompatibility`/`--release` or upgrade the runtime JVM |

---

#### JVM/toolchain mismatch — Maven

**Step 1 — Check active Java and Maven versions**
```bash
java -version
mvn --version
```

**Step 2 — Check compatibility**

Unlike Gradle, Maven doesn't perform bytecode version checks at startup — it runs on any JVM ≥ 8 and won't hard-crash on a newer Java. Skip to Step 3 unless you have a specific reason to suspect the Maven version itself.

**Step 3 — Check what Java version the build expects**

In `pom.xml`:
- Toolchain: `maven-toolchains-plugin` configuration → verify the version has an entry in `~/.m2/toolchains.xml`
- Effective version: `maven-compiler-plugin` configuration or `<properties>` (`maven.compiler.source` / `maven.compiler.release`)

**Step 4 — Analyze and fix**

| Situation | Fix |
|---|---|
| Java too new for Maven | Upgrade Maven wrapper (see [Upgrade Maven Wrapper](#upgrade-maven-wrapper)) |
| Toolchain entry missing | Add the JDK to `~/.m2/toolchains.xml` with the correct `<jdkHome>` |
| Wrong `JAVA_HOME` | `export JAVA_HOME=$(/usr/libexec/java_home -v 17)` (macOS) |
| Bytecode too new for runtime | Lower `<source>`/`<release>` in `maven-compiler-plugin` or upgrade the runtime JVM |

---

## Hints

### Maven Central Lookups

**Maven Central** (`repo1.maven.org`) is the source of truth for artifact metadata.
- It's accessible via `curl` with no auth, no JavaScript, and no blocking.
- `groupId` must be transformed dots → slashes (e.g. `com.fasterxml.jackson.core` → `com/fasterxml/jackson/core`)

**Lookup Endpoints:**
- Artifact and available versions - `curl -s "https://repo1.maven.org/maven2/{group/id}/{artifactId}/maven-metadata.xml"` - Returns `<latest>`, `<release>`, and full `<versions>` list.
- Inspect transitive dependencies, relocations - `curl -s "https://repo1.maven.org/maven2/{group/id}/{artifactId}/{version}/{artifactId}-{version}.pom"` - Returns all `<dependency>` entries with scopes, and any `<relocation>` block if the artifact moved.

**Sites to avoid — they will fail or mislead:**
| Site | Why |
|---|---|
| `mvnrepository.com` | Cloudflare blocks all automated access (HTTP 403) |
| `search.maven.org` Solr API | Stale index — has returned wrong latest versions |
| `central.sonatype.com` REST endpoints | Return 404 or require JavaScript rendering |

### Upgrade Gradle Wrapper

**Never edit `gradle-wrapper.properties` manually** — `distributionUrl` and the `gradle-wrapper.jar` binary must stay in sync; only the wrapper task updates both.

```bash
./gradlew wrapper --gradle-version 8.1.1    # Upgrade to a specific version
./gradlew wrapper --gradle-version latest   # Upgrade to latest (resolves at execution time)

# If ./gradlew itself is broken, use the system Gradle installation
gradle wrapper --gradle-version 8.1.1
gradle wrapper --gradle-version latest
```

### Upgrade Maven Wrapper

**Never edit `.mvn/wrapper/maven-wrapper.properties` manually** — `distributionUrl` and the `maven-wrapper.jar` binary must stay in sync; only the wrapper plugin updates both.

```bash
mvn wrapper:wrapper -Dmaven=3.9.6     # Upgrade to a specific version (system Maven)
./mvnw wrapper:wrapper -Dmaven=3.9.6  # If ./mvnw still works
```

Maven wrapper has no `latest` shortcut — check the current release at https://maven.apache.org/download.cgi and specify the version explicitly.

---

## Example

**Error:**
```
Execution failed for task ':compileTestJava'.
> Could not resolve all files for configuration ':testCompileClasspath'.
   > Could not find tools.jackson:jackson-databind:9.0.1.
```

**Diagnosis:**
Two things look wrong `tools.jackson` is not a known groupId for jackson-databind — the correct one is `com.fasterxml.jackson.core`. Version `9.0.1` also doesn't exist in any known jackson release line (latest stable is 2.x). Confirm the artifact is absent under `tools.jackson` via Maven Central, then WebSearch "tools.jackson jackson-databind" to surface the rename. Fix the groupId and pick a real version from `maven-metadata.xml` (`<release>` for latest stable).
