# CI/CD Pipelines

**CI/CD automates the process of building, testing, and deploying applications — Continuous Integration ensures code is always tested, Continuous Delivery ensures it is always deployable, and Continuous Deployment automates the final step of releasing to production.**

---

## Table of Contents

- [What is CI/CD?](#what-is-cicd)
- [Continuous Integration (CI)](#continuous-integration-ci)
- [Continuous Delivery vs Continuous Deployment](#continuous-delivery-vs-continuous-deployment)
- [CI/CD Pipeline Stages](#cicd-pipeline-stages)
- [CI/CD Tools](#cicd-tools)
- [Integration with GitOps](#integration-with-gitops)
- [What to Remember for the Exam](#what-to-remember-for-the-exam)

---

## What is CI/CD?

CI/CD is a set of practices that automate the software delivery lifecycle. It is the backbone of modern DevOps and cloud native workflows.

```
+-----------------------------------------------------------------------+
|                          CI/CD SPECTRUM                                |
|                                                                       |
|   Continuous          Continuous            Continuous                 |
|   Integration         Delivery              Deployment                |
|                                                                       |
|   +---------------+   +------------------+  +---------------------+   |
|   | Auto build    |   | Always ready     |  | Auto deploy to      |   |
|   | Auto test     |   | to deploy        |  | production          |   |
|   | On every      |   | Manual trigger   |  | No manual step      |   |
|   | commit        |   | to production    |  |                     |   |
|   +---------------+   +------------------+  +---------------------+   |
|                                                                       |
|   <---------- increasing automation --------------------------------> |
+-----------------------------------------------------------------------+
```

---

## Continuous Integration (CI)

**Continuous Integration** is the practice of automatically building and testing code every time a developer pushes a commit to the shared repository.

### Goals

- Catch bugs early (fail fast)
- Ensure code compiles and passes tests
- Maintain a working main branch at all times

### What Happens in CI

```
Developer pushes code
        |
        v
+------------------+
|  Source Control   |  (e.g., GitHub, GitLab)
+--------+---------+
         |
         v  triggers
+------------------+
|   Build Stage    |
|  - Compile code  |
|  - Build image   |
+--------+---------+
         |
         v
+------------------+
|   Test Stage     |
|  - Unit tests    |
|  - Lint / SAST   |
|  - Integration   |
+--------+---------+
         |
    Pass / Fail
         |
    +----+----+
    |         |
  Pass      Fail
    |         |
    v         v
  Merge    Notify developer
  allowed  (fix and re-push)
```

### Key Practices

- Developers push to a shared repo **frequently** (at least daily)
- Every push triggers an automated build and test
- If the build breaks, fixing it is the **top priority**
- Automated tests provide fast feedback

---

## Continuous Delivery vs Continuous Deployment

These two terms are often confused. The key difference is the **manual gate** before production.

### Continuous Delivery (CD)

Code is **always in a deployable state.** After passing CI, the artifact is ready for production, but a **human makes the decision** to deploy.

```
Code --> Build --> Test --> Staging --> [MANUAL APPROVAL] --> Production
                                             ^
                                             |
                                    Human clicks "Deploy"
```

### Continuous Deployment (CD)

Every change that passes the automated pipeline is **automatically deployed to production.** No human intervention.

```
Code --> Build --> Test --> Staging --> Production
                                  (fully automated,
                                   no manual gate)
```

### Comparison

```
                    Continuous       Continuous
                    Delivery         Deployment
                    --------         ----------
Auto build & test:     YES              YES
Auto deploy to
  staging:             YES              YES
Auto deploy to
  production:          NO               YES
                       (manual          (fully
                        trigger)         automated)
```

---

## CI/CD Pipeline Stages

A typical CI/CD pipeline has four main stages:

```
+----------+     +---------+     +----------+     +----------+
|  SOURCE  | --> |  BUILD  | --> |   TEST   | --> |  DEPLOY  |
+----------+     +---------+     +----------+     +----------+
|            |            |             |              |
| Git push   | Compile    | Unit tests  | To staging   |
| PR merge   | Build      | Integration | To production|
| Tag        | container  | Security    | Canary       |
|            | image      | scan        | Blue-green   |
+----------+     +---------+     +----------+     +----------+
```

### Stage 1: Source

- Triggered by a Git event (push, PR, tag)
- Pipeline checks out the code

### Stage 2: Build

- Compile the application (if applicable)
- Build the container image (e.g., `docker build`)
- Push the image to a container registry

### Stage 3: Test

- **Unit tests** — test individual functions/methods
- **Integration tests** — test components working together
- **Security scans** — scan for vulnerabilities (SAST, image scanning)
- **Linting** — code quality checks

### Stage 4: Deploy

- Deploy to staging/production
- Use a deployment strategy (rolling, blue-green, canary)
- In a GitOps workflow: update the manifest repo (not deploy directly)

---

## CI/CD Tools

### Jenkins

- **Traditional**, widely-used CI/CD server
- **Plugin-based architecture** (1800+ plugins)
- Uses `Jenkinsfile` (Groovy DSL) to define pipelines
- Self-hosted, runs on any infrastructure
- Highly flexible but requires more maintenance

```
Jenkinsfile (Declarative Pipeline):

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'docker build -t myapp:latest .'
            }
        }
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        stage('Deploy') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
            }
        }
    }
}
```

### GitHub Actions

- **Native to GitHub** — no separate CI/CD server needed
- Workflows defined in **YAML** (`.github/workflows/`)
- Event-driven (push, PR, schedule, manual)
- Marketplace of reusable actions
- Hosted runners (or self-hosted)

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline
on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Push to registry
        run: docker push myapp:${{ github.sha }}

      - name: Deploy to K8s
        run: kubectl set image deployment/myapp myapp=myapp:${{ github.sha }}
```

### Tekton

- **Kubernetes-native CI/CD** — pipelines run as Kubernetes resources
- Pipelines, Tasks, and Steps are defined as **CRDs**
- Runs on any Kubernetes cluster
- Part of the **CD Foundation** (Linux Foundation)
- Cloud-agnostic, no vendor lock-in

```
+-----------------------------------------------------------------+
|                    TEKTON ARCHITECTURE                           |
|                                                                 |
|   +------------+     +------------+     +------------+          |
|   |   Task 1   |     |   Task 2   |     |   Task 3   |          |
|   |  (Build)   | --> |  (Test)    | --> |  (Deploy)  |          |
|   |            |     |            |     |            |          |
|   | Step 1:    |     | Step 1:    |     | Step 1:    |          |
|   |  checkout  |     |  run tests |     |  kubectl   |          |
|   | Step 2:    |     |            |     |  apply     |          |
|   |  build img |     |            |     |            |          |
|   +------------+     +------------+     +------------+          |
|                                                                 |
|   All Tasks run as Kubernetes Pods                              |
|   Each Step runs as a container within the Pod                  |
+-----------------------------------------------------------------+
```

**Tekton CRDs:**

| CRD            | Description                                    |
|----------------|------------------------------------------------|
| `Task`         | A set of steps (containers) that run in a Pod  |
| `Pipeline`     | A sequence of Tasks                            |
| `TaskRun`      | An execution of a Task                         |
| `PipelineRun`  | An execution of a Pipeline                     |

```yaml
# Tekton Task example
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-image
spec:
  steps:
    - name: build
      image: gcr.io/kaniko-project/executor:latest
      command:
        - /kaniko/executor
      args:
        - --dockerfile=Dockerfile
        - --destination=myregistry/myapp:latest
```

### Tool Comparison

| Feature            | Jenkins              | GitHub Actions       | Tekton               |
|--------------------|----------------------|----------------------|----------------------|
| **Architecture**   | Server-based         | SaaS / hosted        | Kubernetes-native    |
| **Config format**  | Jenkinsfile (Groovy) | YAML workflows       | CRDs (YAML)          |
| **Hosting**        | Self-hosted          | GitHub-hosted        | Self-hosted (on K8s) |
| **Extensibility**  | Plugins              | Marketplace actions  | Custom Tasks         |
| **K8s integration**| Via plugins          | Via actions          | Native (runs on K8s) |
| **Learning curve** | Steep                | Low                  | Moderate             |
| **Best for**       | Complex, legacy      | GitHub-based teams   | K8s-native pipelines |

---

## Integration with GitOps

In a GitOps workflow, CI/CD and GitOps work together but have distinct responsibilities:

```
+-------------------------------------------------------------------+
|                CI/CD + GITOPS INTEGRATION                         |
|                                                                   |
|   +------------------+                                           |
|   | App Source Repo   |  (application code)                      |
|   +--------+---------+                                           |
|            |                                                     |
|            | 1. Developer pushes code                            |
|            v                                                     |
|   +------------------+                                           |
|   | CI Pipeline      |                                           |
|   | - Build image    |                                           |
|   | - Run tests      |                                           |
|   | - Push to        |                                           |
|   |   registry       |                                           |
|   +--------+---------+                                           |
|            |                                                     |
|            | 2. Update image tag in config repo                  |
|            v                                                     |
|   +------------------+                                           |
|   | Config Repo      |  (Kubernetes manifests / Helm values)     |
|   | (GitOps repo)    |                                           |
|   +--------+---------+                                           |
|            |                                                     |
|            | 3. GitOps agent detects change (PULL)               |
|            v                                                     |
|   +------------------+                                           |
|   | ArgoCD / FluxCD  |                                           |
|   | (in-cluster)     |                                           |
|   +--------+---------+                                           |
|            |                                                     |
|            | 4. Deploys to cluster                               |
|            v                                                     |
|   +------------------+                                           |
|   | K8s Cluster      |                                           |
|   +------------------+                                           |
+-------------------------------------------------------------------+

CI handles:  Build, Test, Push image, Update config repo
GitOps handles:  Deploy, Reconcile, Drift detection
```

**Key separation:**
- The CI pipeline **never touches the cluster directly**
- The CI pipeline updates the **config repo** (e.g., changes an image tag)
- The GitOps agent (ArgoCD/FluxCD) handles actual deployment

---

## What to Remember for the Exam

1. **Continuous Integration (CI):** Automatically build and test code on every commit. Goal: catch bugs early, keep the main branch working.

2. **Continuous Delivery:** Code is always in a deployable state. Deployment to production requires a **manual trigger**.

3. **Continuous Deployment:** Fully automated — every change that passes the pipeline goes straight to production. **No manual approval.**

4. **Pipeline stages:** Source (Git event) --> Build (compile, build image) --> Test (unit, integration, security) --> Deploy.

5. **Jenkins:** Traditional, self-hosted, plugin-based CI/CD server. Uses `Jenkinsfile` (Groovy).

6. **GitHub Actions:** Native to GitHub, YAML-based workflows, event-driven, uses hosted runners.

7. **Tekton:** Kubernetes-native CI/CD. Pipelines and Tasks are **CRDs** that run as Pods. Part of the CD Foundation.

8. **CI/CD + GitOps:** CI builds and tests the app, then updates a config repo. The GitOps agent (ArgoCD/FluxCD) pulls from the config repo and deploys. **CI never deploys directly.**

9. **Two repos pattern:** Application source code repo (triggers CI) and config/manifest repo (watched by GitOps agent). This separation of concerns is a GitOps best practice.

10. **Tekton key CRDs:** `Task` (steps in a Pod), `Pipeline` (sequence of Tasks), `TaskRun` (execution of a Task), `PipelineRun` (execution of a Pipeline).
