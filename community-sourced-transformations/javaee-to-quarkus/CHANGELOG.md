# CHANGELOG — javaee-to-quarkus v2.4 (manual condensation of v2.3)

## Iteration (2026-07) — Manual Condensation

Post-processing of the v2.3 auto-evolved spec (session 00e85250, iter 10). NOT an automated
TransForge iteration — a manual maintenance pass by the spec owner.

### Size Reduction
- Total: 247.9 KB -> 191.2 KB (-22.9%). Approaches the 150 KB monitoring threshold without
  removing any reviewer-preserved technical content.
- Condensed 5 large files + SKILL.md (all named patterns/sections/rules preserved; only merged
  duplicate before/after blocks, collapsed near-identical tables, trimmed verbose prose):
  - SKILL.md                       37.2 KB -> 26.9 KB  (deduped .bak sweep, merged grep blocks)
  - references/arc-limitations.md  19.2 KB ->  7.4 KB  (tightened per-item prose; all patterns + full config table kept)
  - references/arquillian-*.md     28.3 KB -> 19.9 KB  (all 25 sections + both quick-ref tables kept)
  - references/jpa-*.md            25.1 KB -> 12.8 KB  (merged duplicate pool/PU-resolution/dialect blocks; Property Conflict Resolution + Dialect Rules kept)
  - references/phases-detail.md    20.8 KB -> 13.9 KB  (deduped .bak sweep 3x, Gradle setup 2x, descriptor list 2x)
  - references/jms-to-smallrye.md  22.3 KB -> 13.7 KB  (merged duplicate MDB before/after)

### Correctness
- Retained v2.3's CORRECT colon-aware grep filters (`| grep -v '^[^:]*:[0-9]*:\s*//' | grep -v '^[^:]*:[0-9]*:\s*\*'`)
  everywhere — these account for grep -rn's `file:lineno:` prefix. (Supersedes the broken
  `^[[:space:]]*` filter briefly shipped as manual v1.6.1.)
- All v2.3 auto-evolved improvements (JPA property-conflict rules, batch @Transactional placement,
  strict-stubbing decision table, H2 test profile, Gradle companion steps, Javadoc cleanup) preserved.

### Preserved
No named patterns, sections, rules, or reference files removed. All 18 reference files carried
forward (13 unchanged from iter 10, 5 condensed) + SKILL.md condensed.

---

# CHANGELOG — javaee-to-quarkus v2.3

## Summary

This is a REFINE iteration fixing 3 spec bugs (grep filter, assertion wording, strict-stubbing gaps),
adding 1 proactive step (Javadoc cleanup), and addressing judge recommendations. All 24 projects
pass (100% success rate, no regressions). Changes are surgical — spec stays at 381 lines (was 379).

## Changes in SKILL.md

### High Priority (5+ projects affected)

1. **FIX: javax.* grep filter pattern** (9/24 projects)
   - Exit Criteria: Replaced broken `| grep -v '^\s*//'` and `| grep -v '^\s*\*'` with
     colon-aware `| grep -v '^[^:]*:[0-9]*:\s*//'` and `| grep -v '^[^:]*:[0-9]*:\s*\*'`
   - Same fix applied in Validation Commands section
   - Root cause: grep -rn output prefixes `filename:lineno:` so bare `^\s*` never matched
   - Added `javax\.xml\|javax\.annotation\|javax\.management` to exclusion list (judge recommendation)

2. **ADD: Phase 1 Javadoc javax.* cleanup step** (8/24 projects)
   - Added inline instruction to Phase 1 namespace migration bullet: after swapping imports,
     also scan Javadoc blocks and inline comments for literal javax.* strings — replace with prose
   - Extended Tips item about MIGRATION comments to also cover pre-existing Javadoc

3. **REWRITE: Phase 4 assertion arg reorder rule** (5/24 projects)
   - Removed misleading "3-arg overloads ONLY" and "2-arg exemption" language
   - New wording: "Reorder required whenever first argument is a String message, regardless of
     total arg count. Exemption: calls with NO message parameter."
   - Added `assertNotEquals` to the enumerated affected-forms list

4. **EXPAND: Phase 4 strict stubbing audit** (5/24 projects)
   - Added 3 sub-rules: (a) retry-loop stubs ARE reachable; (b) inline mock() calls tracked
     equally; (c) void stub refinement — delete doNothing() ONLY when method is NEVER called
   - Refined from "delete doNothing() unconditionally" to conditional rule

5. **ADD: Phase 5 mandatory pre-validation .bak gate** (5/24 projects)
   - Added as FIRST bullet of Phase 5: `find ... -delete`
   - Per-phase reminders kept intact as defense-in-depth

### Medium Priority (2-3 projects affected)

6. **QUALIFY: Phase 2 @PostConstruct removal scope** (2 projects + judge)
   - Reworded to explicitly state: "@PostConstruct removal applies ONLY to @Singleton+@Startup
     pattern. For all other bean types, RETAIN and migrate to jakarta.annotation.PostConstruct."
   - Also updated Common Patterns section for consistency

7. **FIX: H2 test profile completeness** (2 projects)
   - Phase 4 "Build file companion steps" now includes full property set:
     `%test.quarkus.datasource.jdbc.url`, `username=sa`, `password=`
   - Added note: "Required when main config has a non-H2 jdbc.url"

8. **RENAME + EXTEND: "pom.xml companion steps" → "Build file companion steps"** (2 projects)
   - Renamed label to be build-tool-neutral
   - Added Gradle equivalents: `testImplementation` declarations with BOM management

9. **FIX: ELYTRON_SECURITY_DOMAIN find command** (judge recommendation)
   - Changed from `find . -name 'jboss-ejb3.xml' -o -name 'jboss-app.xml' | xargs...`
     to `find . \( -name 'jboss-ejb3.xml' -o -name 'jboss-app.xml' \) | xargs...`
   - Escaped parentheses fix the operator precedence bug

## Reference Files Updated

### `references/jpa-to-quarkus-persistence.md`
- **Added**: "Property Conflict Resolution" section with 3 rules:
  - Rule 1: JPA standard property wins over Hibernate vendor property
  - Rule 2: Do NOT set dialect when db-kind is configured
  - Rule 3: Drop hibernate.order_inserts/order_updates (no Quarkus equivalent)
- **Removed**: `hibernate.order_inserts` and `hibernate.order_updates` rows from Property
  Mapping Table (previously mapped to non-existent `quarkus.hibernate-orm.order-inserts/updates`)
- **Added**: `javax.persistence.schema-generation.database.action` row to mapping table
- Source for drop: https://github.com/quarkusio/quarkus/issues/19129

### `references/batch-jberet-fallback.md`
- **Fixed**: @Transactional placement — moved from `processChunk()` (self-invocation, bypasses
  CDI proxy) to `readItems()` and `writeItems()` methods on separate beans (cross-bean calls)
- **Added**: "CDI Self-Invocation Warning" callout explaining proxy bypass
- **Restructured**: Component implementations now show @Transactional on reader/writer beans

## Reference Files Unchanged (carried forward)

- `references/application-properties-checklist.md`
- `references/arc-limitations.md`
- `references/arquillian-to-quarkustest.md`
- `references/compatibility-matrix.md`
- `references/ear-consolidation.md`
- `references/ejb-to-cdi-mapping.md`
- `references/jms-to-smallrye.md`
- `references/jsf-migration-patterns.md`
- `references/pattern-remote-ejb-limitation.md`
- `references/phase0-detection-flags.md`
- `references/phases-detail.md`
- `references/quarkus-extension-catalog.md`
- `references/security-migration.md`
- `references/troubleshooting-pitfalls.md`
- `references/worked-examples-complete.md`

## Removed Content

None. This is a refine iteration — all content preserved.

## Projects Driving Changes

| Change | Driven By |
|---|---|
| Grep filter fix | phase0-noop-already-quarkus, quarkus-config-traps-micro, simple-ejb-jpa-wildfly-war, soap-batch-payara-war, wildfly-fqn-annotation-jndi-micro, wildfly-jms-reactive-async-war, wildfly-lra-booking-war, wildfly-mockito-uni-strictmode-micro, wildfly-observability-spi-war |
| Javadoc cleanup | jms-mdb-messaging-wildfly-war, junit5-assertion-safety-micro, simple-ejb-jpa-wildfly-war, soap-batch-payara-war, wildfly-jms-reactive-async-war, wildfly-lra-booking-war, wildfly-mockito-uni-strictmode-micro, wildfly-observability-spi-war |
| Assertion reorder | descriptor-config-properties-micro, gradle-ear-multimodule-wildfly, payara-stateful-security-jpa-war, simple-ejb-jpa-wildfly-war, wildfly-lra-booking-war |
| Strict stubbing | junit4-migration-pitfalls, junit5-assertion-safety-micro, maven-concurrency-panache-wildfly, wildfly-jms-reactive-async-war, wildfly-mockito-uni-strictmode-micro |
| .bak gate | jms-mdb-messaging-wildfly-war, junit5-assertion-safety-micro, maven-concurrency-panache-wildfly, simple-ejb-jpa-wildfly-war, soap-batch-payara-war |
| @PostConstruct scope | payara-qute-async-emitter-war, security-stateful-jsf-wildfly-war |
| JPA property conflicts | wildfly-fqn-annotation-jndi-micro, wildfly-jms-reactive-async-war, wildfly-lra-booking-war |
| H2 test profile | payara-qute-async-emitter-war, payara-stateful-security-jpa-war |
| Build file companion | gradle-ear-multimodule-wildfly, gradle-ejb-jpa-concurrency-wildfly |

### Sources

- `hibernate.order_inserts/order_updates no Quarkus equivalent` -- `https://github.com/quarkusio/quarkus/issues/19129` -- "order_updates is enforced by Quarkus; other properties are mapped from general Quarkus properties"
