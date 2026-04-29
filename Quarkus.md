# Quarkus Build & Run Modes

---

## 0. Start infrastructure (local development)

Start Docker and Keycloak before running the app:

```bash
docker-compose -f docker-compose.dev-env.yml up -d
```

---

## 1. Dev Mode (local development)

Hot reload, no build needed. Quarkus recompiles on the fly.

```bash
mvn quarkus:dev
```

- Starts on `http://localhost:8080`
- Dev UI available at `http://localhost:8080/q/dev`
- No need to rebuild on code changes
- DevServices auto-starts Postgres, Kafka, Keycloak via Docker

---

## 2. JVM Mode (standard build)

Compiles to a regular JAR, runs on the JVM. Fastest to build, slowest to start.

**Build:**
```bash
mvn clean package -DskipTests
```

**Run:**
```bash
java -jar target/quarkus-app/quarkus-run.jar
```

With a specific profile:
```bash
java -Dquarkus.profile=dev -jar target/quarkus-app/quarkus-run.jar
```

Output structure:
```
target/
  quarkus-app/
    quarkus-run.jar       ← entry point
    lib/                  ← dependencies
    app/                  ← your classes
    quarkus/              ← quarkus internals
```

---

## 3. Native Mode (GraalVM/Mandrel)

Compiles to a native binary. Slow to build, very fast to start, low memory footprint.

### 3a. Local (GraalVM/Mandrel installed on your machine)

```bash
mvn clean package -Pnative -DskipTests -Dquarkus.native.container-build=false
```

Or to run it locally with no GraalVM needed locally, Docker pulls the GraalVM/Mandrel builder image and runs `native-image` inside it. Your machine only needs Docker.

```bash
mvn clean package -Pnative -DskipTests -Dquarkus.native.container-build=true
```

**Run:**
```bash
./application/api/api-core/target/api-core-999-SNAPSHOT-runner
```

### 3b. Remote container native (your project's setup)

```bash
mvn clean package -Pnative -DskipTests -Dquarkus.profile=dev -T 1
```
and w
It uses whatever the `remote-container-build=true` conf says in the `native` profile of the parent `pom.xml`, in tenant case we have it to `true`, so it uses the company's remote builder image automatically.

**Run:**
```bash
./application/api/api-core/target/api-core-999-SNAPSHOT-runner
```

### 3d. Native build command variants — what's the difference?

These three commands all produce a native binary, but differ in **where** GraalVM runs:

**`mvn package -Pnative`**
Activates the `native` Maven profile from your `pom.xml`. That profile sets `quarkus.package.type=native` plus any other properties your team configured (builder image, memory limits, etc.). Runs `native-image` using GraalVM/Mandrel **installed locally on your machine**.
```bash
mvn package -Pnative -DskipTests
```
→ Requires GraalVM or Mandrel installed locally. Uses whatever extra config your `native` profile defines.

---

**`mvn package -Dquarkus.package.type=native`**
Does the same thing (triggers native compilation) but **bypasses the Maven profile entirely** — sets the property directly on the command line. Useful if your `pom.xml` has no `native` profile, or you want native compilation without activating anything else the profile might configure.
```bash
mvn package "-Dquarkus.package.type=native" -DskipTests
```
→ Requires GraalVM or Mandrel installed locally. No profile side-effects.

---

**`mvn package -Pnative -Dquarkus.native.container-build=true`**
Same as the first, but the extra flag tells Quarkus: *"don't use my local GraalVM — pull the Mandrel builder Docker image and run `native-image` inside a container."* Your machine don't need GraalVM at all.
```bash
mvn package -Pnative "-Dquarkus.native.container-build=true" -DskipTests
```
→ Requires Docker locally. No GraalVM/Mandrel needed. This is what most CI pipelines use.

Here's exactly what happens here step by step:

**1. Maven builds all modules normally** (compile, package) for the non-native ones

**2. For the native modules (`api-core`, `cron`, `listener-core`), Quarkus:**
- Pulls the Mandrel builder Docker image (`quay.io/quarkus/ubi9-quarkus-mandrel-builder-image:...`)
- Spins up a container from that image
- Mounts your project's `target/` folder into the container
- Runs `native-image` inside the container (this is the slow part — 5 to 15 min)
- Outputs the binary back into your local `target/` folder
- Stops and removes the container automatically

**3. You get a binary in:**
```
application/api/api-core/target/api-core-999-SNAPSHOT-runner        ← HTTP API
application/cron/target/cron-999-SNAPSHOT-runner                    ← cron jobs
application/listener/listener-core/target/listener-core-999-SNAPSHOT-runner  ← kafka listener
```

**To run the native binary** (example with api-core):

On WSL/Linux:
```bash
./application/api/api-core/target/api-core-999-SNAPSHOT-runner
```

On PowerShell (the binary is a Linux executable — it won't run on Windows directly):
```powershell
# You need to run it inside WSL
wsl ./application/api/api-core/target/api-core-999-SNAPSHOT-runner
```

With a Quarkus profile:
```bash
./application/api/api-core/target/api-core-999-SNAPSHOT-runner -Dquarkus.profile=dev
```

---

**Summary:**

| Command | Needs local GraalVM | Needs Docker | Uses `pom.xml` profile |
|---|---|---|---|
| `-Pnative` | ✅ Yes | ❌ No | ✅ Yes |
| `-Dquarkus.package.type=native` | ✅ Yes | ❌ No | ❌ No |
| `-Pnative -Dquarkus.native.container-build=true` | ❌ No | ✅ Yes | ✅ Yes |

---

## 4. Native container image (build native + wrap in Docker image)

```bash
mvn clean package -Pnative -DskipTests \
  -Dquarkus.native.container-build=true \
  -Dquarkus.container-image.build=true
```

**Run:**
```bash
docker run -i --rm -p 8080:8080 your-org/your-app:1.0
```

---

## Comparison Table

| Mode | Build time | Startup time | Memory | Use case |
|---|---|---|---|---|
| Dev | None | ~3s | High | Local development |
| JVM | Fast (~30s) | ~2-5s | Medium | Staging / quick test |
| Native local | Very slow (~10min) | ~50ms | Very low | Prod-like local test |
| Native container | Very slow (~10min) | ~50ms | Very low | CI/CD pipeline |

---

## Useful Flags

| Flag | Effect |
|---|---|
| `-DskipTests` | Skip all tests |
| `-Dquarkus.profile=dev` | Use dev profile config |
| `-T 1` | Single thread (saves RAM during native build) |
| `-Pnative` | Activate native Maven profile |
| `-Dquarkus.native.remote-container-build=false` | Force local native build |
| `-Dquarkus.native.container-build=true` | Build native inside local Docker |
| `-Dquarkus.native.native-image-xmx=6G` | Max heap for native compiler |
