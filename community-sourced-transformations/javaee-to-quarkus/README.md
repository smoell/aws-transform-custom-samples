# JavaEE/JakartaEE to Quarkus Migration Skill

Automated migration of Java EE / Jakarta EE applications from JBoss EAP, WildFly, Payara, GlassFish, WebLogic, or TomEE to Quarkus 3.x standalone JARs using an AI agent skill with conditional pipeline architecture.

## Table of Contents

- [Overview](#overview)
- [The Problem](#the-problem)
- [What This Skill Does](#what-this-skill-does)
- [Skill Architecture](#skill-architecture)
- [Prerequisites](#prerequisites)
- [Getting Started with AWS Transform Custom](#getting-started-with-aws-transform-custom)
- [Running the Migration](#running-the-migration)
- [Example Configuration](#example-configuration)
- [Supported Source Applications](#supported-source-applications)
- [Documentation & References](#documentation--references)
- [Known Limitations](#known-limitations)
- [Contributing](#contributing)

## Overview

Automate the migration of enterprise Java applications from traditional application servers to Quarkus 3.x — a Kubernetes-native Java framework optimized for fast startup, low memory, and cloud-native deployment. This skill eliminates application server dependencies, converts EJBs to CDI beans (ArC), modernizes to uber-jar or native-image packaging, and produces containerized applications ready for cloud deployment.

## The Problem

Organizations running Java EE applications on traditional application servers face:

- **Operational overhead**: Managing and patching JBoss, WildFly, Payara, GlassFish, or WebLogic across environments
- **Slow startup times**: Application servers take 30-60+ seconds to boot; microservice architectures need sub-second startup
- **High memory footprint**: App server + application often requires 512MB-2GB+ heap; Kubernetes clusters need density
- **Vendor lock-in**: Tight coupling to EJB APIs, JNDI, proprietary XML configs, and server-specific deployment descriptors
- **Cloud-unfriendly**: WAR/EAR deployment model doesn't fit container orchestration or serverless
- **No native compilation option**: Traditional app servers cannot produce GraalVM native images
- **Manual migration is complex**: Requires deep knowledge of both EJB and CDI/Quarkus patterns; typically weeks per application

## What This Skill Does

Given a Java EE application deployed on any supported application server, this skill automatically:

1. Converts WAR/EAR packaging to standalone executable JAR (uber-jar or fast-jar)
2. Replaces EJBs (`@Stateless`, `@Stateful`, `@Singleton`, `@MessageDriven`) with CDI beans (`@ApplicationScoped` + `@Transactional`)
3. Eliminates JNDI lookups in favor of CDI `@Inject` + `@ConfigProperty`
4. Migrates `javax.*` namespaces to `jakarta.*` (when source is Java EE 8 or earlier)
5. Converts `persistence.xml` to `application.properties` (`quarkus.datasource.*`, `quarkus.hibernate-orm.*`)
6. Preserves JAX-RS endpoints unchanged (Quarkus is JAX-RS native via RESTEasy Reactive)
7. Migrates EJB timers to Quarkus `@Scheduled`
8. Converts JMS/MDB to SmallRye Reactive Messaging or preserves JMS API via `quarkus-artemis-jms`
9. Migrates security constraints to Quarkus HTTP auth policies
10. Converts Arquillian tests to `@QuarkusTest` + REST Assured
11. Adds health checks (`/q/health`) and metrics (`/q/metrics`)
12. Creates production-ready Dockerfiles (JVM and optional native)
13. Removes all application server descriptors and dependencies

## Skill Architecture

The skill uses a 6-phase conditional pipeline. Phase 0 scans the project and sets feature flags, then only the required phases execute:

```
Phase 0: Scanner ──→ Phase 1: Build & Namespace ──→ Phase 2: Core Migration
  Analyze project,     Quarkus BOM, WAR→JAR,         EJBs→CDI, JPA config,
  detect features,     javax→jakarta,                 JAX-RS adjust, JNDI→DI,
  set phase flags      quarkus-maven-plugin           timers, WebSockets
  │                    ALWAYS runs                    CONDITIONAL (EJB detected)
  │
  ├──→ Phase 3: Conditional Services
  │      Security, JMS/MDB, SOAP/CXF
  │      ONLY if SECURITY_NEEDED || JMS_NEEDED || SOAP_NEEDED
  │
  ├──→ Phase 4: Testing & UI
  │      Arquillian→@QuarkusTest (if ARQUILLIAN_TESTS)
  │      JSF→Qute (if JSF_NEEDED)
  │      Batch (if BATCH_NEEDED)
  │      ALWAYS runs, scope varies by flags
  │
  └──→ Phase 5: Deployment & Verification
         Dockerfile, health/metrics, 20-point validation
         ALWAYS runs
```

### Key Design Decisions

1. **Conditional phase execution.** Phase 0 scans and sets flags. Phases 2-4 conditionally execute steps based on detected features, preventing unnecessary changes.
2. **JAX-RS preservation.** Unlike Spring Boot migration, Quarkus is JAX-RS native — REST endpoints require minimal changes (only `@ApplicationPath` → `quarkus.rest.path` config).
3. **ArC-aware CDI migration.** All CDI conversions respect Quarkus ArC build-time restrictions (no Portable Extensions, no bean-discovery-mode=all, proxy requirements).
4. **On-demand reference dispatch.** 10 reference documents are loaded only when the agent encounters specific patterns, keeping context focused.
5. **20-point exit criteria validation.** Every transformation is validated against 20 criteria covering build, runtime, security, and deployment readiness.

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| Java | 17+ (21 recommended) | Compile Quarkus 3.x projects |
| Maven | 3.9+ | Build Maven projects |
| Docker | 20+ | Container image builds |
| Git | Any | Version control |
| Quarkus CLI | (optional) | `quarkus dev` for hot-reload development |
| GraalVM | 21+ (optional) | Only if native image target is required |

### Installing Quarkus CLI (optional)

```bash
# macOS (Homebrew)
brew install quarkusio/tap/quarkus

# Linux (SDKMAN)
sdk install quarkus

# Or use Maven wrapper directly (no CLI needed)
./mvnw quarkus:dev
```

## Getting Started with AWS Transform Custom

To set up the AWS Transform CLI, configure authentication, and run your first transformation, see the [AWS Transform Custom Getting Started Guide](https://docs.aws.amazon.com/transform/latest/userguide/custom-get-started.html).

### Publishing the Transformation

```bash
git clone https://github.com/aws-samples/aws-transform-custom-samples
cd community-sourced-transformations

atx custom def publish -n javaee-to-quarkus \
    --sd javaee-to-quarkus \
    --description "Migrates Java EE/Jakarta EE applications from JBoss, WildFly, Payara, GlassFish, or WebLogic to Quarkus 3.x. Covers EJB to CDI (ArC), JPA, JMS, JSF, security, namespace migration, and containerization."
```

## Running the Migration

### Interactive Mode (with confirmation prompts)

```bash
atx custom def exec \
  -n javaee-to-quarkus \
  -p ./my-javaee-app \
  -t \
  --configuration file://config.json
```

### Direct Mode (fully automated)

```bash
atx custom def exec \
  -n javaee-to-quarkus \
  -p ./my-javaee-app \
  -x -t \
  --configuration file://config.json
```

### Validating the Output

```bash
# Build
./mvnw clean verify

# Run
java -jar target/quarkus-app/quarkus-run.jar

# Health check
curl http://localhost:8080/q/health

# Metrics
curl http://localhost:8080/q/metrics

# Legacy import scan (should return zero)
grep -rn "import javax\." src/main/java/ | grep -v "javax\.crypto\|javax\.net\|javax\.sql\|javax\.security\.cert"

# EJB annotation scan (should return zero)
grep -rn "@Stateless\|@Stateful\|@EJB\|@MessageDriven" src/main/java/

# Dev mode (hot-reload)
./mvnw quarkus:dev
```

### Expected Output

- Standalone fast-jar in `target/quarkus-app/` (10-30 MB typical)
- JVM startup time: 1-5 seconds
- Native startup time: 0.01-0.1 seconds (if native build used)
- All REST endpoints preserved at original paths
- Health/readiness probes at `/q/health/live` and `/q/health/ready`
- Prometheus metrics at `/q/metrics`
- Dockerfiles in `src/main/docker/`

## Example Configuration

### config.json for a simple JavaEE WAR application

```json
{
  "sourceType": "javaee",
  "targetFramework": "quarkus",
  "targetVersion": "3.17",
  "sourcePath": "./my-javaee-app",
  "options": {
    "javaVersion": "17",
    "packageType": "fast-jar",
    "database": "postgresql",
    "includeDocker": true,
    "includeHealth": true,
    "includeMetrics": true,
    "nativeImage": false
  }
}
```

### config.json for a multi-module EAR with JMS and Security

```json
{
  "sourceType": "javaee",
  "targetFramework": "quarkus",
  "targetVersion": "3.17",
  "sourcePath": "./enterprise-ear-app",
  "options": {
    "javaVersion": "21",
    "packageType": "uber-jar",
    "database": "oracle",
    "messaging": "amqp",
    "security": "oidc",
    "includeDocker": true,
    "includeHealth": true,
    "includeMetrics": true,
    "nativeImage": true
  }
}
```

## Supported Source Applications

| Application Server | Versions | Support Level |
|---|---|---|
| JBoss EAP | 6.x, 7.x, 8.x | Full — primary development target |
| WildFly | 10+, 26+, 30+ | Full — same as JBoss EAP |
| Payara | 5.x, 6.x | Full — standard Java EE/Jakarta EE |
| GlassFish | 5.x, 6.x, 7.x | Full — reference implementation |
| WebLogic | 12c, 14c | Supported — common EE patterns handled |
| TomEE | 7.x, 8.x, 9.x | Supported — CDI/EJB patterns |
| Generic Java EE WAR/EAR | Java EE 7, 8; Jakarta EE 9, 10 | Supported — standard spec compliance |

### Source Application Requirements

- Maven-based build (pom.xml required)
- Java 8+ source code (output targets Java 17+)
- Standard Java EE / Jakarta EE APIs (not proprietary server extensions)

## Documentation & References

### Skill Definition

| File | Description |
|---|---|
| [SKILL.md](SKILL.md) | Complete skill definition — objective, scope, constraints, 6-phase workflow, 20-point validation criteria, worked examples |

### Reference Documents (On-Demand)

Loaded by the agent when specific patterns are encountered:

| Reference | Trigger | Description |
|---|---|---|
| [phase0-detection-flags.md](references/phase0-detection-flags.md) | Phase 0 scanning | Feature flag detection rules |
| [ejb-to-cdi-mapping.md](references/ejb-to-cdi-mapping.md) | `@Stateless`, `@EJB`, `@TransactionAttribute` | Complete EJB→CDI annotation mapping with code examples |
| [arc-limitations.md](references/arc-limitations.md) | ArC proxy errors, CDI issues | Build-time CDI restrictions, workarounds, config |
| [jpa-to-quarkus-persistence.md](references/jpa-to-quarkus-persistence.md) | `persistence.xml`, `EntityManager` | JPA config migration, multi-datasource, Panache |
| [jms-to-smallrye.md](references/jms-to-smallrye.md) | `@MessageDriven`, JMS | JMS→SmallRye Reactive Messaging patterns |
| [security-to-quarkus-security.md](references/security-to-quarkus-security.md) | `@RolesAllowed`, security constraints | Security migration: JAAS→IdentityProvider, web.xml→config |
| [jsf-to-qute.md](references/jsf-to-qute.md) | `.xhtml`, `@Named`, JSF | JSF→Qute template migration |
| [arquillian-to-quarkustest.md](references/arquillian-to-quarkustest.md) | `@RunWith(Arquillian`, tests | Arquillian→@QuarkusTest + REST Assured |
| [quarkus-extension-catalog.md](references/quarkus-extension-catalog.md) | Extension selection | JavaEE spec→Quarkus extension mapping table |
| [worked-examples-conditional.md](references/worked-examples-conditional.md) | MDB, Security, SOAP, JSF examples | Complete before/after code for conditional phases |

## Known Limitations

These are documented in the **Non-Goals** section of [SKILL.md](SKILL.md) and require separate handling:

| Limitation | Severity | Notes |
|---|---|---|
| EJB Remote Interfaces (IIOP/CORBA) | HIGH | Must redesign to REST or gRPC |
| JCA Resource Adapters | MEDIUM | Replace with vendor client libraries |
| EJB 2.x Entity Beans (CMP/BMP) | HIGH | Manually convert to JPA entities first |
| CDI Portable Extensions | MEDIUM | Convert to Quarkus `@BuildStep` (see `references/arc-limitations.md`) |
| XA Distributed Transactions | MEDIUM | Use idempotent consumer or outbox pattern |
| Vaadin UI | LOW | Requires separate Vaadin-specific migration |
| Proprietary app server clustering (JGroups) | MEDIUM | Replace with Kubernetes-native scaling |
| Applications >50K LOC | LOW | May exceed context limits; consider phased migration |

## Contributing

### Adding Benchmark Applications

1. Fork the repository
2. Add a Java EE test application in a new directory
3. Document the application's features (EJB types, security, JMS, etc.)
4. Run the transformation and record results
5. Submit a PR with the benchmark results

### Improving Reference Documents

Reference documents in `references/` encode patterns discovered during migration. To contribute:

1. Identify a migration pattern not covered (or an edge case that failed)
2. Document the before/after code pattern with explanation
3. Add to the appropriate reference file (or create a new one)
4. Update the Reference Dispatch table in `SKILL.md` if a new file is added

### Testing Changes

After modifying `SKILL.md` or reference files:

```bash
# Verify the skill definition is valid
atx custom def validate --sd javaee-to-quarkus

# Run against a benchmark app to verify changes
atx custom def exec -n javaee-to-quarkus -p ./benchmark-apps/simple-ejb-app -x -t
```

## Repository Structure

```
javaee-to-quarkus/
├── README.md                          # This file
├── SKILL.md                           # Skill definition (6-phase conditional pipeline)
├── references/                        # On-demand reference documents (10 files)
│   ├── phase0-detection-flags.md      # Feature flag detection rules
│   ├── ejb-to-cdi-mapping.md         # EJB→CDI mapping with code examples
│   ├── arc-limitations.md            # ArC build-time CDI limitations
│   ├── jpa-to-quarkus-persistence.md # JPA config & Panache patterns
│   ├── jms-to-smallrye.md           # JMS/MDB→Reactive Messaging
│   ├── security-to-quarkus-security.md # Security migration
│   ├── jsf-to-qute.md               # JSF→Qute template migration
│   ├── arquillian-to-quarkustest.md  # Test framework migration
│   ├── quarkus-extension-catalog.md  # Extension selection catalog
│   └── worked-examples-conditional.md # Complete worked examples
```
