# Feature Flag Implementation Plan — TIP / TipQA (LaunchDarkly)

> **Status:** Draft plan for review
> **Date:** 2026-06-09
> **Author:** Generated with Claude Code (synthesized from the Deltek "Feature Flags Service Implementation" doc, the actual TIP codebase at `C:\TIP\TIP\code`, and LaunchDarkly official docs)
> **Scope:** High-level architecture, best practices, and a phased rollout plan for introducing LaunchDarkly feature flags into the TIP/TipQA Java (Spring MVC 6) backend and Angular 20 frontend.

---

## 1. Executive Summary

TIP/TipQA is a three-tier QMS: an **Angular 20 standalone-component SPA** talking over REST to a **Spring MVC 6.1.10 / Java 16 WAR** on Tomcat 10.1, backed by Oracle/SQL Server. Introducing feature flags lets us:

- **Decouple deploy from release** — ship code dark, turn it on per tenant/business-unit when ready.
- **Roll out safely** — gradual rollout, instant kill-switch on regressions (the "two testing gates" model in the Deltek doc: flags OFF until prod, ON only during Dev + QA gates).
- **Target by tenant / business unit / version** — critical for a multi-tenant QMS where not every tenant or version can support a new route.

**Recommended approach (the short version):**

| Layer | Mechanism |
|---|---|
| **Backend (Java)** | LaunchDarkly **Java Server SDK 7.14.0** as a **single long-lived `LDClient` Spring `@Bean`**, wrapped by a central `FeatureFlagManager` → `LaunchDarklyFeatureFlagManager` (matches the Deltek doc's prescribed class names). |
| **Frontend (Angular)** | LaunchDarkly **JS client SDK** (`@launchdarkly/js-client-sdk` v4.x) behind an injectable `LaunchDarklyService`, initialized in `APP_INITIALIZER`. UI evaluates flags **directly** with a public **client-side ID** (LD does not charge UI eval the same way — see §6). |
| **Evaluation model** | **In-process local evaluation** in each tier (the SDK keeps flags in memory via a streaming connection). Do **not** build a network "flag service" hop for evaluation — see §7. |
| **Context** | **Multi-context**: `user` + `organization` (tenant/business-unit) + `application` (version). |
| **Governance** | Deltek naming (`com.deltek.TFS123.feature`), central flag registry, archive-don't-delete lifecycle. |

This aligns the Deltek SST standards (naming, lifecycle, central manager class, OCI Vault secrets, frontend-SDK for UI) with LaunchDarkly's own architectural guidance.

---

## 2. Current-State Findings (verified against `C:\TIP\TIP\code`)

### 2.1 Backend (`server/`)

| Aspect | Finding | Relevance |
|---|---|---|
| Modules | Maven multi-module: `TipQADataModel` (jar), `TipQAServices` (jar), `TipQAWeb` (war) | Add SDK dependency to **`TipQAServices/pom.xml`** (business layer) |
| Java / Spring | Java **16**, Spring **6.1.10**, Spring Security **6.3.1**, Hibernate 6.5.2, Jakarta EE 10 | LD SDK 7.x supports Java 8+ ✅ (note Java 16 is non-LTS — see §10 risks) |
| DI style | **Hybrid** XML (`customContext.xml`, `tipQADataModelContext.xml`, `TipQA-servlet.xml`) **+ annotations** (`@Service @Scope("singleton")`, `@Autowired`). `@Configuration`/`@Bean` already used (`SchedulerConfiguration.java`) | Use a `@Configuration` class with a `@Bean` `LDClient` — pattern already exists in the codebase |
| External REST clients | `D365RestService` (OAuth2 client-credentials via `RestTemplate`), `DeltekInterfaceRestService` (SAML/Basic). Config via DTOs / properties | Closest analog to "init a client from externalized config + secret" |
| Secrets / config | `PropertyPlaceholderConfigurer` loads `system.properties`; values use **`${ENV_VAR:default}`** substitution (see `dela.properties`: `${DELA_OPENAI_API_KEY:}`). JNDI for datasource. Deltek doc mandates **OCI Vault** + K8s secret + token caching | Source the LD SDK key via `${LAUNCHDARKLY_SDK_KEY:}` → env var → OCI Vault |
| App lifecycle | `TipServletContextListener implements ServletContextListener` (`contextInitialized`/`contextDestroyed`) registered in `web.xml`; Spring `ContextLoaderListener` loads root context | `LDClient` bean auto-inits on startup; close it on shutdown |
| Service conventions | `@Service @Scope("singleton")`, heavy `@Autowired`, methods take `*RequestDTO` → return `ResponseDTO`, `@Transactional`. `UserInfo` bean carries user/BU context | Inject `FeatureFlagManager`; evaluate flag at method entry, branch logic |

### 2.2 Frontend (`client/`)

| Aspect | Finding | Relevance |
|---|---|---|
| Angular | **20.1.7**, standalone components, `bootstrapApplication(AppComponent, appConfig)`, lazy routes | Modern; supports latest JS SDK ✅ |
| Config | `src/environments/environment.ts` / `environment.prod.ts` (no `config.js` for env vars) | Add `launchDarklyClientSideId` per environment |
| Bootstrap | `app.config.ts` uses **`APP_INITIALIZER`** — already fetches the user/BU from server before routing (`initializeApp`) | **Ideal hook**: init LD (a 2nd `APP_INITIALIZER`) after user context is loaded |
| Data layer | `RestService` (URL builder), `AngularDataService` (`doGet/doPut<T>` deferred Observables), `InterceptorService` (`HTTP_INTERCEPTORS`, `withCredentials`) | New `LaunchDarklyService` as `@Injectable({providedIn:'root'})` |
| Guards | Functional guards `AuthGuard`, `ModuleAccessGuard` (`moduleAccessMap` of permissions); applied via `canActivate`/`canActivateChild` | Add a `FeatureFlagGuard` mirroring `ModuleAccessGuard`; gate routes via `data: { featureFlag: '...' }` |
| User/tenant context | `UserService` (`loginId`, `businessUnit`), `User` model (`masterBusinessUnit`, `consolidatedSecurityGroup`, `loginSessionID`), stored in `AngularDataService.setUser()` | Maps cleanly to LD context attributes (see §5) |

**Neither tier has any LaunchDarkly SDK installed yet** — greenfield integration.

---

## 3. Target Architecture

```
                       LaunchDarkly SaaS (flag rules, targeting, dashboard)
                          ▲  (streaming, in-memory eval)        ▲
        server SDK key    │                                     │  client-side ID (public)
        (SECRET, OCI Vault)│                                    │  + secureModeHash
                          │                                     │
     ┌────────────────────┴───────────────┐        ┌────────────┴───────────────────┐
     │  Java Backend (TipQAServices)       │        │  Angular SPA (client)           │
     │                                     │        │                                 │
     │  @Bean LDClient  (singleton)        │        │  LaunchDarklyService (root)     │
     │      ▲                              │        │      ▲  init in APP_INITIALIZER │
     │  LaunchDarklyFeatureFlagManager     │        │  @launchdarkly/js-client-sdk    │
     │      ▲                              │        │      ▲                          │
     │  FeatureFlagManager (facade)        │        │  *ngIf / FeatureFlagGuard       │
     │      ▲                              │        │                                 │
     │  NonconformanceService, etc.        │        │  Components / Routes            │
     └─────────────────────────────────────┘        └─────────────────────────────────┘
                  │  REST (existing)                          │
                  └───────────────────────────────────────────┘
       (Optionally: secure-mode hash computed in Java, passed to the SPA)
```

**Key decisions:**

1. **Local in-process evaluation in both tiers.** The SDK maintains an in-memory flag store fed by a streaming connection; `boolVariation(...)` is a local lookup (no per-request network call). This is fast and resilient.
2. **Backend = server SDK key (secret).** Grants read of the full ruleset → must live in OCI Vault / env var, never in code or the browser.
3. **Frontend = client-side ID (public).** Only flags explicitly marked "client-side available" are exposed. Pair with **secure mode** (Java computes HMAC of the context; SPA passes it) for authenticated multi-tenant safety.
4. **Central wrapper, not scattered `if`s.** `FeatureFlagManager` (facade) → `LaunchDarklyFeatureFlagManager` (all LD-specific calls) — exactly the structure prescribed in the Deltek doc, plus a flag-key registry.

---

## 4. Backend Implementation (Java / Spring)

### 4.1 Dependency — `TipQAServices/pom.xml`

```xml
<dependency>
  <groupId>com.launchdarkly</groupId>
  <artifactId>launchdarkly-java-server-sdk</artifactId>
  <version>7.14.0</version>   <!-- latest stable, May 2026; Java 8+ -->
</dependency>
```

### 4.2 Singleton client bean — graceful startup & shutdown

`com.tiptech.tipqaweb.config.LaunchDarklyConfig`:

```java
@Configuration
public class LaunchDarklyConfig {

    private static final Logger log = LogManager.getLogger(LaunchDarklyConfig.class);

    // destroyMethod = "close" → Tomcat tears the client down cleanly on undeploy/shutdown,
    // flushing pending analytics events. Matches the existing contextDestroyed() lifecycle.
    @Bean(destroyMethod = "close")
    public LDClient ldClient(@Value("${launchdarkly.sdk.key:}") String sdkKey) {
        LDClient client = new LDClient(sdkKey);   // connects on construct (5s default timeout)
        if (!client.isInitialized()) {
            log.warn("LaunchDarkly client not initialized at startup; serving default "
                   + "variations and retrying in background.");
        }
        return client;
    }
}
```

> Source the key the same way `dela.properties` sources API keys: `launchdarkly.sdk.key=${LAUNCHDARKLY_SDK_KEY:}` in `system.properties`, env var injected at runtime, backed by **OCI Vault** per the Deltek standard. Never commit it.

### 4.3 Central facade + LD-specific manager (Deltek-prescribed structure)

```java
// Facade the rest of the app depends on — keeps business code provider-agnostic.
public interface FeatureFlagManager {
    boolean isEnabled(FeatureFlag flag, LDContext context);
    String  stringValue(FeatureFlag flag, LDContext context, String fallback);
}

// Single class that touches the LD SDK. Swapping providers = rewrite only this class.
@Service
public class LaunchDarklyFeatureFlagManager implements FeatureFlagManager {

    private final LDClient ldClient;

    @Autowired
    public LaunchDarklyFeatureFlagManager(LDClient ldClient) { this.ldClient = ldClient; }

    @Override
    public boolean isEnabled(FeatureFlag flag, LDContext context) {
        return ldClient.boolVariation(flag.key(), context, flag.defaultValue());
    }

    @Override
    public String stringValue(FeatureFlag flag, LDContext context, String fallback) {
        return ldClient.stringVariation(flag.key(), context, fallback);
    }
}
```

### 4.4 Central flag registry (no stringly-typed keys scattered around)

```java
public enum FeatureFlag {
    NC_ENHANCED_WORKFLOW("com.deltek.TFS123.nc.enhanced.workflow", false),
    UAM_LOGIN          ("com.deltek.TFS789.uam.login",            false);

    private final String key; private final boolean defaultValue;
    FeatureFlag(String key, boolean def) { this.key = key; this.defaultValue = def; }
    public String key() { return key; }
    public boolean defaultValue() { return defaultValue; }
}
```

### 4.5 Building the context from `UserInfo`

```java
public LDContext currentContext(UserInfo userInfo) {
    LDContext user = LDContext.builder(userInfo.getLoginId()).kind("user")
            .name(userInfo.getFullName()).build();
    LDContext tenant = LDContext.builder(userInfo.getMasterBusinessUnit()).kind("organization")
            .set("businessUnit", userInfo.getBusinessUnit()).build();
    LDContext app = LDContext.builder("tipqa@" + buildVersion).kind("application")
            .set("version", buildVersion).build();
    return LDContext.createMulti(user, tenant, app);
}
```

### 4.6 Usage at the service layer (Factory pattern for complex features)

For a **simple** toggle (the Deltek "simple feature" example):

```java
if (featureFlagManager.isEnabled(FeatureFlag.NC_ENHANCED_WORKFLOW, ctx)) {
    return processNonconformanceV2(request);
}
return processNonconformanceV1(request);   // legacy / noop fallback
```

For a **complex/long-lived** feature, use the **Factory + interface** pattern from the Deltek doc (`NotificationSender` → `EmailNotification` / `NoopNotification` chosen by `NotificationFactory.create()`), so removal later is a localized edit.

### 4.7 Testing — `TestData` source (no network, deterministic)

```java
TestData td = TestData.dataSource();
td.update(td.flag("com.deltek.TFS123.nc.enhanced.workflow")
            .variationForKey(ContextKind.of("organization"), "ACME", true)
            .fallthroughVariation(false));
LDClient testClient = new LDClient("test", new LDConfig.Builder().dataSource(td).build());
```

---

## 5. Frontend Implementation (Angular)

### 5.1 Dependency & environment

```bash
npm install @launchdarkly/js-client-sdk   # v4.x — recommended for new Angular 20 work
```
> Alternative: `launchdarkly-js-client-sdk` v3.x (the package named in the Deltek doc) is still supported and uses `initialize(...)`. Prefer v4.x (`createClient().start()`) for new work.

```typescript
// environment.ts / environment.prod.ts
export const environment = {
  production: false,
  launchDarklyClientSideId: 'NON-SECRET-CLIENT-SIDE-ID',   // public, per-env
  // ...existing keys
};
```

### 5.2 Injectable service + init in `APP_INITIALIZER`

```typescript
@Injectable({ providedIn: 'root' })
export class LaunchDarklyService {
  private client!: LDClient;
  private flags$ = new BehaviorSubject<Record<string, any>>({});

  async init(user: User): Promise<void> {
    const context = {
      kind: 'multi',
      user:         { key: user.getLoginId(), name: `${user.getFirstName()} ${user.getLastName()}` },
      organization: { key: user.getUserData().masterBusinessUnit, businessUnit: user.getBusinessUnit() },
      application:  { key: 'tipqa', version: environment.appVersion },
    };
    this.client = createClient(environment.launchDarklyClientSideId, context /*, { hash } */);
    this.client.start();
    await this.client.waitForInitialization({ timeout: 5 });
    this.client.on('change', () => this.flags$.next(this.client.allFlags()));
    this.flags$.next(this.client.allFlags());
  }

  isEnabled(key: string): boolean { return this.client?.boolVariation(key, false) ?? false; }
  isEnabled$(key: string): Observable<boolean> {
    return this.flags$.pipe(map(() => this.isEnabled(key)), distinctUntilChanged());
  }
}
```

Wire a **second `APP_INITIALIZER`** in `app.config.ts` that runs after the existing `initializeApp` (which already loads the user), calling `ldService.init(userService.user)`. This guarantees flags are ready before routing and **avoids flicker** (LD bootstrapping pattern).

### 5.3 Conditional rendering & route guards (match existing style)

```html
<!-- Template: same *ngIf idiom already used across the app -->
<new-nc-panel *ngIf="ldService.isEnabled('com.deltek.TFS123.nc.enhanced.workflow')"></new-nc-panel>
```

```typescript
// FeatureFlagGuard — mirrors the existing functional ModuleAccessGuard
export const FeatureFlagGuard: CanActivateFn = (route) => {
  const ld = inject(LaunchDarklyService);
  const router = inject(Router);
  const flag = route.data?.['featureFlag'] as string;
  if (!flag || ld.isEnabled(flag)) return true;
  router.navigate(['/app/dashboard']);
  return false;
};
// Route: { path: 'beta', loadComponent: ..., canActivate: [FeatureFlagGuard], data: { featureFlag: 'com.deltek.TFS999.beta' } }
```

### 5.4 Billing-aware note

UI evaluation is metered as **Monthly Active Users (MAU)** by the highest-cardinality context kind; backend evaluation is metered as **service connections**. Keep `user.key` stable (the login ID) and avoid inventing high-cardinality kinds (e.g., per-device) unnecessarily.

---

## 6. Client-side vs Server-side — why both, and the cost model

| | Backend (Java Server SDK) | Frontend (JS Client SDK) |
|---|---|---|
| Key | **Server SDK key** (secret) | **Client-side ID** (public) |
| Exposes | Entire ruleset | Only flags marked "client-side available" |
| Billing meter | Service connections (per SDK instance/time) | MAU (unique contexts/month) |
| Use for | Business logic, API route gating, data decisions | Show/hide UI, route guards, UX variations |
| Security add-on | computes `secureModeHash(context)` | passes `hash` to prevent context spoofing |

Per the Deltek doc and LD billing model, **UI flags should be evaluated directly in the browser** with the client-side ID (don't proxy every UI check through the backend).

---

## 7. Architecture Trade-off: SDK-in-process vs a network "Feature Flag Service"

The Deltek doc speaks of consumers calling a "Feature Flag service" via **client-credentials (Keycloak shared-services realm) + OCI Vault + token caching**. Reconciling that with LaunchDarkly's design:

- **For flag *evaluation*: use the in-process SDK (local eval).** Building a per-request network hop to a central "flag service" reintroduces latency and a single point of failure that the SDK's in-memory streaming model is specifically designed to avoid. LaunchDarkly's recommended model is local evaluation inside each process.
- **The Deltek "Feature Flag Service" is best read as the provisioning/management + secrets layer** (client/scope creation in Keycloak during CI/CD, SDK-key custody in OCI Vault), **not** an evaluation hop. TIP should consume that layer to *obtain its SDK key/credentials*, then evaluate locally via the SDK.
- **Centralization in TIP = an in-process library** (`FeatureFlagManager`), not a microservice.
- **Relay Proxy** becomes relevant **only** when TIP scales to many backend instances/microservices, needs private/air-gapped flag delivery, or uses large segments. Not needed for the initial WAR-based deployment.

**Recommendation:** Direct SDK + in-process wrapper now; revisit Relay Proxy at fleet scale.

---

## 8. Flag Governance & Standards (Deltek + LaunchDarkly)

- **Naming (Deltek):**
  - Headline/feature: `com.deltek.TFS123.<area>.<feature>`
  - Story/PBI: `com.deltek.PBI456.<area>.<detail>`
  - Defect: `com.deltek.PBI2435.<area>`
  - Long-lived/value/config (no TFS id): `com.deltek.<area>.<config>` (e.g. `com.deltek.uam.people.import.limit`)
  - Enforce key case + required prefix at the LD project level.
- **Lifecycle (Deltek "two gates"):** flags created **OFF**; turned **ON only during the Dev gate and the QA gate** for regression/feature testing; PM toggles on in prod after Resolved/Closed.
- **Temporary vs permanent:** mark release flags temporary (delete after 100% rollout verified); kill-switches/config flags permanent.
- **Removal:** use **Code References** to find all usages → remove from code → **archive (don't delete)** in LD to preserve history and avoid key reuse. Add flag cleanup to the sprint definition-of-done.
- **One flag = one concern;** compose complex features with dependent flags rather than one mega-flag.
- **Flags created/toggled only via the LD UI** (never in code).

---

## 9. Phased Rollout Plan

| Phase | Goal | Backend | Frontend | Exit criteria |
|---|---|---|---|---|
| **0. Foundations** | Accounts, key custody | LD project/env setup; SDK key in OCI Vault; client created in Keycloak shared-services realm via CI/CD | LD client-side ID added to env files | Keys retrievable at runtime; naming/case rules enforced in LD |
| **1. Backend wiring** | Singleton client + facade | Add dep to `TipQAServices`; `LaunchDarklyConfig` `@Bean`; `FeatureFlagManager` + registry + context builder; close() on shutdown | — | `isInitialized()==true` in a real env; `/health` reflects LD status |
| **2. Frontend wiring** | Service + init | — | `LaunchDarklyService`; 2nd `APP_INITIALIZER`; `FeatureFlagGuard` | A test flag flips UI live (streaming) without flicker |
| **3. Pilot flag** | One real headline behind a flag, end-to-end | Gate one NC/CA method (Factory pattern) | Gate the matching UI panel | Toggle ON in Dev gate → QA gate → prod, per tenant |
| **4. Secure mode** | Harden multi-tenant SPA | `secureModeHash(context)` endpoint | pass `hash` to `createClient` | Spoofing other contexts blocked |
| **5. Governance & cleanup** | Make it sustainable | TestData in unit tests | bootstrap flags in tests | Code References enabled; cleanup in DoD; first flag archived |

---

## 10. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| **Java 16 is non-LTS / EOL** | LD SDK 7.x runs on Java 8+, so unblocked today; plan Java 17/21 LTS upgrade separately (Spring 6 supports 17+). |
| **SDK key leakage** | Server key only in OCI Vault/env; client-side ID is public by design; mark only intended flags "client-side available"; enable secure mode. |
| **Flag sprawl / tech debt** | Central registry/enum, Code References, archive-don't-delete, cleanup in DoD, naming enforcement. |
| **Startup blocking / outage** | Constructor has a 5s timeout then serves defaults & retries; always pass a safe default to every `*Variation` call. |
| **UI flicker on first paint** | Bootstrap flags (server-evaluated `allFlagsState` injected into initial payload) and/or init in `APP_INITIALIZER` before routing. |
| **MAU cost surprise** | Keep `user.key` stable; avoid high-cardinality context kinds; review the billing meter periodically. |
| **JS package ambiguity** | Standardize on `@launchdarkly/js-client-sdk` v4.x for new work (doc references the older v3.x `launchdarkly-js-client-sdk`). |
| **Tension with "network FF service"** | Use in-process SDK for evaluation; treat Deltek's FF service as provisioning/secrets layer (§7). Confirm with platform team. |

---

## 11. Open Questions for the Team

1. **Keycloak / OCI Vault provisioning:** Is the client+scopes auto-created against the `deltek-shared-services` realm during TIP's CI/CD, and where exactly is the LD SDK key stored/retrieved? (Dheeraj's unresolved comment in the source doc.)
2. **Central FF service scope:** Is the Deltek "Feature Flag Service" an evaluation hop or a provisioning/secrets layer? (Drives §7.)
3. **Relay Proxy:** Current/expected number of backend instances — do we need it on day one?
4. **Context model sign-off:** Are `loginId` (user), `masterBusinessUnit` (organization/tenant), and build version (application) the right targeting dimensions?
5. **Secure mode:** Required for the initial release, or fast-follow?
6. **JS SDK version:** OK to standardize new Angular work on `@launchdarkly/js-client-sdk` v4.x?

---

## Appendix A — Key File Touchpoints

**Backend**
- `server/TipQAServices/pom.xml` — add SDK dependency
- `server/TipQAWeb/.../config/LaunchDarklyConfig.java` — `@Bean LDClient` (new)
- `server/TipQAServices/.../service/FeatureFlagManager.java` + `LaunchDarklyFeatureFlagManager.java` + `FeatureFlag.java` (new)
- `server/TipQAWeb/src/main/webapp/WEB-INF/system.properties` — `launchdarkly.sdk.key=${LAUNCHDARKLY_SDK_KEY:}`
- `server/TipQAWeb/.../listener/TipServletContextListener.java` — optional shutdown log

**Frontend**
- `client/package.json` — add `@launchdarkly/js-client-sdk`
- `client/src/environments/environment*.ts` — add `launchDarklyClientSideId`
- `client/src/sm_client/app/common/services/launchDarkly.service.ts` (new)
- `client/src/sm_client/app/app.config.ts` — second `APP_INITIALIZER`
- `client/src/sm_client/app/services/feature-flag.guard.ts` (new)

## Appendix B — Sources

- LaunchDarkly Java SDK — https://launchdarkly.com/docs/sdk/server-side/java
- Maven Central (Java SDK 7.14.0) — https://central.sonatype.com/artifact/com.launchdarkly/launchdarkly-java-server-sdk/versions
- Context configuration — https://launchdarkly.com/docs/sdk/features/context-config
- Shutting down — https://launchdarkly.com/docs/sdk/features/shutdown
- Test data sources — https://launchdarkly.com/docs/sdk/features/test-data-sources
- Relay Proxy use cases — https://launchdarkly.com/docs/sdk/relay-proxy/use-cases
- JavaScript SDK — https://launchdarkly.com/docs/sdk/client-side/javascript
- Client-side vs server-side — https://launchdarkly.com/docs/sdk/concepts/client-side-server-side
- Bootstrapping (flicker) — https://launchdarkly.com/docs/sdk/features/bootstrapping
- Secure mode — https://launchdarkly.com/docs/sdk/features/secure-mode
- Calculating billing (MAU vs service connections) — https://launchdarkly.com/docs/home/account/calculating-billing
- Flag naming / conventions — https://launchdarkly.com/docs/guides/flags/flag-conventions
- Reducing technical debt — https://launchdarkly.com/docs/guides/flags/technical-debt
- Internal: Deltek "Feature Flags Service Implementation" doc; TIP codebase `C:\TIP\TIP\code`
</content>
</invoke>
