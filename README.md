# Docker for Java Applications — Complete Mastery Guide
> Based on your real Quarkus Dockerfiles: JVM, Legacy-JAR, Native, Native-Micro

---

## PART 1 — The Java Toolchain: JDK, JRE, JVM, JIT (and what actually matters in Docker)

Before touching a single Dockerfile, you need to understand what Java actually *needs* to run.

### The Family Tree

```
JDK  (Java Development Kit)
 ├── Compiler (javac) — turns .java → .class bytecode
 ├── Debugger (jdb)
 ├── Build tools (jar, javadoc, jlink...)
 └── JRE  (Java Runtime Environment)
      ├── Class libraries (java.lang, java.util, etc.)
      └── JVM  (Java Virtual Machine)
           ├── Class Loader — loads .class files into memory
           ├── Bytecode Verifier — security check
           ├── Interpreter — executes bytecode line by line (slow at start)
           └── JIT Compiler (Just-In-Time) — watches hot code, compiles to native machine code on the fly
```

**The key mental model:**
- You write `.java` files
- The JDK compiler (`javac`) turns them into `.class` files (bytecode — not machine code, not readable text)
- The JVM runs that bytecode on ANY operating system — this is the "write once, run anywhere" promise
- The JIT watches which methods are called repeatedly and compiles those to native CPU instructions at runtime for speed

### What does this mean for Docker?

| Need | What to include in image |
|---|---|
| Build the app | Full **JDK** |
| Run a JVM app | Only **JRE** (no compiler needed) |
| Run a native app | **Nothing** — the binary IS the machine code |

Your `Dockerfile.jvm` uses `ubi9/openjdk-21` which is a JRE (Red Hat's trimmed runtime). Your `Dockerfile.native` uses `ubi9-minimal` with zero Java at all.

### JDK Distributions — Why So Many Names?

You'll encounter: OpenJDK, GraalVM, Temurin, Corretto, Zulu, Mandrel...

They are all "flavors" of the same open-source OpenJDK codebase. The differences:
- **OpenJDK** — the reference implementation (Red Hat ships this in `ubi9/openjdk-21`)
- **GraalVM** — adds a powerful JIT + the native image compiler (`native-image` binary) — this is what Quarkus uses to build native executables
- **Mandrel** — Red Hat's GraalVM distribution stripped to only what Quarkus needs for native builds
- **Temurin** — Eclipse Foundation build, popular for Spring Boot

For your work: use **Mandrel or GraalVM** to BUILD native images, use **OpenJDK JRE** to RUN JVM images.

---

## PART 2 — How Your Build Output Works (What Maven Puts in Your `target/` Folder)

Before understanding the Dockerfiles, you need to understand what Maven puts in your `target/` folder.

### Fast-JAR (your `Dockerfile.jvm`)

When you run `./mvnw package`, Quarkus produces:
```
target/quarkus-app/
├── quarkus-run.jar          ← tiny 8KB launcher JAR (just Main class)
├── lib/
│   ├── boot/                ← Quarkus bootstrap JARs
│   └── main/                ← all your dependency JARs (hibernate, jackson, etc.)
├── app/
│   └── my-app-1.0.jar       ← YOUR application code only
└── quarkus/
    ├── generated-bytecode.jar
    └── transformed-bytecode.jar  ← Quarkus pre-processed bytecode
```

`quarkus-run.jar` is the entry point. It reads a manifest that says "load everything from these directories." This is why you copy 4 separate folders — they map exactly to this structure.

### Legacy-JAR (your `Dockerfile.legacy-jar`)

When you run `./mvnw package -Dquarkus.package.type=legacy-jar`:
```
target/
├── my-app-1.0.0-runner.jar   ← fat/uber JAR with everything inside
└── lib/                      ← external dependencies as separate JARs
```

This is the old way. Everything is jammed into one JAR. Simple but less cache-friendly.

### Native (your `Dockerfile.native` and `Dockerfile.native-micro`)

When you run `./mvnw package -Pnative`:
```
target/
└── my-app-1.0.0-runner       ← a Linux executable binary (no .jar, no .class)
```

This binary contains: your code + all libraries + a mini garbage collector + everything — compiled ahead-of-time to x86/ARM machine code by GraalVM's `native-image` tool. **Zero JVM needed to run this.**

---

## PART 3 — Every Dockerfile Instruction Explained (With Your Real Files)

### 3.1 — `Dockerfile.jvm` line by line

```dockerfile
FROM registry.access.redhat.com/ubi9/openjdk-21:1.22
```
**What it does:** Sets the base image — the starting filesystem. `ubi9` = Red Hat Universal Base Image 9 (based on RHEL 9). `openjdk-21` means a JRE for Java 21 is pre-installed. Everything you add goes on top of this layer.

**Why ubi9?** It's enterprise-grade, has security patches, and Red Hat maintains it. For Spring Boot you'd often see `eclipse-temurin:21-jre-alpine` instead (much smaller, ~180MB vs ~400MB).

```dockerfile
ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en'
```
**What it does:** Sets environment variables baked into the image. Every process that runs inside the container inherits these. `LANG` affects how Java formats dates, numbers, and reads files. Without this, Java may behave oddly with non-ASCII characters.

```dockerfile
COPY --chown=185 target/quarkus-app/lib/ /deployments/lib/
COPY --chown=185 target/quarkus-app/*.jar /deployments/
COPY --chown=185 target/quarkus-app/app/ /deployments/app/
COPY --chown=185 target/quarkus-app/quarkus/ /deployments/quarkus/
```
**What it does:** Copies files from your host machine (where you run `docker build`) INTO the image. The destination is `/deployments/` which is where the `run-java.sh` script in the base image looks for the app.

**`--chown=185`** — Sets file ownership to user ID 185 (the non-root user defined in the base image). This is a security practice. If the app gets compromised, it can't write to system files because it's not root.

**WHY 4 SEPARATE COPY instructions?** This is the most important optimization in this file:

Docker builds images in **layers**. Each instruction creates a new layer. Layers are cached. If a layer hasn't changed, Docker reuses it from cache without re-running it.

```
Layer 1: FROM (base image)              → never changes
Layer 2: ENV LANG                       → almost never changes
Layer 3: COPY lib/                      → changes only when you add a new dependency
Layer 4: COPY *.jar (quarkus-run.jar)   → rarely changes
Layer 5: COPY app/                      → changes every time you change YOUR code
Layer 6: COPY quarkus/                  → changes when Quarkus config changes
```

When you change a line of business logic, only `app/` changes. Docker reuses layers 1-4 from cache and only rebuilds layers 5-6. This makes rebuilds **10x faster** and **registry pushes much smaller** (only the changed layers are pushed).

If you had written `COPY target/quarkus-app/ /deployments/` (one line), Docker would invalidate the entire thing every single build.

```dockerfile
EXPOSE 8080
```
**What it does:** Documents that the container listens on port 8080. This is metadata only — it does NOT actually open any port. You still need `-p 8080:8080` when running `docker run`. Think of it as "this is the intended port for whoever runs this image."

```dockerfile
USER 185
```
**What it does:** Switches the active user for all subsequent instructions AND for the container process at runtime. After this line, nothing runs as root. Security best practice — never run production containers as root.

```dockerfile
ENV AB_JOLOKIA_OFF=""
ENV JAVA_OPTS="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
ENV JAVA_APP_JAR="/deployments/quarkus-run.jar"
```
These ENVs are read by `run-java.sh` (a script baked into the base image):
- `AB_JOLOKIA_OFF=""` — disables Jolokia JMX agent (you don't want it leaking metrics endpoints in prod)
- `JAVA_OPTS` — JVM flags:
  - `-Dquarkus.http.host=0.0.0.0` — tells Quarkus to listen on ALL network interfaces (not just localhost — critical in containers, otherwise the port is unreachable from outside)
  - `-Djava.util.logging.manager=...` — uses JBoss log manager, needed for Quarkus logging to work correctly
- `JAVA_APP_JAR` — tells `run-java.sh` which JAR to run

Notice there is **no CMD or ENTRYPOINT** — they're inherited from the base image (`ubi9/openjdk-21`). That base image already has `ENTRYPOINT ["/opt/jboss/container/java/run/run-java.sh"]`.

---

### 3.2 — `Dockerfile.legacy-jar` line by line

```dockerfile
FROM registry.access.redhat.com/ubi9/openjdk-21:1.22
ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en'

COPY target/lib/* /deployments/lib/
COPY target/*-runner.jar /deployments/quarkus-run.jar
```

Same base, but only 2 COPY instructions. The legacy format puts all deps in one fat JAR (`-runner.jar`), so there's less to copy — but you lose the fine-grained layer caching of the fast-jar approach.

**When would you use this?** For compatibility with tooling that expects a single JAR, or for simpler deployments where build speed isn't a priority. In most cases, prefer the fast-jar format.

---

### 3.3 — `Dockerfile.native` line by line

```dockerfile
FROM registry.access.redhat.com/ubi9-minimal:9.6
```
`ubi9-minimal` instead of `ubi9/openjdk-21`. No JDK, no JRE, just the bare RHEL 9 system libraries (~100MB vs ~400MB). Since native images don't need Java, you save ~300MB per image.

```dockerfile
WORKDIR /work/
```
**What it does:** Sets the working directory for all subsequent instructions. If `/work/` doesn't exist, Docker creates it. It's the "cd into this directory" for the container. After this, `COPY ./file` means `COPY /work/file`.

```dockerfile
RUN chown 1001 /work \
    && chmod "g+rwX" /work \
    && chown 1001:root /work
```
**What it does:** Runs a shell command inside the image during build time. Here it sets permissions on `/work/` so user 1001 can write to it. The `&&` chains commands in one `RUN` instruction — this is important because each `RUN` is its own layer. Chaining keeps it as one layer.

```dockerfile
COPY --chown=1001:root target/*-runner /work/application
```
Copies the native binary (`my-app-1.0.0-runner`, the executable file) to `/work/application`. The `*` glob matches any filename ending in `-runner`.

```dockerfile
CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
```
**What it does:** Defines the default command to run when the container starts. The exec form `["./application", "arg"]` is preferred over shell form `./application arg` because:
- Exec form: process PID 1 IS your application → it receives OS signals (SIGTERM for graceful shutdown) directly
- Shell form: a `/bin/sh` process is PID 1, and it may not forward signals to your app → container doesn't shut down gracefully

**`-Dquarkus.http.host=0.0.0.0`** — this looks like a JVM flag (`-D`), but there's no JVM! Quarkus native images support this argument syntax for compatibility with the JVM mode configuration style.

---

### 3.4 — `Dockerfile.native-micro` line by line (most complex)

This is a **multi-stage build** — the most powerful Docker pattern.

```dockerfile
FROM registry.access.redhat.com/ubi9:latest AS builder
```
Stage 1, named `builder`. We use a full ubi9 here because we need `dnf` (package manager) to install system libraries.

```dockerfile
RUN mkdir -p /mnt/rootfs
```
Creates a directory that will become the root filesystem for our final tiny image.

```dockerfile
RUN dnf install --installroot /mnt/rootfs \
    redhat-release --releasever 9 \
    --setopt install_weak_deps=false --nodocs --nogpgcheck -y && \
    rpm --root=/mnt/rootfs --import /mnt/rootfs/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```
Installs packages INTO `/mnt/rootfs` (not into the builder itself). `--installroot` is like a chroot — all files go there. This constructs a minimal Linux filesystem from scratch. GPG key import is for signature verification in later steps.

```dockerfile
RUN dnf install --installroot /mnt/rootfs \
    coreutils-single \        # basic Linux commands (ls, cp, echo...)
    glibc-minimal-langpack \  # C standard library (native binaries need this)
    curl-minimal \            # for health checks
    openssl-libs \            # TLS support
    ca-certificates \         # trust store (HTTPS calls need this)
    zlib \                    # compression
    ...
```
Only the absolute minimum libraries the native binary needs to run. No bash, no package manager, no kernel, no editor. Just the C runtime and TLS libs.

```dockerfile
RUN rm -rf /mnt/rootfs/var/cache/* /mnt/rootfs/var/log/dnf* ...
```
Cleans up package manager caches from the rootfs before we copy it. These files are useless at runtime and add megabytes.

```dockerfile
FROM scratch
```
**`scratch`** is Docker's magic empty image — literally nothing. Zero bytes. No OS, no shell, no files. We build the filesystem ourselves by copying from the builder.

```dockerfile
COPY --from=builder /mnt/rootfs/ /
COPY --from=builder /etc/yum.repos.d/ubi.repo /etc/yum.repos.d/
```
`--from=builder` references the first stage. We copy our carefully constructed minimal rootfs into the final image. The final image only contains what WE explicitly put in it.

The rest is identical to `Dockerfile.native`. Result: an image potentially under 60MB vs 400MB+ for the JVM version.

---

## PART 4 — The Four Image Types: Which to Use When

| | JVM Fast-JAR | Legacy-JAR | Native | Native-Micro |
|---|---|---|---|---|
| **Image size** | ~400MB | ~400MB | ~150MB | ~60MB |
| **Startup time** | 1-5 seconds | 1-5 seconds | 10-50ms | 10-50ms |
| **Memory (idle)** | 200-500MB | 200-500MB | 20-50MB | 20-50MB |
| **Build time** | 30 seconds | 30 seconds | 5-20 minutes | 5-20 minutes |
| **Peak throughput** | Highest (JIT) | Highest (JIT) | Medium | Medium |
| **Needs JVM in image** | Yes | Yes | No | No |
| **Debugging** | Easy | Easy | Hard | Hard |
| **Reflection/dynamic** | Full support | Full support | Limited | Limited |

**Use JVM Fast-JAR when:** Your app uses lots of reflection, you want maximum throughput under sustained load, you need easy debugging, or build times matter.

**Use Native/Native-Micro when:** You're building serverless functions, need sub-second cold starts, have strict memory constraints, or run thousands of instances (cost savings on memory).

**Use Legacy-JAR when:** You're migrating an old app packaged as a fat JAR and don't want to restructure yet.

---

## PART 5 — How a Container Actually Runs Your Java App

When you do `docker run quarkus/tenant-jvm`, here's the exact sequence:

1. Docker creates an isolated Linux namespace (its own network, filesystem, process tree)
2. The image filesystem is mounted (read-only layers + a writable top layer)
3. The ENTRYPOINT command starts: `/opt/jboss/container/java/run/run-java.sh`
4. `run-java.sh` reads the ENV vars you set (`JAVA_OPTS`, `JAVA_APP_JAR`, etc.)
5. It calculates JVM memory settings based on container memory limits
6. It executes: `java [computed-options] -jar /deployments/quarkus-run.jar`
7. The JVM starts, loads `quarkus-run.jar`, reads its manifest
8. The manifest says: classpath includes `/deployments/lib/boot/`, `/deployments/lib/main/`, `/deployments/app/`
9. Quarkus bootstrap runs, sets up dependency injection, starts HTTP server
10. Port 8080 inside the container is ready

**For native images:**
Steps 3-8 are replaced by: the binary starts, runs its pre-built initialization, HTTP server is up. No JVM startup, no classloading, no JIT warmup. That's why it's 10ms vs 2000ms.

---

## PART 6 — Brokers (Kafka, RabbitMQ) in Docker

When your Java app connects to a broker, the broker is usually a **separate container**. Your app container just has the client library (kafka-clients.jar etc) in its `/deployments/lib/`.

### The network problem

Inside Docker, containers can't reach each other by IP address directly (the IP changes every restart). You use **Docker networks** and container names:

```yaml
# docker-compose.yml
services:
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    hostname: kafka
    environment:
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092

  my-quarkus-app:
    image: quarkus/tenant-jvm
    environment:
      # Use the SERVICE NAME as hostname — Docker DNS resolves it
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    depends_on:
      - kafka
```

Your app's `application.properties` would have:
```properties
kafka.bootstrap.servers=${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
```

The `${VAR:default}` syntax means "use the env var if present, else use localhost:9092 for local dev without Docker."

### Consumer containers vs API containers

One image, two different behaviors. The same JAR can run as an HTTP API or as a Kafka consumer depending on config:

```yaml
services:
  api:
    image: quarkus/tenant-jvm
    environment:
      QUARKUS_HTTP_PORT: 8080

  consumer:
    image: quarkus/tenant-jvm  # same image!
    environment:
      QUARKUS_HTTP_PORT: 0      # disable HTTP
      CONSUMER_ENABLED: "true"
    # No ports exposed — this container just processes messages
```

Or with Quarkus profiles:
```yaml
  consumer:
    image: quarkus/tenant-jvm
    environment:
      QUARKUS_PROFILE: consumer  # activates @IfBuildProfile("consumer") beans
```

---

## PART 7 — Cron Jobs in Docker

There are three ways to handle scheduled jobs in Java containers:

### Option 1: Built-in scheduler (Quarkus `@Scheduled`)

Your app container runs permanently and an internal thread fires the job:

```java
@ApplicationScoped
public class MyJob {
    @Scheduled(cron = "0 0 2 * * ?")  // every day at 2 AM
    void run() { ... }
}
```

In Docker: just a normal long-running container. Nothing special needed.

**Pros:** Simple, lives with the app, can access all app beans/services.  
**Cons:** The container must stay alive even when idle. If it crashes between runs, the job doesn't run.

### Option 2: Dedicated cron container (same image, different config)

```yaml
services:
  cron-daily-report:
    image: quarkus/tenant-jvm
    environment:
      JOB_NAME: "daily-report"
      QUARKUS_HTTP_PORT: 0
    restart: "no"  # don't restart after it exits cleanly
```

Your app detects `JOB_NAME` and runs only that job then exits.

### Option 3: Kubernetes CronJob (for k8s environments)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: job
            image: quarkus/tenant-jvm:latest
            env:
            - name: JOB_NAME
              value: "daily-report"
          restartPolicy: OnFailure
```

Kubernetes starts a fresh container at the scheduled time, runs it, and tears it down. Perfect for native images (fast startup + low memory while idle = $0 cost between runs).

---

## PART 8 — Optimizing Your Dockerfiles

### 8.1 — Multi-stage build (build inside Docker — no Java needed on the CI machine)

Many teams do the Maven build INSIDE Docker so the build machine doesn't need Java installed:

```dockerfile
# ============ STAGE 1: Build ============
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build

# Copy Maven wrapper and pom first (for layer caching)
COPY mvnw .
COPY .mvn/ .mvn/
COPY pom.xml .

# Download dependencies — this layer is cached unless pom.xml changes
RUN ./mvnw dependency:go-offline -q

# Now copy source and build
COPY src/ src/
RUN ./mvnw package -DskipTests -q

# ============ STAGE 2: Runtime ============
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
EXPOSE 8080
USER 1001

# For Spring Boot 3+ (layered JAR)
COPY --from=builder /build/target/dependency/ ./dependency/
COPY --from=builder /build/target/spring-boot-loader/ ./spring-boot-loader/
COPY --from=builder /build/target/snapshot-dependencies/ ./snapshot-dependencies/
COPY --from=builder /build/target/application/ ./application/

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

The final image has NO compiler, NO Maven, NO source code — just the JRE and your app files. The builder stage is discarded.

### 8.2 — Spring Boot Layered JARs (same concept as Quarkus fast-JAR)

Spring Boot 2.3+ supports layered JARs. In your `pom.xml`:
```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <layers><enabled>true</enabled></layers>
    </configuration>
</plugin>
```

Then extract layers in Docker:
```dockerfile
RUN java -Djarmode=layertools -jar app.jar extract
```

This produces the same 4-folder structure as Quarkus fast-JAR, giving you the same layer caching benefits.

### 8.3 — `.dockerignore` (often forgotten, very impactful)

Create a `.dockerignore` file next to your Dockerfile:
```
.git/
.gitignore
README.md
target/
*.md
.idea/
*.iml
node_modules/
```

Without this, `COPY . .` sends your entire Git history and IDE files to the Docker daemon. With it, builds start faster and layers are smaller.

### 8.4 — Pin your base image versions

```dockerfile
# ❌ Bad — image changes without you changing anything
FROM registry.access.redhat.com/ubi9:latest

# ✅ Good — fully reproducible
FROM registry.access.redhat.com/ubi9/openjdk-21:1.22
```

`latest` breaks reproducibility. You already do this correctly in your JVM Dockerfile.

### 8.5 — Combine RUN commands

```dockerfile
# ❌ Bad — 3 layers, cleanup doesn't work (previous layer still holds the cache files)
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*

# ✅ Good — 1 layer, cleanup is in the same layer so it actually reduces image size
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
```

---

## PART 9 — Memory Management for Java in Containers

This is where most people get hurt. Java by default looks at the TOTAL host RAM (e.g., 32GB) and sets heap to 25% of it (8GB). But your container limit is 512MB. Result: the JVM tries to allocate 8GB, gets OOMKilled.

Modern Java (11+) is container-aware — it reads cgroup limits. But you still need to set it up correctly.

### In your JVM Dockerfile, the base image handles this via `run-java.sh`:

- `JAVA_MAX_MEM_RATIO=50` → heap = 50% of container memory limit
- `JAVA_INITIAL_MEM_RATIO=25` → initial heap = 25% of max heap

So if your container has `--memory=1g`:
- Max heap = 512MB (50%)
- Initial heap = 128MB (25% of 512)

### Explicit JVM flags (more control):

```dockerfile
ENV JAVA_OPTS="\
  -Dquarkus.http.host=0.0.0.0 \
  -Djava.util.logging.manager=org.jboss.logmanager.LogManager \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseG1GC \
  -XX:+ExitOnOutOfMemoryError"
```

- `+UseContainerSupport` — reads cgroup memory/CPU limits (default ON in Java 11+, but being explicit is clearer)
- `MaxRAMPercentage=75.0` — heap = 75% of container limit
- `+UseG1GC` — better GC for most web app workloads
- `+ExitOnOutOfMemoryError` — crash immediately on OOM instead of thrashing (lets the orchestrator restart you cleanly)

---

## PART 10 — Health Checks

Kubernetes and Docker need to know if your container is healthy.

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/q/health/live || exit 1
```

Quarkus exposes `/q/health/live` (liveness) and `/q/health/ready` (readiness) with the `quarkus-smallrye-health` extension. Spring Boot Actuator exposes `/actuator/health`.

- `--start-period=30s` — don't fail health checks during the first 30s (JVM warmup time)
- `liveness` — "is the process alive and not deadlocked?" → if fails, restart container
- `readiness` — "is it ready to serve traffic?" → if fails, remove from load balancer but don't restart

---

## PART 11 — Full Optimized Reference Dockerfile (Quarkus JVM)

```dockerfile
# ============================================================
# STAGE 1: Build (only runs in CI, not shipped in the final image)
# ============================================================
FROM registry.access.redhat.com/ubi9/openjdk-21:1.22 AS builder
WORKDIR /build

# Maven wrapper and pom (cached unless these files change)
COPY mvnw .
COPY .mvn/ .mvn/
COPY pom.xml .

# Download dependencies (cached unless pom.xml changes)
RUN ./mvnw dependency:go-offline -q

# Copy source and build
COPY src/ src/
RUN ./mvnw package -DskipTests -q

# ============================================================
# STAGE 2: Runtime image
# ============================================================
FROM registry.access.redhat.com/ubi9/openjdk-21:1.22

LABEL maintainer="yourteam@company.com"
LABEL app="tenant-service"
LABEL version="1.0.0"

ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en'

# === LAYER STRATEGY ===
# Ordered from least-changed to most-changed
# If a layer is unchanged, Docker reuses it from cache

# Layer A: 3rd party deps — only changes when pom.xml changes (~weekly)
COPY --chown=185 --from=builder /build/target/quarkus-app/lib/ /deployments/lib/

# Layer B: Quarkus launcher — rarely changes (~monthly with Quarkus upgrades)
COPY --chown=185 --from=builder /build/target/quarkus-app/*.jar /deployments/

# Layer C: Your app code — changes every commit
COPY --chown=185 --from=builder /build/target/quarkus-app/app/ /deployments/app/

# Layer D: Quarkus processed bytecode — changes when Quarkus config changes
COPY --chown=185 --from=builder /build/target/quarkus-app/quarkus/ /deployments/quarkus/

EXPOSE 8080
USER 185

HEALTHCHECK --interval=10s --timeout=3s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/q/health/live || exit 1

ENV AB_JOLOKIA_OFF=""
ENV JAVA_OPTS=" \
  -Dquarkus.http.host=0.0.0.0 \
  -Djava.util.logging.manager=org.jboss.logmanager.LogManager \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof"
ENV JAVA_APP_JAR="/deployments/quarkus-run.jar"
```

---

## PART 12 — Quick Reference: What Goes Where

```
Host machine (before docker build):
  target/quarkus-app/   ← Maven build output
  src/                  ← source code (NOT copied to prod image)
  .dockerignore         ← excludes .git, IDE files, target/ etc.

Inside the container at runtime:
  /deployments/lib/     ← all dependency JARs (Hibernate, Jackson, Kafka client...)
  /deployments/*.jar    ← quarkus-run.jar (tiny launcher)
  /deployments/app/     ← YOUR compiled code
  /deployments/quarkus/ ← Quarkus framework bytecode
  /tmp/                 ← writable temp (heap dumps etc)
  (everything else is read-only — the image layers)
```

---

## Summary Decision Tree

```
Do you need max throughput under sustained traffic?
  YES → JVM Fast-JAR (JIT warms up and beats native after ~5min of traffic)
  NO  → go to next

Do you need < 100ms startup or < 50MB memory?
  YES → Native-Micro (if you control the Linux libs) or Native (if you use ubi9-minimal)
  NO  → JVM Fast-JAR (simpler to maintain)

Are you on Spring Boot?
  YES → JVM with layered JARs + eclipse-temurin:21-jre-alpine base (smallest JRE image)
  NO (Quarkus) → use Quarkus generated Dockerfiles as starting point, add health checks + explicit JVM flags

Is it a cron job or batch task?
  YES → Native image in a Kubernetes CronJob (fast start, exits cleanly, zero idle cost)
  NO  → Long-running service with health checks
```

---

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

### 3a. Local native (GraalVM/Mandrel installed on your machine)

```bash
mvn clean package -Pnative -DskipTests "-Dquarkus.native.remote-container-build=false"
```

**Run:**
```bash
./target/your-app-1.0-runner
```

### 3b. Container native (Docker builds the native image — no GraalVM needed locally)

```bash
mvn clean package -Pnative -DskipTests "-Dquarkus.native.container-build=true"
```

Docker pulls the Mandrel builder image and runs `native-image` inside it. Your machine only needs Docker.

### 3c. Remote container native (your project's setup)

```bash
mvn clean package -Pnative -DskipTests -Dquarkus.profile=dev -T 1
```

`remote-container-build=true` is already set in the `native` profile in `pom.xml`, so it uses the company's remote builder image automatically.

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
Same as the first, but the extra flag tells Quarkus: *"don't use my local GraalVM — pull the Mandrel builder Docker image and run `native-image` inside a container."* Your machine never needs GraalVM at all.
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

> ⚠️ Since `container-build=true` compiles **inside a Linux container**, the output binary is a **Linux executable**. It won't run natively on Windows — only inside WSL or a Docker container. If you want a binary that runs on Windows, you'd need a Windows GraalVM build, which Quarkus doesn't officially support for production use.

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
