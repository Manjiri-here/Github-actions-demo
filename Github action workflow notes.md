Notes of github actions workflow:

Think of a workflow as a pipeline of jobs. Each job is a box that runs on a runner (host). Jobs can run in parallel or depend on others (needs). Build the pipeline so:

- Cheap checks (lint/unit tests) run first and in parallel.

- Expensive steps (integration tests, security scans, image builds) run after cheap steps pass.

- Artifacts (test results, coverage reports, built image metadata) are passed between jobs via uploads/downloads or a registry.

- Final step is deployment, gated by environment protection or manual approval in production.

**Key concepts to use:**

jobs and needs (dependencies)

runs-on (ubuntu-latest vs self-hosted)

steps inside jobs (checkout, setup, run)

caching (actions/cache) for speed

artifacts (actions/upload-artifact) to pass files between jobs

secrets (DOCKERHUB, KUBECONFIG) stored in repo/org secrets

concurrency to prevent overlapping runs

env for job-level env vars

if: for conditional execution (e.g., only on main branch, only on tags)

matrix for testing multiple versions (Node/Java/Python)

**Step-by-step plan you should follow when writing a new pipeline**

Define scope: which branches, PRs, or tags trigger it? (e.g., push to main, pull_request for PRs).

Minimum pipeline: Make a tiny workflow that checks out code and runs tests. Get that green on PRs.

Add lint/static analysis: fast checks that fail the PR early.

Add unit tests and coverage: collect coverage artifacts.

Add security scans (SAST/Dependency): e.g., CodeQL, dependency-audit.

Add build and package (docker build). Use cache for speed.

Publish artifact/image to a registry.

Staging deployment: deploy to k8s cluster for further tests.

Production deployment: gated (manual approval / protected env).

Observability: add logs, notifications (Slack/email) for failures.

Optimize: parallelize jobs, cache dependencies, move heavy jobs to self-hosted runners if needed.

Always iterate. Don’t try to do everything in one commit. Get tests → then build → then deploy.

Secrets & environment you’ll need (store in GitHub repo settings → Secrets)

DOCKERHUB_USERNAME / DOCKERHUB_TOKEN (or registry user/token)

KUBECONFIG (base64-encoded kubeconfig or use cluster credentials via a GitHub Action that authenticates)

COVERAGE_TOKEN (if you upload to codecov/coveralls)

Any cloud provider credentials for login (AWS/GCP/Azure) if you use ECR/GCR/ACR

SLACK_WEBHOOK (optional for notifications)

Never hardcode secrets in your YAML.

**Full sample workflow**

This is a general, practical pipeline for a containerized app (Node/Python/Java — language agnostic). It:

Runs on PRs and pushes to main

Runs lint + unit tests in parallel

Runs CodeQL (security SAST)

Uploads coverage

Builds and pushes Docker image (on main)

Deploys to Kubernetes (on main)

Save as .github/workflows/ci-cd-k8s.yml.

name: CI/CD to Kubernetes

on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]
  workflow_dispatch: {}

concurrency:
  group: ci-cd-${{ github.ref }}            # github.ref = the branch or tag that triggered the workflow. We dont need to define this github.ref anywhere, github crewtes/picks it up automatically. You only reference it. GitHub replaces ${{ github.ref }} → refs/heads/main at runtime. Stops multiple workflows for the same branch from running at the same time → cancels earlier ones.
  
  cancel-in-progress: true                  # If a new pipeline starts on the same branch, GitHub cancels the older one.

env:                                                               # these are global variables, Defines variables for Docker image name and tag so all jobs use consistent naming → prevents mistakes.
  IMAGE_NAME: ghcr.io/${{ github.repository_owner }}/myapp         # github.repository_owner = your GitHub username or org
  IMAGE_TAG: ${{ github.sha }}                                     # github.sha = commit ID. So your image tag becomes the exact commit hash. So your image built will be: ghcr.io/your-username/myapp:d7e983f8c822ab53e913a2bdd5ab5380

jobs:
  # -------------------------
  # quick checks: lint + unit tests (parallel)
  # -------------------------
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps: 
      - uses: actions/checkout@v4                    
      - name: Setup Node (example)                  
        uses: actions/setup-node@v4                  # uses: means you are calling an existing action — basically someone else’s script packaged for GitHub. There is zero magic. It’s just: download this action, run its code, pass it inputs
        with: node-version: 18                       # with: is how you pass inputs to the action.It’s the equivalent of passing arguments to a function.
      - name: Install deps (cache)                   # There is no universal syntax for with: because it depends on the action you are using. Every action has its own README that defines: Allowed inputs, output variables, examples Use readme for each action before using it.
        uses: actions/cache@v4
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-
      - name: Install
        run: npm ci
      - name: Lint
        run: npm run lint

  unit-tests:
    name: Unit tests & Coverage
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node
        uses: actions/setup-node@v4
        with: node-version: 18
      - name: Install deps
        run: npm ci
      - name: Run tests with coverage
        run: npm test -- --coverage --coverageDirectory=coverage
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage

  # -------------------------
  # Static analysis / Security
  # -------------------------
  codeql:
    name: CodeQL Security Scan
    runs-on: ubuntu-latest
    permissions:                          # GitHub Actions workflows run with a token called GITHUB_TOKEN. By default, this token has broad permissions (read/write). This is a security risk. So workflows can override the token’s power using this. “Give this workflow only read access to Actions and repository contents. Nothing more.”
      actions: read      
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript
      - name: Autobuild
        uses: github/codeql-action/autobuild@v2
      - name: Run CodeQL queries
        uses: github/codeql-action/analyze@v2

  # -------------------------
  # Build docker image (only on main) and push to registry
  # -------------------------
  build-and-push:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: [unit-tests, codeql]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ env.IMAGE_TAG }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - name: Upload image-tag artifact
        uses: actions/upload-artifact@v4
        with:
          name: image-tag
          path: image-tag.txt
      - name: Save IMAGE TAG
        run: echo "${{ env.IMAGE_TAG }}" > image-tag.txt

  # -------------------------
  # Deploy to Kubernetes (staging or main)
  # -------------------------
  deploy:
    name: Deploy to Kubernetes
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging                                     # Says: “This job is deploying to the staging environment.”
      url: https://staging.example.com                  # “If this workflow is running on the main branch → treat this job as a staging deployment and attach the staging URL.”
    steps:
      - uses: actions/checkout@v4
      # download the image-tag artifact from previous job (example)
      - name: Download image-tag
        uses: actions/download-artifact@v4
        with:
          name: image-tag
          path: .
      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.27.0'
      - name: Configure Kubeconfig
        run: |
          echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig
          export KUBECONFIG=$PWD/kubeconfig
        shell: bash
      - name: Update image in deployment
        run: |
          IMAGE="${{ env.IMAGE_NAME }}:$(cat image-tag.txt)"
          kubectl -n mynamespace set image deployment/myapp myapp=$IMAGE
          kubectl -n mynamespace rollout status deployment/myapp --timeout=120s
      - name: Run smoke tests
        run: ./scripts/smoke-test.sh

  # Optionally: notify on failure (runs always)
  notify:
    name: Notify Slack on Failure
    runs-on: ubuntu-latest
    needs: [build-and-push, deploy]
    if: failure()
    steps:
      - name: Send slack message                    # “Run this job/step ONLY if a previous job/step in the workflow has failed.” failure() is a GitHub Actions built-in function that returns true when anything earlier failed. So this block is part of a failure-handling job.
        run: |
          curl -X POST -H 'Content-type: application/json' --data '{"text":"CI failed for ${{ github.repository }}"}' ${{ secrets.SLACK_WEBHOOK }}


Notes:

Replace npm steps with your language build/test commands.

For Docker registry, I used GitHub Container Registry as example; swap with DockerHub/AWS ECR and use proper login action & secrets.

KUBECONFIG should be stored as base64 of ~/.kube/config or use GitHub cloud provider actions to authenticate (e.g., aws-actions/amazon-eks-login for EKS).

**Why this structure (explain line-by-line thinking)**

lint first: cheap, fails fast.

unit-tests depends on lint: we don’t waste test compute if lint fails.

codeql can run in parallel because it doesn’t need a built image.

build-and-push requires tests + codeql and only runs on main — prevents unintentional image publish for feature branches.

deploy needs build-and-push — prevents deploying until you have a new image.

Artifacts: we pass the image tag via artifact to deployment job to ensure the exact image is deployed.

concurrency: prevents multiple concurrent deployments for same ref.

if: github.ref == 'refs/heads/main' prevents pushing images for PRs.

**Practical tips & best practices (from experience)**

Use actions/cache for language deps to speed builds.

Fail fast: run lint/test early and in parallel.

Use upload-artifact for test reports & coverage so you can view them on the run page.

Set timeouts for long commands using timeout or job-level timeout-minutes.

Use environments in GitHub for protected deployments (require reviewers for production).

Use GITHUB_TOKEN for ghcr login or dedicated registry tokens.

Prefer specific action versions (use @v4 instead of @master).

Test locally with act to iterate faster (note: not all actions work locally).

Use self-hosted runners for heavy builds or to access private infra (e.g., on-prem Kubernetes).

Use matrix to test multiple runtime versions (Node 16/18/20).

Set up branch protection rules so PR cannot be merged until CI passes.

Keep workflows modular — consider reusable workflows once you have multiple repos.

**Exercises to build skill quickly (do in this order)**

Create a minimal workflow: checkout + echo hello.

Add unit tests and run on PRs.

Add caching for dependencies.

Add lint job that runs in parallel.

Add Docker build + push to your personal registry on main.

Add a deploy job that updates a Kubernetes deployment (use a test cluster or k3d).

Add CodeQL and a dependency vulnerability scan.

Add manual approval for production environment deploys.

Convert repeated logic into a reusable workflow and call it from another repo.

Do these one at a time and validate on GitHub Actions UI.

**Debugging workflows — fast checklist**

If an action fails: open the job log, expand the failing step, copy the exact command and run locally or in a runner shell.

Use echo / env to dump variables (be careful not to leak secrets).

If you see authentication errors: confirm secrets and token scopes.

If kubectl fails: check kubeconfig contents and network access from runner.

If builds are slow: inspect cache hit/miss lines in logs.

For flaky tests: record test artifacts and re-run job with retries or isolate flaky tests.

If you want, next I’ll do one of these (pick one — I’ll produce code/config ready to run):

Tailor the sample workflow to your stack (Node/Python/Java + exact commands).

Create a deploy-to-k8s job that uses GitHub OIDC to authenticate to AWS EKS (no kubeconfig secret).

Convert the workflow into reusable workflows (for mono-repo / multi-repo reuse).

Give you a step-by-step lab using k3d on your laptop to test deployments locally.
