# LaunchDarkly POC — Java backend + Angular frontend

A deliberately tiny POC to test LaunchDarkly end to end. One flag, evaluated in
two places:

1. **Backend** (Java / Spring Boot) — uses the **server SDK key** (secret).
2. **Frontend** (Angular) — uses the **client-side ID** (public), and updates
   **live** when you toggle the flag in the LaunchDarkly dashboard.

```
poc/
├── backend/    Spring Boot app  →  GET /api/feature-flag?user=poc-user
└── frontend/   Angular app      →  shows the flag ON/OFF + "Check backend" button
```

---

## Prerequisites (already verified on this machine)

| Tool | Version |
|------|---------|
| Java | 21 |
| Node | 20.19 |
| npm | 10.8 |

> **Maven:** you don't need it installed. The backend ships with a **Maven Wrapper**
> (`mvnw` / `mvnw.cmd`) pinned to Maven 3.9.9, which it downloads automatically on
> first use. (The system Maven 3.3.9 is too old for Spring Boot 3.3, so use `.\mvnw`.)

---

## Step 1 — Create the flag in LaunchDarkly

1. Log in to LaunchDarkly → your project → **Feature flags** → **Create flag**.
2. Key: **`poc-test-flag`** (must match exactly). Type: **Boolean**.
3. **Important for the frontend:** in the flag's settings, enable
   **"SDKs using Client-side ID"** (a.k.a. *make available to client-side SDKs*).
   Without this the Angular app cannot see the flag.
4. Leave the flag **Off** to start (we'll toggle it later to watch it change live).

## Step 2 — Get your two keys

From **Account settings → Projects → (your environment)**:

- **SDK key** (server, secret) → used by the **backend**.
- **Client-side ID** (public) → used by the **frontend**.

---

## Step 3 — Run the backend

Set the SDK key as an environment variable and start the app.

**PowerShell:**
```powershell
cd "C:\FF POC\LD_POC\poc\backend"
$env:LD_SDK_KEY = "sdk-xxxxxxxx-your-server-sdk-key"
.\mvnw spring-boot:run
```

Verify (in a browser or new terminal):
```
http://localhost:8080/api/feature-flag?user=poc-user
```
You should see JSON like:
```json
{ "flagKey": "poc-test-flag", "user": "poc-user", "enabled": false, "evaluatedBy": "backend (Java Server SDK)" }
```

## Step 4 — Run the frontend

1. Paste your **client-side ID** into
   `frontend/src/environments/environment.ts` (`launchDarklyClientSideId`).
2. Install and serve:

**PowerShell:**
```powershell
cd "C:\FF POC\LD_POC\poc\frontend"
npm install
npm start
```

3. Open **http://localhost:4200**.

---

## Step 5 — See it work

- The page shows the flag as **OFF** (client-side, from the browser SDK).
- Click **Check backend** → it calls the Java app and shows the same value.
- Now go to the **LaunchDarkly dashboard** and **toggle `poc-test-flag` ON**.
  - The Angular page flips to **ON within a second or two — no refresh** (streaming).
  - Click **Check backend** again → the backend also returns **ON**.

That's the whole loop: one flag, controlled from the dashboard, evaluated on
both the server and the client.

---

## How it maps to the bigger plan

| POC piece | Real TIP equivalent (see the implementation plan) |
|-----------|----------------------------------------------------|
| `LaunchDarklyConfig` (`@Bean LDClient`) | Same, added to `TipQAWeb` config; key from OCI Vault |
| `FeatureFlagManager` / `LaunchDarklyFeatureFlagManager` | Same facade names, injected into existing services |
| single `user` context | multi-context: `user` + `organization`(tenant/BU) + `application`(version) |
| `LaunchDarklyService` (Angular) | Same, initialized in `APP_INITIALIZER` after user load |
| flag key `poc-test-flag` | Deltek naming, e.g. `com.deltek.TFS123.area.feature` |

---

## Notes / gotchas

- **Two different keys.** Server SDK key ≠ client-side ID. Don't swap them.
- **Client-side availability.** If the frontend always shows OFF but the backend
  is correct, you almost certainly forgot Step 1.3 (mark the flag client-side available).
- **Default value.** Every evaluation passes a default (`false`). If LaunchDarkly
  is unreachable or the flag is missing, you get the default — the app never breaks.
- **SDK package choice.** The frontend uses `launchdarkly-js-client-sdk` (stable,
  the package referenced in the Deltek doc). For new production work the plan
  recommends `@launchdarkly/js-client-sdk` v4.x — we can switch later.
</content>
