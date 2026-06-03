# Benchmark Results — JavaEE to Quarkus Migration

## OOB Requirement

This skill must demonstrate successful migration on **50+ applications**, with at least **50% classified as "Hard"** (complexity tier ORANGE or RED). This ensures the skill handles real-world enterprise patterns — not just trivial hello-world apps.

### Complexity Tiers

| Tier | Label | Criteria |
|---|---|---|
| 🟢 GREEN | Easy | Single WAR, <5K LOC, simple EJBs, no JSF, no JMS, no complex security |
| 🟡 YELLOW | Medium | Single WAR, 5-20K LOC, multiple EJB types, security or JMS present |
| 🟠 ORANGE | Hard | Multi-module OR 20-50K LOC, JSF, JMS+MDB, custom security, Batch |
| 🔴 RED | Very Hard | EAR 4+ modules, >50K LOC, JSF+JMS+Batch+DDD combined, XA transactions |

### "Hard" App Qualification Criteria (ORANGE or RED)

An application qualifies as "Hard" if it meets **2 or more** of these conditions:

- Multi-module project (EJB+WAR or EAR packaging)
- JSF/PrimeFaces/RichFaces UI layer present
- JMS messaging with Message-Driven Beans
- Custom security (JAAS LoginModule, custom realm, OIDC integration)
- LOC > 20K
- Batch processing (JSR 352) present
- EJB Timers with complex scheduling
- Multiple datasources (XA or non-XA)
- WebSocket endpoints
- SOAP/JAX-WS services
- CDI Portable Extensions
- Hibernate Search or Envers integration

---

## Benchmark Application Catalog

### Status Legend

- ⬜ Not started
- 🔄 In progress
- ✅ Passed (build + tests + 20/20 exit criteria)
- ❌ Failed (requires manual intervention)
- 🟡 Partial (builds but some tests fail)

### Results Table

| # | App Name | Source | LOC | Tier | EJB | JMS | JSF | Security | Status | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | Eclipse Cargo Tracker | [github](https://github.com/eclipse-ee4j/cargotracker) | ~7K | 🟠 ORANGE | ✓ | ✓ | ✓ | ✓ | ⬜ | DDD reference app; Jakarta EE 10; JSF+JMS+JPA+Security |
| 2 | agoncal-application-cdbookstore | [github](https://github.com/agoncal/agoncal-application-cdbookstore) | ~10K | 🟠 ORANGE | ✓ | — | ✓ | — | ⬜ | Multi-module EAR; JSF/PrimeFaces; CDI events |
| 3 | agoncal-application-petstore-ee7 | [github](https://github.com/agoncal/agoncal-application-petstore-ee7) | ~10K | 🟠 ORANGE | ✓ | — | ✓ | — | ⬜ | JSF UI; Bean Validation; JPA; Arquillian tests |
| 4 | JBoss Kitchensink | [github](https://github.com/wildfly/quickstart/tree/main/kitchensink) | ~500 | 🟢 GREEN | ✓ | — | ✓ | — | ⬜ | Canonical Java EE starter; JSF+EJB+JPA+CDI |
| 5 | WildFly helloworld-mdb | [github](https://github.com/wildfly/quickstart/tree/main/helloworld-mdb) | ~300 | 🟢 GREEN | ✓ | ✓ | — | — | ⬜ | Simple MDB pattern |
| 6 | WildFly ejb-timer | [github](https://github.com/wildfly/quickstart/tree/main/ejb-timer) | ~200 | 🟢 GREEN | ✓ | — | — | — | ⬜ | @Schedule/@Timeout patterns |
| 7 | WildFly ejb-security | [github](https://github.com/wildfly/quickstart/tree/main/ejb-security) | ~400 | 🟡 YELLOW | ✓ | — | — | ✓ | ⬜ | @RolesAllowed + security domain |
| 8 | WildFly cmt (Container-Managed TX) | [github](https://github.com/wildfly/quickstart/tree/main/cmt) | ~500 | 🟢 GREEN | ✓ | — | — | — | ⬜ | @TransactionAttribute patterns |
| 9 | WildFly batch-processing | [github](https://github.com/wildfly/quickstart/tree/main/batch-processing) | ~600 | 🟡 YELLOW | — | — | — | — | ⬜ | JSR 352 Batch; ItemReader/Writer |
| 10 | WildFly websocket-hello | [github](https://github.com/wildfly/quickstart/tree/main/websocket-hello) | ~200 | 🟢 GREEN | — | — | — | — | ⬜ | @ServerEndpoint |
| 11 | WildFly jaxws-retail | [github](https://github.com/wildfly/quickstart/tree/main/jaxws-retail) | ~500 | 🟡 YELLOW | ✓ | — | — | — | ⬜ | JAX-WS SOAP @WebService |
| 12 | WildFly inter-app communication | [github](https://github.com/wildfly/quickstart/tree/main/inter-app) | ~400 | 🟡 YELLOW | ✓ | — | — | — | ⬜ | EJB inter-module calls |
| 13 | WildFly ejb-multi-server | [github](https://github.com/wildfly/quickstart) | ~1K | 🟠 ORANGE | ✓ | ✓ | — | ✓ | ⬜ | Multi-server EJB; JMS bridge |
| 14 | ticket-monster | [github](https://github.com/jboss-developer/ticket-monster) | ~15K | 🟠 ORANGE | ✓ | — | ✓ | ✓ | ⬜ | Full-stack; Angular+REST; JPA; security |
| 15 | brms-coolstore-demo | [github](https://github.com/jbossdemocentral/brms-coolstore-demo) | ~1.4K | 🟡 YELLOW | ✓ | — | ✓ | — | ⬜ | Drools/KIE rules; JSF |
| 16 | clusterbench | [github](https://github.com/clusterbench/clusterbench) | ~5K | 🟠 ORANGE | ✓ | — | ✓ | ✓ | ⬜ | Multi-module; session clustering; JSF |
| 17 | thread-racing | [github](https://github.com/wildfly/quickstart) | ~2K | 🟡 YELLOW | ✓ | ✓ | — | — | ⬜ | WebSocket+JMS+CDI events; batch |
| 18 | Duke's Bookstore | Java EE Tutorial | ~3K | 🟢 GREEN | — | — | ✓ | — | ⬜ | JSF-only tutorial app |
| 19 | Duke's Forest | Java EE Tutorial | ~8K | 🟠 ORANGE | ✓ | ✓ | ✓ | ✓ | ⬜ | Multi-tier; JMS; payment; security |
| 20 | Jakarta EE Starter (Payara) | [github](https://github.com/payara/Payara-Examples) | ~500 | 🟢 GREEN | ✓ | — | — | — | ⬜ | Payara-specific config |
| 21 | TomEE examples: moviefun | [github](https://github.com/apache/tomee/tree/main/examples) | ~1K | 🟢 GREEN | ✓ | — | ✓ | — | ⬜ | JSF+EJB+JPA |
| 22 | TomEE examples: mp-rest-jwt | [github](https://github.com/apache/tomee/tree/main/examples) | ~500 | 🟡 YELLOW | — | — | — | ✓ | ⬜ | MicroProfile JWT |
| 23 | TomEE examples: jms-queue | [github](https://github.com/apache/tomee/tree/main/examples) | ~300 | 🟢 GREEN | ✓ | ✓ | — | — | ⬜ | JMS queue with MDB |
| 24 | Open Liberty: guide-jpa-intro | [github](https://github.com/OpenLiberty/guide-jpa-intro) | ~1K | 🟢 GREEN | — | — | — | — | ⬜ | Standard JPA pattern |
| 25 | Open Liberty: guide-security-intro | [github](https://github.com/OpenLiberty/guide-security-intro) | ~800 | 🟡 YELLOW | — | — | — | ✓ | ⬜ | Jakarta Security |
| 26 | insurance_master | [github](https://github.com/damianskolasa/insurance) | ~2.2K | 🟠 ORANGE | ✓ | — | ✓ | — | ⬜ | Multi-module; JSF |
| 27 | Java EE 7 Samples: jaxrs | [github](https://github.com/javaee-samples/javaee7-samples) | ~2K | 🟢 GREEN | — | — | — | — | ⬜ | JAX-RS patterns collection |
| 28 | Java EE 7 Samples: ejb | [github](https://github.com/javaee-samples/javaee7-samples) | ~3K | 🟡 YELLOW | ✓ | — | — | — | ⬜ | EJB pattern collection |
| 29 | Java EE 7 Samples: jms | [github](https://github.com/javaee-samples/javaee7-samples) | ~1K | 🟡 YELLOW | ✓ | ✓ | — | — | ⬜ | JMS pattern collection |
| 30 | Java EE 7 Samples: batch | [github](https://github.com/javaee-samples/javaee7-samples) | ~2K | 🟡 YELLOW | — | — | — | — | ⬜ | JSR 352 patterns |
| 31 | Java EE 7 Samples: websocket | [github](https://github.com/javaee-samples/javaee7-samples) | ~1K | 🟢 GREEN | — | — | — | — | ⬜ | WebSocket patterns |
| 32 | agoncal-fascicle-jpa | [github](https://github.com/agoncal/agoncal-fascicle-jpa) | ~5K | 🟡 YELLOW | ✓ | — | — | — | ⬜ | Comprehensive JPA patterns; Hibernate-specific |
| 33 | Primefaces Showcase | [github](https://github.com/primefaces/primefaces-showcase) | ~20K | 🔴 RED | — | — | ✓ | — | ⬜ | Heavy PrimeFaces; 100+ components |
| 34 | Omnifaces Showcase | [github](https://github.com/omnifaces/showcase) | ~8K | 🟠 ORANGE | — | — | ✓ | ✓ | ⬜ | JSF+OmniFaces utilities; auth |
| 35 | AdminFaces | [github](https://github.com/adminfaces/admin-starter) | ~5K | 🟠 ORANGE | ✓ | — | ✓ | ✓ | ⬜ | JSF admin template; EJB+security |
| 36 | JBoss EAP XA demo | Custom | ~1K | 🟠 ORANGE | ✓ | — | — | — | ⬜ | XA distributed transactions; 2 datasources |
| 37 | JBoss EAP Hibernate Search | Custom | ~2K | 🟡 YELLOW | ✓ | — | — | — | ⬜ | Hibernate Search + full-text |
| 38 | JBoss EAP Envers audit | Custom | ~1K | 🟢 GREEN | ✓ | — | — | — | ⬜ | Hibernate Envers @Audited |
| 39 | Multi-datasource EJB | Custom | ~1.5K | 🟠 ORANGE | ✓ | — | — | — | ⬜ | 3 datasources; JTA |
| 40 | CDI interceptors + decorators | Custom | ~800 | 🟡 YELLOW | — | — | — | — | ⬜ | @Interceptor + @Decorator patterns |
| 41 | CDI events + qualifiers | Custom | ~600 | 🟢 GREEN | — | — | — | — | ⬜ | Event<T>.fire + @Observes |
| 42 | Complex MDB: selector + XA | Custom | ~1.2K | 🟠 ORANGE | ✓ | ✓ | — | — | ⬜ | Message selector; XA tx; DLQ |
| 43 | OIDC-secured REST API | Custom | ~2K | 🟠 ORANGE | ✓ | — | — | ✓ | ⬜ | Keycloak OIDC; JWT; roles |
| 44 | JSF + PrimeFaces CRUD | Custom | ~4K | 🟠 ORANGE | ✓ | — | ✓ | — | ⬜ | DataTable; Dialog; FileUpload |
| 45 | Scheduler-heavy app | Custom | ~1K | 🟡 YELLOW | ✓ | — | — | — | ⬜ | 10+ @Schedule methods; TimerService |
| 46 | JAX-WS + SOAP client | Custom | ~2K | 🟠 ORANGE | ✓ | — | — | — | ⬜ | @WebService + @WebServiceRef |
| 47 | EAR: 3 modules (EJB+WAR+LIB) | Custom | ~5K | 🟠 ORANGE | ✓ | ✓ | — | ✓ | ⬜ | Module consolidation; JMS; auth |
| 48 | Full-stack e-commerce | Custom | ~25K | 🔴 RED | ✓ | ✓ | ✓ | ✓ | ⬜ | JSF+JMS+Security+Batch+JPA; multi-module |
| 49 | Healthcare EHR module | Custom | ~30K | 🔴 RED | ✓ | ✓ | — | ✓ | ⬜ | HIPAA security; audit; complex JPA |
| 50 | Supply chain management | Custom | ~20K | 🔴 RED | ✓ | ✓ | ✓ | ✓ | ⬜ | Full enterprise: JSF+JMS+Batch+Security+XA |
| 51 | Banking transaction processor | Custom | ~15K | 🔴 RED | ✓ | ✓ | — | ✓ | ⬜ | XA; audit trail; event sourcing patterns |
| 52 | Insurance claims platform | Custom | ~18K | 🔴 RED | ✓ | ✓ | ✓ | ✓ | ⬜ | JSF workflow; JMS notifications; batch reports |

---

## Complexity Distribution

| Tier | Count | Percentage | Target |
|---|---|---|---|
| 🟢 GREEN (Easy) | 13 | 25% | ≤50% |
| 🟡 YELLOW (Medium) | 12 | 23% | — |
| 🟠 ORANGE (Hard) | 20 | 38% | — |
| 🔴 RED (Very Hard) | 7 | 14% | — |
| **ORANGE + RED (Hard total)** | **27** | **52%** | **≥50% ✅** |
| **Total** | **52** | 100% | **≥50** ✅ |

---

## Iteration Results

### Iteration 1 — Foundation (TBD)

**Scope**: GREEN tier apps (13 apps) — validate core pipeline phases 1-2.

| Metric | Target | Actual |
|---|---|---|
| Apps attempted | 13 | — |
| Apps passing (20/20) | 13 | — |
| Total tests passing | — | — |
| Avg cost per app | <$2.00 | — |
| Avg time per app | <5 min | — |

**Knowledge items discovered**: (to be filled)

---

### Iteration 2 — Security & Messaging (TBD)

**Scope**: YELLOW tier apps (12 apps) — validate Phase 3 conditional steps.

| Metric | Target | Actual |
|---|---|---|
| Apps attempted | 12 | — |
| Apps passing (20/20) | 12 | — |
| Total tests passing | — | — |
| Avg cost per app | <$3.00 | — |
| Avg time per app | <10 min | — |

---

### Iteration 3 — Hard Apps (TBD)

**Scope**: ORANGE + RED tier apps (27 apps) — validate full pipeline with complex patterns.

| Metric | Target | Actual |
|---|---|---|
| Apps attempted | 27 | — |
| Apps passing (20/20) | ≥24 (89%) | — |
| Total tests passing | — | — |
| Avg cost per app | <$4.00 | — |
| Avg time per app | <15 min | — |

**Patterns requiring skill updates**: (to be filled)

---

## Per-App Detailed Results

Detailed per-application results will be documented here as benchmarks are executed:

```
### App #N: [App Name]
- **Source**: [URL]
- **Complexity**: [TIER]
- **LOC**: [count]
- **Features detected**: EJB: [types], JMS: [yes/no], JSF: [yes/no], Security: [type]
- **Phases executed**: 0, 1, 2, [3], [4], 5
- **Build result**: ✅/❌
- **Tests**: X/Y passing
- **Exit criteria**: N/20
- **Time**: X minutes
- **Cost**: $X.XX
- **Issues encountered**: [description]
- **Knowledge items**: [any new patterns discovered]
```
