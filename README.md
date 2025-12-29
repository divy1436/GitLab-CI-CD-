# GitLab-CI-CD-
# 1. Run SonarQube with Docker

```bash
docker -d -p 9000:9000 sonarqube:community
```

1. Open [`http://localhost:9000`](http://localhost:9000) in your browser.
2. Log in to SonarQube.
3. Create a new project and give it a name.

## 2. Configure GitLab CI/CD Variables

When you connect SonarQube with GitLab CI, SonarQube shows a screen explaining which CI variables are missing. This is not an error. It is just telling you what to configure.

In your GitLab project, go to:

> **Settings → CI/CD → Variables**
> 

Add the following variables:

1. **Sonar token**
    - **Key:** `SONAR_TOKEN`
    - **Value:** your SonarQube token
    - **Masked:** enabled
    - **Protected:** disabled
2. **Sonar host URL**
    - **Key:** `SONAR_HOST_URL`
    - **Value:** [`http://localhost:9000`](http://localhost:9000)
    - **Masked:** disabled
    - **Protected:** disabled

After saving these variables, rerun the pipeline. The SonarQube analysis should now work.

![image.png](attachment:8a793ec7-a8d9-4892-8b56-4b038cffc142:image.png)

This screen is **SonarQube telling you exactly what variables are missing** and where to add them. Nothing is broken.

### What you must do (only this)

### 1. Open GitLab project settings

GitLab → **Project** → **Settings** → **CI/CD** → **Variables**

---

### 2. Add Sonar token variable

Add a new variable:

- **Key:** `SONAR_TOKEN`
- **Value:** generate or paste your SonarQube token
- **Masked:** ✅ checked
- **Protected:** ❌ unchecked

Save it.

---

### 3. Add Sonar host URL variable

Add another variable:

- **Key:** `SONAR_HOST_URL`
- **Value:** `http://localhost:9000`
- **Masked:** ❌ unchecked
- **Protected:** ❌ unchecked

Save it.

---

### Why this is required

- SonarQube needs **authentication** (`SONAR_TOKEN`)
- SonarQube needs to know **where the server is running** (`SONAR_HOST_URL`)
- Your pipeline cannot run analysis without these two variables

![image.png](attachment:07f52668-5974-45fd-9d90-e12e9697a05a:image.png)

---

### After this

- Rerun your GitLab pipeline
- SonarQube analysis will start working
- This screen will disappear once a successful scan runs

---

### One-line conclusion

This page means **GitLab CI variables are missing**; add `SONAR_TOKEN` and `SONAR_HOST_URL` in GitLab CI/CD variables and rerun the pipeline.

---

![image.png](attachment:3c0ff1b0-0190-45b4-9ca4-b25684a51246:image.png)

### What this highlighted line means

```
sonar.java.binaries=target/classes

```

It tells **SonarQube where the compiled Java `.class` files are**.

Sonar **must** find compiled classes to analyze Java code correctly.

`target/classes` is the **default Maven output directory**.

---

### Why this can cause a problem

You will get errors if **`target/classes` does not exist** at scan time.

That happens when **Sonar runs before Maven build**.

Common error symptoms:

- “No files nor directories matching target/classes”
- “Bytecode of dependencies was not provided”

---

### The only correct fix

**Make sure Maven compile/package runs before Sonar scan.**

Your pipeline order must be:

```
mvn compile or mvn package
↓
sonar-scanner

```

---

### Correct GitLab CI example (important part only)

```yaml
sonarqube-check:
  stage: test
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  script:
    - mvn clean compile
    - sonar-scanner

```

This guarantees:

- `target/classes` is created
- SonarQube can read bytecode
- The warning/error disappears

---

### Do NOT change this line unless

- You are **not using Maven**
- Or your build outputs classes somewhere else

For Maven projects, this line is **correct and required**.

---

# The code of Attaching Sonarqube

```yaml
image:
  name: sonarsource/sonar-scanner-cli:11
  entrypoint: [""]

variables:
  SONAR_USER_HOME: "$CI_PROJECT_DIR/.sonar"
  GIT_DEPTH: "0"

stages:
  - install
  - compile
  - test
  - sonarqube
  - build
  - deploy

# ---------------- INSTALL ----------------
install-job:
  stage: install
  script:
    - set -euo pipefail
    - export DEBIAN_FRONTEND=noninteractive
    - sudo -n apt-get update
    - sudo -n apt-get install -y openjdk-21-jdk-headless maven
    - java -version
    - mvn -version
  tags:
    - self-hosted

# ---------------- COMPILE ----------------
compile-job:
  stage: compile
  needs:
    - install-job
  script:
    - mvn clean compile
  artifacts:
    paths:
      - target/classes
  tags:
    - self-hosted

# ---------------- UNIT TESTS ----------------
unit-test-job:
  stage: test
  needs:
    - compile-job
  script:
    - mvn test
  artifacts:
    paths:
      - target/classes
    expire_in: 1 week
  tags:
    - self-hosted

# ---------------- LINT TESTS ----------------
lint-test-job:
  stage: test
  needs:
    - compile-job
  script:
    - echo "Linting code..."
    - sleep 10
    - echo "No lint issues found."
  tags:
    - self-hosted

# ---------------- SONARQUBE ----------------
sonarqube-check:
  stage: sonarqube
  needs:
    - compile-job
  cache:
    policy: pull-push
    key: "sonar-cache-$CI_COMMIT_REF_SLUG"
    paths:
      - .sonar/cache
  script:
  - |
    sonar-scanner -X \
      -Dsonar.projectKey=BoardGame \
      -Dsonar.projectName=BoardGame \
      -Dsonar.sources=src \
      -Dsonar.java.binaries=target/classes \
      -Dsonar.host.url="$SONAR_HOST_URL" \
      -Dsonar.login="$SONAR_TOKEN"

  allow_failure: true
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "master"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
  tags:
    - self-hosted

# ---------------- BUILD ----------------
build-job:
  stage: build
  needs:
    - unit-test-job
    - lint-test-job
    - sonarqube-check
  script:
    - mvn package
  artifacts:
    paths:
      - target/*.jar
    expire_in: 14 days
  tags:
    - self-hosted

# ---------------- DEPLOY ----------------
deploy-job:
  stage: deploy
  needs:
    - build-job
  environment:
    name: production
  script:
    - echo "Deploying application..."
    - echo "Application successfully deployed."
  tags:
    - self-hosted

```

**Summary**

This is a GitLab CI/CD pipeline for a Java Maven project. It installs tools, compiles code, runs tests, performs SonarQube analysis, builds a JAR, and deploys it. Below is a clear, section-by-section explanation.

---

## 1. Global Image and Variables

### Image

```yaml
image:
  name: sonarsource/sonar-scanner-cli:11
  entrypoint: [""]

```

- Uses a Docker image that already has **SonarScanner** installed.
- `entrypoint: [""]` resets the default entrypoint so GitLab can run shell commands normally.

### Variables

```yaml
variables:
  SONAR_USER_HOME: "$CI_PROJECT_DIR/.sonar"
  GIT_DEPTH: "0"

```

- `SONAR_USER_HOME`: Stores Sonar cache inside the project directory.
- `GIT_DEPTH: 0`: Fetches full Git history, required for accurate Sonar analysis.

---

## 2. Stages Definition

```yaml
stages:
  - install
  - compile
  - test
  - sonarqube
  - build
  - deploy

```

Defines the execution order of jobs. Each stage runs only after the previous one succeeds (unless allowed to fail).

---

## 3. Install Stage

### `install-job`

```yaml
install-job:
  stage: install

```

Purpose: Prepare the runner with required tools.

What happens:

- Updates package list.
- Installs **OpenJDK 21** and **Maven**.
- Prints Java and Maven versions for verification.

Why needed:

- Your runner is self-hosted and does not have Java/Maven preinstalled.

---

## 4. Compile Stage

### `compile-job`

```yaml
compile-job:
  stage: compile
  needs:
    - install-job

```

Purpose: Compile the Java source code.

Key points:

- `mvn clean compile` cleans old builds and compiles fresh code.
- `target/classes` is saved as an artifact for later stages.
- `needs` allows faster pipelines by reusing outputs.

---

## 5. Test Stage

### Unit Tests (`unit-test-job`)

```yaml
unit-test-job:
  stage: test

```

Purpose: Run automated tests.

Details:

- Runs `mvn test`.
- Stores compiled classes as artifacts for 1 week.
- Depends on `compile-job`.

---

### Lint Tests (`lint-test-job`)

```yaml
lint-test-job:
  stage: test

```

Purpose:

- Placeholder for code quality or lint checks.
- Currently simulates linting with `sleep`.

Both test jobs run in parallel within the `test` stage.

---

## 6. SonarQube Stage

### `sonarqube-check`

```yaml
sonarqube-check:
  stage: sonarqube

```

Purpose: Static code analysis using SonarQube.

Important parts:

- Uses Sonar cache to speed up repeated scans.
- Runs `sonar-scanner` with:
    - `sonar.projectKey` and `sonar.projectName`: Project identity in SonarQube.
    - `sonar.sources=src`: Source code location.
    - `sonar.java.binaries=target/classes`: Compiled bytecode.
    - `sonar.host.url`: SonarQube server URL.
    - `sonar.login`: Authentication token.

Special behavior:

- `X` enables debug logging.
- `allow_failure: true` means pipeline continues even if Sonar fails.
- `rules` restrict execution to main branches and merge requests.

---

## 7. Build Stage

### `build-job`

```yaml
build-job:
  stage: build

```

Purpose: Create the final application package.

What it does:

- Runs `mvn package`.
- Produces a `.jar` file.
- Stores JAR artifacts for 14 days.
- Runs only after tests and Sonar analysis complete.

---

## 8. Deploy Stage

### `deploy-job`

```yaml
deploy-job:
  stage: deploy

```

Purpose: Deployment step.

Current behavior:

- Simulated deployment using `echo`.
- Marks environment as `production`.

In real use, this is where you would:

- Copy JAR to server
- Run Docker/Kubernetes deployment
- Restart services

---

## 9. Tags

```yaml
tags:
  - self-hosted

```

- Ensures all jobs run on your **self-hosted GitLab runner**, not shared runners.

---

## Overall Flow

1. Install Java and Maven on the self-hosted runner.
2. Compile the project and save `target/classes` as an artifact.
3. Run unit tests and (optional) lint checks.
4. Run SonarQube analysis using the compiled classes.
5. Build the final JAR with Maven.
6. Deploy the application (currently a placeholder with `echo` commands).

![image.png](attachment:3d37b9cc-e043-4003-83a9-8139ce75277e:image.png)

![image.png](attachment:c8b4fdb4-dc0c-4603-b097-2c8de9d696dc:image.png)

![image.png](attachment:976d2d1c-b598-4dd2-8b5d-3934eeafcf4d:image.png)
