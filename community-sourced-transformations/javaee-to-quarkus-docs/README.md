# JavaEE to Quarkus — Transformation Definition

Automated migration of JavaEE/WildFly applications to Quarkus 3.33.2 (LTS) using AWS Transform.

## Quick Start

```bash
atx custom def exec -t -n javaee-to-quarkus --tv latest --configuration file:///tmp/config.json -p /path/to/app
```

## Benchmark Results

| Metric | Value |
|--------|-------|
| Apps benchmarked | 52 |
| Build success rate | 98% (51/52) |
| Test pass rate | 100% |
| HIGH complexity apps | 50% (26/52) |
| BES Composite Score | 0.8754 |
| BES Decision | READY_FOR_NEXT_STEPS |

### App Sources
- WildFly Quickstarts (41 apps)
- IBM WASdev samples (3 apps)
- javaee-samples/javaee7-samples (5 apps)
- Eclipse Cargo Tracker (1 app)
- ScarfBench DayTrader Jakarta (1 app)
- IBM CustomerOrderServices (1 app)

## Supported Patterns

| Pattern | Target |
|---------|--------|
| EJB (@Stateless, @Stateful, @Singleton) | CDI (@ApplicationScoped, @SessionScoped) |
| JMS (MDB, ConnectionFactory) | SmallRye Reactive Messaging |
| JPA (persistence.xml, Hibernate) | Quarkus Hibernate ORM |
| JSF (Facelets, @ManagedBean) | MyFaces Quarkus / Qute |
| SOAP/JAX-WS | quarkus-cxf |
| Batch (JSR 352) | JBeret / CDI fallback |
| Security (JAAS, JASPIC) | quarkus-security |
| LRA (Narayana) | quarkus-narayana-lra |
| Servlet | JAX-RS (RESTEasy Reactive) |

## Architecture

5-phase conditional pipeline with Phase 0 detection:
1. **Phase 0**: Analysis & feature flag detection (8+ flags)
2. **Phase 1**: Build system migration (WAR→JAR, Quarkus BOM)
3. **Phase 2**: Core conversions (EJB→CDI, JPA, Servlet→JAX-RS)
4. **Phase 3**: Messaging & Security (conditional)
5. **Phase 4**: JSF migration (conditional)
6. **Phase 5**: Containerization (Dockerfile, Helm, health probes)

## Known Blockers (graceful HALT)
- Remote EJB (not portable to Quarkus)
- Elytron Security Domain (requires manual redesign)

## File Structure

```
javaee-to-quarkus/
├── SKILL.md              # Main TD (287 lines)
└── references/           # 17 on-demand reference files
    ├── ejb-to-cdi-mapping.md
    ├── jms-to-smallrye.md
    ├── security-migration.md
    └── ... (14 more)
```

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025-06-01 | GA release, 52 apps benchmarked, BES approved |
| 0.9 | 2025-05-15 | Reference dispatch pattern, 17 reference files |
| 0.8 | 2025-05-01 | Initial 5-phase pipeline, 28 apps |

## Author
Sascha Moellering — Principal GTM SSA, AWS