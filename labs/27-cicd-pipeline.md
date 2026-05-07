# Lab 27: CI/CD Pipelines

## Objective

Build a CI/CD pipeline using GitHub Actions that builds a container image and (optionally) deploys to Kubernetes. Also explore Tekton pipeline concepts for Kubernetes-native CI/CD.

## Prerequisites

- A GitHub account
- Basic understanding of Docker and container images
- For the Tekton section: a running Kubernetes cluster with `kubectl`

---

## Part 1: GitHub Actions CI/CD Pipeline

### Exercise 1: Create a Simple Application

### Step 1: Create a new GitHub repository

Go to https://github.com/new and create a repo called `cicd-lab`. Clone it locally:

```bash
git clone https://github.com/<your-username>/cicd-lab.git
cd cicd-lab
```

### Step 2: Create a simple application

```bash
cat <<'EOF' > app.py
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        self.wfile.write(b"Hello from CI/CD Lab!\n")

if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", 8080), Handler)
    print("Server running on port 8080")
    server.serve_forever()
EOF
```

### Step 3: Create a Dockerfile

```bash
cat <<'EOF' > Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
EOF
```

### Step 4: Create a simple test script

```bash
cat <<'EOF' > test_app.py
import unittest

class TestApp(unittest.TestCase):
    def test_server_response(self):
        """Basic test to verify the app module loads"""
        import app
        self.assertTrue(hasattr(app, 'Handler'))

if __name__ == "__main__":
    unittest.main()
EOF
```

### Step 5: Create Kubernetes manifests

```bash
mkdir -p k8s

cat <<'EOF' > k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cicd-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cicd-demo
  template:
    metadata:
      labels:
        app: cicd-demo
    spec:
      containers:
        - name: app
          image: IMAGE_PLACEHOLDER
          ports:
            - containerPort: 8080
EOF

cat <<'EOF' > k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: cicd-demo
spec:
  selector:
    app: cicd-demo
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
EOF
```

---

### Exercise 2: Create a GitHub Actions Workflow

### Step 1: Create the workflow directory

```bash
mkdir -p .github/workflows
```

### Step 2: Create the CI pipeline

This workflow runs on every push: it checks out the code, runs tests, and builds the container image.

```bash
cat <<'EOF' > .github/workflows/ci.yml
name: CI Pipeline

# Trigger on push to main or on pull requests
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Run tests
        run: python -m pytest test_app.py -v || python -m unittest test_app.py -v

  build:
    name: Build Container Image
    runs-on: ubuntu-latest
    needs: test  # Only runs if tests pass
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build image
        run: |
          docker build -t cicd-demo:${{ github.sha }} .
          echo "Image built successfully: cicd-demo:${{ github.sha }}"

      # In a real pipeline, you would push to a registry:
      # - name: Log in to container registry
      #   uses: docker/login-action@v3
      #   with:
      #     registry: ghcr.io
      #     username: ${{ github.actor }}
      #     password: ${{ secrets.GITHUB_TOKEN }}
      #
      # - name: Push image
      #   run: |
      #     docker tag cicd-demo:${{ github.sha }} ghcr.io/${{ github.repository }}/cicd-demo:${{ github.sha }}
      #     docker push ghcr.io/${{ github.repository }}/cicd-demo:${{ github.sha }}
EOF
```

### Step 3: Create a full CI/CD pipeline (with deploy stage)

This is a more complete pipeline that includes a deploy step. The deploy step is commented out since it requires cluster credentials.

```bash
cat <<'EOF' > .github/workflows/cicd.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  # ──────────────────────────────────────────────
  # Stage 1: TEST
  # ──────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: python -m unittest test_app.py -v

  # ──────────────────────────────────────────────
  # Stage 2: BUILD & PUSH
  # ──────────────────────────────────────────────
  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: test
    # permissions:
    #   packages: write  # Needed to push to GHCR
    steps:
      - uses: actions/checkout@v4

      - name: Build container image
        run: docker build -t cicd-demo:${{ github.sha }} .

      # Uncomment to push to GitHub Container Registry:
      # - uses: docker/login-action@v3
      #   with:
      #     registry: ghcr.io
      #     username: ${{ github.actor }}
      #     password: ${{ secrets.GITHUB_TOKEN }}
      # - run: |
      #     docker tag cicd-demo:${{ github.sha }} ghcr.io/${{ github.repository }}/cicd-demo:${{ github.sha }}
      #     docker push ghcr.io/${{ github.repository }}/cicd-demo:${{ github.sha }}

  # ──────────────────────────────────────────────
  # Stage 3: DEPLOY (GitOps approach)
  # ──────────────────────────────────────────────
  # In a GitOps workflow, the deploy step updates the
  # config repo with the new image tag. The GitOps agent
  # (ArgoCD/FluxCD) then deploys it.
  #
  # deploy:
  #   name: Update Config Repo
  #   runs-on: ubuntu-latest
  #   needs: build
  #   steps:
  #     - name: Checkout config repo
  #       uses: actions/checkout@v4
  #       with:
  #         repository: <your-username>/argocd-lab-config
  #         token: ${{ secrets.CONFIG_REPO_TOKEN }}
  #
  #     - name: Update image tag
  #       run: |
  #         sed -i "s|image: .*|image: ghcr.io/${{ github.repository }}/cicd-demo:${{ github.sha }}|" k8s/deployment.yaml
  #
  #     - name: Commit and push
  #       run: |
  #         git config user.name "GitHub Actions"
  #         git config user.email "actions@github.com"
  #         git add .
  #         git commit -m "Update image to ${{ github.sha }}"
  #         git push
EOF
```

### Step 4: Push everything to GitHub

```bash
git add .
git commit -m "Add application, Dockerfile, tests, and CI/CD workflows"
git push origin main
```

### Step 5: Watch the pipeline run

1. Go to your GitHub repo in the browser
2. Click the **Actions** tab
3. You should see the workflow running
4. Click on it to see the stages: Test --> Build

### Step 6: Understand the pipeline flow

```
+------------------------------------------------------------------+
|                  GITHUB ACTIONS PIPELINE                         |
|                                                                  |
|  Trigger: push to main                                          |
|       |                                                          |
|       v                                                          |
|  +----------+     +-----------+     +-------------------+        |
|  |   TEST   | --> |   BUILD   | --> |  DEPLOY (GitOps)  |        |
|  |          |     |           |     |                   |        |
|  | - python |     | - docker  |     | - update config   |        |
|  |   tests  |     |   build   |     |   repo with new   |        |
|  |          |     | - push to |     |   image tag       |        |
|  |          |     |   registry|     | - ArgoCD syncs    |        |
|  +----------+     +-----------+     +-------------------+        |
|                                                                  |
+------------------------------------------------------------------+
```

---

## Part 2: Tekton Concepts

Tekton is a Kubernetes-native CI/CD framework. Instead of running pipelines on a separate server, Tekton runs each task as a Pod in your cluster.

### Exercise 3: Install Tekton (Optional Hands-On)

### Step 1: Install Tekton Pipelines

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

### Step 2: Wait for Tekton to be ready

```bash
kubectl wait --for=condition=Ready pods --all -n tekton-pipelines --timeout=120s
```

### Step 3: Install the Tekton CLI (tkn)

**macOS:**

```bash
brew install tektoncd-cli
```

**Linux:**

```bash
curl -LO https://github.com/tektoncd/cli/releases/latest/download/tkn_Linux_x86_64.tar.gz
tar xvzf tkn_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn
```

---

### Exercise 4: Create a Tekton Task

A Tekton **Task** defines a series of steps that run in containers within a single Pod.

### Step 1: Create a simple "hello" Task

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello-task
spec:
  steps:
    - name: say-hello
      image: alpine:3.18
      command:
        - echo
      args:
        - "Hello from Tekton!"

    - name: show-date
      image: alpine:3.18
      command:
        - date
EOF
```

### Step 2: Run the Task

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: hello-task-run
spec:
  taskRef:
    name: hello-task
EOF
```

Or using the CLI:

```bash
tkn task start hello-task --showlog
```

### Step 3: View the results

```bash
tkn taskrun logs hello-task-run
# or
kubectl logs hello-task-run-pod -c step-say-hello
```

---

### Exercise 5: Create a Tekton Pipeline

A **Pipeline** chains multiple Tasks together.

### Step 1: Create a build Task

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-task
spec:
  steps:
    - name: build
      image: alpine:3.18
      command: ["/bin/sh"]
      args:
        - -c
        - |
          echo "=== Building application ==="
          echo "Running: docker build -t myapp:latest ."
          echo "Build complete!"
EOF
```

### Step 2: Create a test Task

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: test-task
spec:
  steps:
    - name: test
      image: alpine:3.18
      command: ["/bin/sh"]
      args:
        - -c
        - |
          echo "=== Running tests ==="
          echo "Test 1: PASSED"
          echo "Test 2: PASSED"
          echo "Test 3: PASSED"
          echo "All tests passed!"
EOF
```

### Step 3: Create a deploy Task

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-task
spec:
  steps:
    - name: deploy
      image: alpine:3.18
      command: ["/bin/sh"]
      args:
        - -c
        - |
          echo "=== Deploying to cluster ==="
          echo "Running: kubectl apply -f deployment.yaml"
          echo "Deployment complete!"
EOF
```

### Step 4: Create the Pipeline

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: ci-pipeline
spec:
  tasks:
    - name: build
      taskRef:
        name: build-task

    - name: test
      taskRef:
        name: test-task
      runAfter:
        - build

    - name: deploy
      taskRef:
        name: deploy-task
      runAfter:
        - test
EOF
```

This creates a pipeline: **build --> test --> deploy**

### Step 5: Run the Pipeline

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: ci-pipeline-run
spec:
  pipelineRef:
    name: ci-pipeline
EOF
```

Or using the CLI:

```bash
tkn pipeline start ci-pipeline --showlog
```

### Step 6: View the Pipeline results

```bash
tkn pipelinerun logs ci-pipeline-run
```

You should see output from all three tasks in order.

### Step 7: Understand the Tekton resource model

```
+------------------------------------------------------------------+
|                   TEKTON RESOURCE MODEL                          |
|                                                                  |
|   Pipeline                                                       |
|   +------------------------------------------------------------+ |
|   |                                                            | |
|   |  Task: build          Task: test          Task: deploy     | |
|   |  +---------------+   +---------------+   +--------------+ | |
|   |  | Step: build   |-->| Step: test    |-->| Step: deploy | | |
|   |  | (runs in a    |   | (runs in a    |   | (runs in a   | | |
|   |  |  container)   |   |  container)   |   |  container)  | | |
|   |  +---------------+   +---------------+   +--------------+ | |
|   |                                                            | |
|   +------------------------------------------------------------+ |
|                                                                  |
|   Each Task = a Pod                                              |
|   Each Step = a container within the Pod                        |
|   Pipeline = ordered sequence of Tasks                          |
+------------------------------------------------------------------+
```

---

## Cleanup

### GitHub Actions

No cleanup needed — the workflows run on GitHub-hosted runners.

### Tekton

```bash
kubectl delete pipelinerun ci-pipeline-run
kubectl delete pipeline ci-pipeline
kubectl delete taskrun hello-task-run
kubectl delete task hello-task build-task test-task deploy-task
kubectl delete -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

### Delete the test repo

Remove the `cicd-lab` repository from GitHub if you no longer need it.

---

## Review Questions

1. What is the difference between Continuous Delivery and Continuous Deployment?
   - **Continuous Delivery** requires a manual trigger to deploy to production. **Continuous Deployment** is fully automated.

2. What are the typical stages of a CI/CD pipeline?
   - **Source --> Build --> Test --> Deploy**

3. Where are GitHub Actions workflows defined?
   - In `.github/workflows/` as **YAML files**

4. What is Tekton and how does it differ from Jenkins?
   - Tekton is **Kubernetes-native** CI/CD — pipelines run as CRDs/Pods in the cluster. Jenkins is a traditional server-based CI/CD tool.

5. In a GitOps-integrated CI/CD pipeline, what does the CI pipeline do instead of deploying directly?
   - It **updates the config repo** with the new image tag. The GitOps agent (ArgoCD/FluxCD) handles the actual deployment.

6. What are the key Tekton CRDs?
   - **Task** (steps in a Pod), **Pipeline** (sequence of Tasks), **TaskRun** (execution of a Task), **PipelineRun** (execution of a Pipeline)
