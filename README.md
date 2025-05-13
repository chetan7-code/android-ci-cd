# Android CI-CD Pipeline  
_A GitHub Actions + Firebase App Distribution playbook_

---

## 💡 What you get
* **Automatic checks on every Pull-Request.**  
  Lint, unit-tests, and a debuggable APK artifact.
* **Release pipeline on `main` and version tags.**  
  Signed AAB, uploaded to Firebase App Distribution (and optionally Google Play Internal Sharing).
* **Everything codified in one workflow file** – see `.github/workflows/android.yml`.

The rest of this document is a step-by-step guide that explains **how** to enable the pipeline and, more importantly, **why each stage exists**.

---

## 📜 Table of Contents
1. Architecture diagram  
2. Prerequisites  
3. GitHub secrets  
4. Branch strategy & triggers  
5. Full workflow listing  
6. Stage-by-stage explanation (“What / Why”)  
7. Local testing hints  
8. Troubleshooting  
9. FAQ  
10. License

---

## 1. Architecture Diagram

```
┌──────────────────────┐        Pull-Request
│ Developer pushes code│  ─────────────────────────▶ “PR Checks” job
└──────────┬───────────┘
           │                        ┌─────────────────────────────────────────────────┐
           │ push main / v* tag     │        “Release” job                           │
           ├───────────────────────▶│ 1. Checkout + cache + JDK                      │
           │                       │ 2. Lint & tests                                 │
           │                       │ 3. Build debug APK (artifact)                   │
           │                       │ 4. Build & sign release AAB                     │
           │                       │ 5. Upload to Firebase App Distribution          │
           │                       │ 6. (opt) Promote to Google Play Internal track  │
           │                       └─────────────────────────────────────────────────┘
    GitHub Actions
```

---

## 2. Prerequisites

| Item | Min. version | Why you need it |
|------|--------------|-----------------|
| Android Gradle Plugin | 7.0 | Compatible with modern tooling & JDK 17 |
| Gradle Wrapper | 7.5 | Supported by `gradle/actions/cache` |
| Firebase Project | n/a | Destination for QA builds |
| Service-account (or CI token) | IAM role: _Firebase App Distribution Admin_ | Auth for upload step |
| Android release keystore | any | Signing the release bundle (.aab) |
| (Optional) Google Play service-account | Internal App Sharing permission | For post-tag promotion |

---

## 3. GitHub Secrets

| Secret | Description |
|--------|-------------|
| `FIREBASE_TOKEN` or `FIREBASE_SERVICE_ACCOUNT_JSON` | Auth for `wzieba/Firebase-Distribution-Github-Action` |
| `FIREBASE_APP_ID` | The **App ID** (not project ID) e.g. `1:1234567890:android:abcdef123456` |
| `ANDROID_KEYSTORE_BASE64` | Base64-encoded `keystore.jks` |
| `KEYSTORE_PASSWORD` | Store password |
| `KEY_ALIAS` | Key alias |
| `KEY_PASSWORD` | Alias password |
| `PLAY_SERVICE_ACCOUNT_JSON` (optional) | Base64 of Play Console JSON key |

_Why keep them in Secrets? ➜ They are encrypted, masked in logs, and only exposed to trusted branches/jobs._

---

## 4. Branch Strategy & Triggers

| Branch / Tag | Pipeline executed | Distribution target |
|--------------|------------------|---------------------|
| Pull-Requests → `main`/`develop` | Lint • Unit tests • Debug APK artifact | _none_ |
| `develop`     | Same as PR + Firebase debug build _(groups=qa)_ | QA testers |
| `main`        | Full release build + Firebase (groups=stakeholders) | Stakeholders |
| `v*` tag      | Same as `main` + upload to Play Internal Sharing | Org-wide |

_Why this split? ➜ Prevents malicious PRs from accessing signing keys, reduces Firebase quota usage, and keeps fast feedback for contributors._

---

## 5. Full Workflow File

the workflow file is present in .github/workflows directory still some changes has to be made but the artifact(apk) has generated

## 6. Stage-by-Stage Explanation

| # | Stage | What happens | Why we do it |
|---|-------|--------------|--------------|
| 1 | **Checkout** | Pulls full git history. | Versioning scripts often use `git describe`. Shallow clones break that. |
| 2 | **Cache Gradle** | Restores downloaded dependencies & wrapper. | Saves ≈3-5 min per run and network traffic. |
| 3 | **Setup JDK** | Installs Temurin 17 & configures Gradle cache. | AGP 8.x requires JDK 17. |
| 4 | **Decode Keystore** | Writes `keystore.jks` to disk (release builds only). | Keep signing credentials out of PRs for security. |
| 5 | **Lint & Tests** | Runs `lintDebug` and JUnit/Kotlin tests. | Catch style & logic errors early, fail fast. |
| 6 | **Assemble Debug** | Produces `app-debug.apk`; uploaded as artifact. | Allows anyone to sideload the latest build without deploying. |
| 7 | **Assemble & Sign Release** | Builds `app-release.aab` with injected signing params. | Generates store-ready bundle; avoids committing secrets in `build.gradle`. |
| 8 | **Firebase App Distribution** | Uploads bundle & release notes to groups. | OTA delivery, crash reporting integration, stakeholder feedback loop. |
| 9 | **Google Play Internal** | For version tags only, pushes the same bundle to Internal Sharing track. | Verify Play-store compliance before public rollout; no review needed, live in minutes. |

---

## 7. Local Testing Hints

1. Run `./gradlew lintDebug testDebugUnitTest` locally before opening a PR.  
2. Use `./gradlew bundleRelease` with env vars exported to ensure signing works.  
3. Need to test the Firebase step? Use `firebase appdistribution:distribute` locally with the same service-account.

---

## 8. Troubleshooting

| Symptom | Fix |
|---------|-----|
| Gradle cache miss every build | Ensure `gradle-wrapper.properties` is committed; check cache key pattern. |
| “Keystore tampered or password incorrect” | Validate `KEYSTORE_PASSWORD` / `KEY_PASSWORD` pairs; confirm base64 generated with `-w0`. |
| Firebase upload fails 403 | Service-account lacks `firebaseappdistribution.apps.releases` permissions or token expired. |
| Play upload 401 | Link the service-account to the Play Console and grant _Internal App Sharing_. |

Enable verbose logging by re-running the workflow with the **“Re-run – Enable debug logging”** button.

---

## 9. FAQ

**Why not build on self-hosted Mac runners?**  
Linux GitHub-hosted `ubuntu-latest` images are free (public) or covered by minutes (private) and already contain Android SDK components, saving setup time.

**Can I deploy an APK instead of an AAB?**  
Yes – replace `bundleRelease` with `assembleRelease` and change the `file:` path in the Firebase step. Google Play Internal Sharing accepts both.

**What about instrumented UI tests?**  
Add another job that uses `macos-latest` or **Firebase Test Lab** action before the release step.

---

## 10. License

MIT. See [`LICENSE`](LICENSE) for full text.
