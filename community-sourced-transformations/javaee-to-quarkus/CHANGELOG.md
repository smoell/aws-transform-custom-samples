# CHANGELOG — JavaEE to Quarkus Skill v1.6.1 (manual condensation)

## Iteration (2026-07) — Manual Condensation + Grep Hardening

Post-processing of the v1.6 auto-evolved spec (session 00e85250, iter 3). NOT an
automated TransForge iteration — a manual maintenance pass by the spec owner.

### Size Reduction (moderate condensation)
- Total spec: 231.7 KB -> 198.2 KB (-14.5%). Still above the 150 KB monitoring
  threshold; further reduction would require removing reviewer-preserved content.
- Condensed the 4 largest reference files by ~30% each, preserving ALL named
  patterns/sections — only merged redundant before/after code blocks and trimmed
  verbose prose:
  - arquillian-to-quarkustest.md  29.9 KB -> 21.2 KB (all 11 sections + both Quick Ref tables kept)
  - ejb-to-cdi-mapping.md         28.8 KB -> 18.7 KB (merged duplicate TransactionAttribute tables; all 6 examples kept)
  - jms-to-smallrye.md            22.3 KB -> 13.7 KB (merged duplicate MDB before/after; decision tree + all config kept)
  - jpa-to-quarkus-persistence.md 21.1 KB -> 13.4 KB (condensed persistence.xml example; all tables + Panache/Envers/Search kept)

### Correctness Fixes (from spec review)
- **Hardened comment-blind greps** (functional bug): EJB-annotation and JNDI exit-criteria
  / validation greps now exclude comment lines (`| grep -v '^[[:space:]]*\(//\|\*\)'`).
  Migration `MIGRATION:`/Javadoc comment lines previously produced false positives — this
  was observed in the iter-1 validator, which had to filter 6 `@EJB`/`@Stateless` and 3
  `InitialContext` comment-line hits by hand.
- **Unified javax.* scan scope** to `src/main/java/ src/test/java/` in the Validation
  Commands block (was `src/`), matching the Exit Criteria scope.

### Preserved
No named patterns, sections, or reference files removed. All 14 reference files carried
forward; 10 unchanged, 4 condensed, SKILL.md grep-hardened.

---

# CHANGELOG — JavaEE to Quarkus Skill v1.6

## Summary

This iteration fixes the only project failure (gradle-ejb-jpa-concurrency-wildfly) caused by
curl/wget sandbox prohibition and Gradle+Maven BUILD COMMAND mismatch. It also enriches reference
files from 4 successful projects (80% success rate) with patterns validated across 3+ projects.

## Changes to SKILL.md

- **ADDED**: "Shell Safety Constraints" section — curl/wget prohibition causing hard pipeline abort.
  Driven by: gradle-ejb-jpa-concurrency-wildfly failure.
- **ADDED**: Phase 0 "Complete ALL blocker scans before HALT" note — prevents incomplete BLOCKERS.md.
  Driven by: phase0-blocker-remote-ejb.
- **MODIFIED**: Phase 1 Build tool bullet — explicit Gradle-only project handling ("do NOT create pom.xml").
  Driven by: gradle-ejb-jpa-concurrency-wildfly.
- **MODIFIED**: Quick Reference "Application class removal" pattern — conditional on NON-ROOT path only.
  Resolves internal contradiction between Quick Reference and Phase 2 checklist.
  Driven by: simple-ejb-jpa-wildfly-war.
- **MODIFIED**: Phase 3 — added messaging validation grep (`@MessageDriven|javax.jms|jakarta.jms`).
- **MODIFIED**: Phase 4/5 — added `mkdir -p` prerequisite steps for directories.
- **MODIFIED**: Container Verification curl annotated as "(localhost only — environment-dependent)".
- **PRESERVED**: HttpAuthenticationMechanism Quick Reference row (review failure #5).
- **PRESERVED**: ServiceLoader SPI → CDI @StaticInitSafe tip (review failure #5).
- **PRESERVED**: Non-Goals entries for EJB security context propagation and BMT→CMT conversion (review failure #5).
- **PRESERVED**: All existing content from input spec v1.5 changes (EJB2_ENTITY_BEAN_FOUND, @Remove,
  JNDI resolvable blocker, JPA_NEEDED flag, MockitoExtension guidance, env-entry resolution).

## Changes to Reference Files

### references/ejb-to-cdi-mapping.md
- **ADDED**: "@PostConstruct → StartupEvent: Test Compatibility Pattern" subsection with full
  before/after example and 4-point rules list. Driven by: 3 projects (jms-mdb, security-stateful-jsf,
  simple-ejb-jpa).
- **ADDED**: "@Schedules (Plural) → Multiple @Scheduled Methods" subsection.
  Driven by: jms-mdb-messaging-wildfly-war.

### references/jms-to-smallrye.md
- **ADDED**: "Test Profile Configuration (In-Memory Connector)" section — mandatory for tests.
  Driven by: jms-mdb-messaging-wildfly-war.
- **ADDED**: "Message Properties / Headers Warning" section — Emitter<String> loses JMS properties.
  Driven by: jms-mdb-messaging-wildfly-war.
- **ADDED**: "Durable Subscription Mapping" table.
  Driven by: jms-mdb-messaging-wildfly-war.
- **PRESERVED**: Structured Message Types / JSON deserialization section (review failure #2).
- **PRESERVED**: @Outgoing pipeline pattern (review failure #2).
- **PRESERVED**: Detailed ack/nack code examples (review failure #2).
- **PRESERVED**: XA/Outbox pattern 3rd option (review failure #2).

### references/jpa-to-quarkus-persistence.md
- **ADDED**: "WildFly ExampleDS Default → H2 In-Memory" section.
  Driven by: jms-mdb-messaging-wildfly-war.
- **ADDED**: "Persistence Unit Name Resolution" section (unitName='primary' → plain @Inject).
  Driven by: jms-mdb-messaging-wildfly-war.
- **PRESERVED**: Entire Hibernate Envers section (review failure #1).
- **PRESERVED**: Entire Hibernate Search section (review failure #1).
- **PRESERVED**: Lazy Loading in Native Mode section (review failure #1).
- **PRESERVED**: PanacheEntity active record pattern (review failure #1).
- **PRESERVED**: All 18 property table rows including physical_naming_strategy,
  implicit_naming_strategy, order_inserts, order_updates, generate_statistics,
  default_catalog, Connection pool (review failure #1).

### references/arquillian-to-quarkustest.md
- **ADDED**: Expanded "Path configuration with root-path" note with GitHub #28001 reference
  and @TestHTTPResource pattern for /q/health.
  Driven by: jms-mdb-messaging-wildfly-war.
- **PRESERVED**: CDI Beans per Profile alternative (review failure #3).
- **PRESERVED**: Full TestContainers implementation (review failure #3).
- **PRESERVED**: All JUnit 4→5 Quick Reference assertion rows (review failure #3).
- **PRESERVED**: Detection command for assertion migration (review failure #3).
- **PRESERVED**: Continuous-testing config (review failure #3).

### references/application-properties-checklist.md
- **ADDED**: SmallRye in-memory test profile entries in SmallRye section.
  Driven by: jms-mdb-messaging-wildfly-war.
- **PRESERVED**: Micrometer section (review failure #4).
- **PRESERVED**: JTA Minimal Configuration section (review failure #4).
- **PRESERVED**: Embedded Artemis config (review failure #4).

### references/troubleshooting-pitfalls.md
- **ADDED**: "Gradle + mvn BUILD COMMAND mismatch" row in symptom table + detail section.
  Driven by: gradle-ejb-jpa-concurrency-wildfly.
- **ADDED**: "find exit-code false positive" row + section.
  Driven by: jms-mdb-messaging-wildfly-war.

### references/phases-detail.md
- **ADDED**: Phase 3 messaging validation grep (review failure #6).
- **ADDED**: `mkdir -p` prerequisite for Phase 4 Step 16 (META-INF/resources/).
- **ADDED**: `mkdir -p src/main/docker/` prerequisite for Phase 5 Step 19.

## Unchanged Reference Files (carried forward)
- references/arc-limitations.md
- references/batch-jberet-fallback.md
- references/compatibility-matrix.md
- references/ear-consolidation.md
- references/jsf-migration-patterns.md
- references/pattern-remote-ejb-limitation.md
- references/phase0-detection-flags.md
- references/quarkus-extension-catalog.md
- references/security-migration.md
- references/worked-examples-complete.md

## Removed Content
None. All content from the input spec is preserved in the output.

### Sources
- `REST Assured basePath path-doubling with root-path` -- `https://github.com/quarkusio/quarkus/issues/28001` -- "Expected behavior: http request from RestAssured is correct. Actual behavior: http request url contains two values from quarkus.http.root-path and fails with 404"
- `SmallRye in-memory connector for test isolation` -- `https://smallrye.io/smallrye-reactive-messaging/4.24.0/concepts/testing/` -- "SmallRye Reactive Messaging proposes an in-memory connector for this exact purpose"
- `Quarkus @Scheduled only one cron per annotation` -- `https://quarkus.io/guides/scheduler-reference` -- "The syntax used in CRON expressions is controlled by quarkus.scheduler.cron-type property"
