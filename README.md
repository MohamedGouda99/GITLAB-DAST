# GITLAB-DAST
* **Full scans (active + passive)** automatically on **non-production branches** against your 3 *test* sites
* **On-demand (manual) full scans** on **main** against your 3 *prod* sites
* Optional **scheduled** prod scans via the GitLab UI (best practice)

I’ve used placeholder names so you can plug in your exact URLs/selectors without revealing secrets.

---

# A) One-time prep

1. **Create CI/CD variables (masked, protected):**
   Project → Settings → CI/CD → Variables

* For **example1**: `EX1_USERNAME`, `EX1_PASSWORD`
* For **example2**: `EX2_USERNAME`, `EX2_PASSWORD`
* For **example3**: `EX3_USERNAME`, `EX3_PASSWORD`
* (add any API tokens/headers you need, e.g., `EX1_BEARER_TOKEN`)

> These will be injected into the jobs and **not** stored in profiles.

---

# B) Scanner profile (once)

Project → Security & Compliance → Configuration → **DAST → Scanner Profiles → New**

* **Name:** `full-active`
* **Full scan:** **ON** (enables active + passive)
* **Attack strength / Alert threshold:** start **Medium / Medium**
* **AJAX spider:** ON if your app is SPA/uses heavy JS
* **Max duration:** e.g., 45–60 min (tune later)

*(You’ll reuse this one profile everywhere.)*

---

# C) Site profiles (six total)

Create **3 test** and **3 prod** profiles. Below is a template—repeat for each site.

> **Test – example1** → Name: `example1-test`
> **Prod – example1** → Name: `example1-prod`
> (Repeat for example2 & example3)

**For each site profile:**

* **Target URL:**

  * Test: `https://example1.test/`
  * Prod: `https://example1.com/`
* **Authentication URL:** `https://…/sign-in`
* **Excluded URLs:** `https://…/logout`
* **Request headers (optional):**
  If you use bearer/session headers:
  `Authorization: Bearer ${EX1_BEARER_TOKEN}` (store token as CI var)
* **Additional variables** (copy/paste, then adjust selectors):

  ```
  DAST_AUTH_USERNAME_FIELD=css:input[name="identifier"]
  DAST_AUTH_PASSWORD_FIELD=css:input[name="credentials.passcode"]
  DAST_AUTH_FIRST_SUBMIT_FIELD=css:input[type="submit"][value="Next"]       # remove if not needed
  DAST_AUTH_SUBMIT_FIELD=css:input[type="submit"][value="Verify"]
  DAST_AUTH_BEFORE_LOGIN_ACTIONS=css:button[data-testid="signin"],css:button[data-testid="button"][aria-label="Accept EULA Terms"]
  DAST_AUTH_SUCCESS_IF_AT_URL=https://example1.*/*
  ```

  > Fix the selectors to match *your* DOM. If you don’t need the “Next” step, delete the `DAST_AUTH_FIRST_SUBMIT_FIELD` line.

**Site validation (if required by your setup):**
Serve the validation text file **from the target domain** (e.g., `https://example1.com/.well-known/...txt`). A public repo alone won’t validate unless your app serves that file.

---

# D) `.gitlab-ci.yml` — drop-in pipeline

Paste this as your project’s **root** `.gitlab-ci.yml`.
It wires everything to your 6 profiles and injects credentials safely.

```yaml
stages:
  - security

# Use the official DAST template
include:
  - template: Security/DAST.gitlab-ci.yml

# ---------- Helpers: set site-specific creds per job ----------
# example1
.default_ex1_vars: &default_ex1_vars
  DAST_USERNAME: "$EX1_USERNAME"
  DAST_PASSWORD: "$EX1_PASSWORD"

# example2
.default_ex2_vars: &default_ex2_vars
  DAST_USERNAME: "$EX2_USERNAME"
  DAST_PASSWORD: "$EX2_PASSWORD"

# example3
.default_ex3_vars: &default_ex3_vars
  DAST_USERNAME: "$EX3_USERNAME"
  DAST_PASSWORD: "$EX3_PASSWORD"

# ---------- AUTO on NON-PROD branches (test sites) ----------
dast_test_example1:
  stage: security
  variables:
    <<: *default_ex1_vars
    DAST_SITE_PROFILE: "example1-test"
    DAST_SCANNER_PROFILE: "full-active"
  rules:
    - if: '$CI_COMMIT_BRANCH && $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'
      when: on_success
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event" && $CI_MERGE_REQUEST_TARGET_BRANCH_NAME != $CI_DEFAULT_BRANCH'
      when: on_success
    - when: never
  artifacts:
    reports:
      dast: gl-dast-example1-test.json
    expire_in: 1 week

dast_test_example2:
  stage: security
  variables:
    <<: *default_ex2_vars
    DAST_SITE_PROFILE: "example2-test"
    DAST_SCANNER_PROFILE: "full-active"
  rules:
    - if: '$CI_COMMIT_BRANCH && $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'
      when: on_success
    - when: never
  artifacts:
    reports:
      dast: gl-dast-example2-test.json
    expire_in: 1 week

dast_test_example3:
  stage: security
  variables:
    <<: *default_ex3_vars
    DAST_SITE_PROFILE: "example3-test"
    DAST_SCANNER_PROFILE: "full-active"
  rules:
    - if: '$CI_COMMIT_BRANCH && $CI_COMMIT_BRANCH != $CI_DEFAULT_BRANCH'
      when: on_success
    - when: never
  artifacts:
    reports:
      dast: gl-dast-example3-test.json
    expire_in: 1 week

# ---------- MANUAL on DEFAULT branch (prod sites) ----------
# Click "Play" on main to run a full scan against prod
dast_prod_example1:
  stage: security
  variables:
    <<: *default_ex1_vars
    DAST_SITE_PROFILE: "example1-prod"
    DAST_SCANNER_PROFILE: "full-active"
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
      allow_failure: false
    - when: never
  protected: true
  artifacts:
    reports:
      dast: gl-dast-example1-prod.json
    expire_in: 1 week

dast_prod_example2:
  stage: security
  variables:
    <<: *default_ex2_vars
    DAST_SITE_PROFILE: "example2-prod"
    DAST_SCANNER_PROFILE: "full-active"
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
    - when: never
  protected: true
  artifacts:
    reports:
      dast: gl-dast-example2-prod.json
    expire_in: 1 week

dast_prod_example3:
  stage: security
  variables:
    <<: *default_ex3_vars
    DAST_SITE_PROFILE: "example3-prod"
    DAST_SCANNER_PROFILE: "full-active"
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
    - when: never
  protected: true
  artifacts:
    reports:
      dast: gl-dast-example3-prod.json
    expire_in: 1 week
```

**What you get:**

* Any push/MR to **non-main** branches triggers **full scans** on your 3 **test** targets.
* On **main**, you’ll see 3 **manual** jobs for the **prod** targets—click *Play* when you want to run them.

> Want **cron scheduling** for prod? Use **Security & Compliance → On-Demand Scans** in the UI, pick each `exampleX-prod` site profile + `full-active` scanner profile, and add a schedule. (This avoids running on every main push and keeps schedules managed in the security UI.)

---

# E) Quick sanity checklist

* Runner can reach your sites (VPN/VPC/WAF rules OK).
* Site validation file (if used) is served by each domain.
* Login still works when driven by a headless browser—test selectors locally if unsure.
* Start **Medium/Medium**; if you see rate-limits in prod, make a second scanner profile like `full-active-throttled` with lower attack strength and map it for prod jobs.

---

# F) Hand-off

> “We configured GitLab DAST to run full (active+passive) scans automatically on non-production branches for our three test environments, and to expose manual on-demand full scans on the main branch for the three production environments. Profiles are centrally managed (one full-scan Scanner Profile; six Site Profiles with authentication and exclusions). Reports publish to the Security dashboard and as pipeline artifacts.”

---
