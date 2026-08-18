# Portfolio — Docker + GitHub Actions CI/CD

Production-style DevOps deployment pipeline for a static portfolio application using **Docker, Nginx, GitHub Actions, and a self-hosted GitHub Actions runner running on Docker Desktop**.

The pipeline automatically builds, validates, tests, versions, and deploys the portfolio whenever changes are pushed to the `main` branch.

---

## Architecture

```text
                         GitHub Repository
                                │
                         git push → main
                                │
                                ▼
                       GitHub Actions
                                │
                                ▼
                 ┌─────────────────────────┐
                 │ Self-hosted Runner      │
                 │ Docker Container        │
                 │                         │
                 │ GitHub Actions Runner   │
                 └────────────┬────────────┘
                              │
                              │ Docker Engine
                              ▼
                     ┌──────────────────┐
                     │  Docker Desktop  │
                     │                  │
                     │  Build           │
                     │  Test            │
                     │  Deploy          │
                     └────────┬─────────┘
                              │
                              ▼
                  ┌──────────────────────┐
                  │ vignesh-portfolio    │
                  │                      │
                  │ Nginx                │
                  │ Port 80              │
                  └──────────┬───────────┘
                             │
                      Host Port 8085
                             │
                             ▼
                  Nginx Proxy Manager
                             │
                             ▼
                    Cloudflare Tunnel
                             │
                             ▼
                  Portfolio Domain
```

---

# Project Structure

```text
Portfolio/
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── assets/
│   ├── css/
│   │   └── style.css
│   │
│   ├── js/
│   │   ├── data.js
│   │   ├── main.js
│   │   └── chat-widget.js
│   │
│   ├── images/
│   │   ├── profile.jpg
│   │   ├── vs-logo.ico
│   │   └── chatbot_icon.png
│   │
│   ├── guides/
│   └── resume/
│
├── tests/
│   └── smoke-test.ps1
│
├── index.html
├── Dockerfile
├── nginx.conf
├── docker-compose.yml
├── .dockerignore
├── .gitignore
└── README.md
```

---

# Technology Stack

| Component           | Purpose                      |
| ------------------- | ---------------------------- |
| HTML5               | Portfolio structure          |
| CSS3                | Styling                      |
| JavaScript          | Frontend functionality       |
| Nginx               | Production web server        |
| Docker              | Application containerization |
| Docker Desktop      | Local container runtime      |
| Git                 | Source control               |
| GitHub              | Source repository            |
| GitHub Actions      | CI/CD automation             |
| Self-hosted Runner  | Executes CI/CD locally       |
| Nginx Proxy Manager | Reverse proxy                |
| Cloudflare Tunnel   | Secure external access       |

---

# CI/CD Pipeline

Every push to `main` triggers the deployment workflow.

```text
Developer
    │
    │ git push
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Checkout
    │
    ├── Verify Docker
    │
    ├── Validate project structure
    │
    ├── Build Docker image
    │
    ├── Validate Nginx configuration
    │
    ├── Start test container
    │
    ├── Wait for application
    │
    ├── Run smoke tests
    │
    ├── Remove test container
    │
    ├── Deploy production container
    │
    └── Verify production deployment
    │
    ▼
Docker Desktop
    │
    ▼
Portfolio
```

---

# GitHub Actions Workflow

The workflow is located at:

```text
.github/workflows/deploy.yml
```

The workflow runs on:

```yaml
on:
  push:
    branches:
      - main

  workflow_dispatch:
```

This means deployment happens automatically after a push to `main`.

It can also be started manually from:

```text
GitHub
→ Actions
→ Portfolio CI/CD
→ Run workflow
```

---

# Self-hosted Runner

The deployment uses a self-hosted GitHub Actions runner.

Instead of deploying to GitHub-hosted infrastructure, the workflow executes on the local Docker Desktop environment.

The intended architecture is:

```text
GitHub
   │
   ▼
Self-hosted Runner
   │
   ▼
Docker Desktop
   │
   ▼
Portfolio Container
```

The runner remains online and waits for GitHub Actions jobs.

When code is pushed:

```text
Git push
   ↓
GitHub detects change
   ↓
GitHub Actions creates job
   ↓
Self-hosted runner receives job
   ↓
CI/CD pipeline executes
```

---

# Docker Image Versioning

Every GitHub Actions workflow run receives a sequential run number.

The image is tagged using:

```text
v${{ github.run_number }}
```

Example:

```text
vignesh-portfolio:v1
vignesh-portfolio:v2
vignesh-portfolio:v3
vignesh-portfolio:v4
```

The same build also receives:

```text
vignesh-portfolio:latest
```

Example:

```text
vignesh-portfolio:v42
vignesh-portfolio:latest
```

This provides a clear relationship between deployments and GitHub Actions runs.

---

# Why Version Tags Are Used

Sequential image tags make it easier to identify deployments.

For example:

```text
vignesh-portfolio:v41
vignesh-portfolio:v42
vignesh-portfolio:v43
```

If `v43` introduces a problem:

```text
v43 → problematic deployment
v42 → previous deployment
```

The image version can then be correlated with the GitHub Actions workflow run and Git commit.

The Docker image can be inspected with:

```powershell
docker image ls vignesh-portfolio
```

Example:

```text
REPOSITORY           TAG       IMAGE ID
vignesh-portfolio    v43       ...
vignesh-portfolio    v42       ...
vignesh-portfolio    v41       ...
latest               latest    ...
```

---

# Docker Build

The pipeline builds the application using:

```text
Dockerfile
```

The image is based on:

```dockerfile
FROM nginx:alpine
```

The default Nginx website is removed and the portfolio files are copied into:

```text
/usr/share/nginx/html
```

The custom Nginx configuration is copied to:

```text
/etc/nginx/conf.d/default.conf
```

---

# Nginx Configuration

Nginx provides the production web-server layer.

The configuration handles:

* Static file serving
* SPA fallback
* Browser caching
* Gzip compression
* Security headers
* Hidden-file protection
* Sensitive-file protection
* Custom error handling
* Server version hiding

The container listens on:

```text
80
```

---

# Port Mapping

The portfolio container exposes Nginx on port `80`.

Docker maps the host port `8085` to container port `80`:

```text
127.0.0.1:8085 → container:80
```

Therefore:

```text
http://localhost:8085
```

accesses the portfolio locally.

The Docker configuration is conceptually:

```yaml
ports:
  - "127.0.0.1:8085:80"
```

This keeps the portfolio bound to the local machine rather than exposing port `8085` directly to the LAN.

---

# Test Environment

The pipeline does not immediately replace production.

A temporary test container is started first:

```text
Portfolio image
     │
     ▼
Test container
     │
     └── 127.0.0.1:18080 → 80
```

The workflow then checks:

```text
HTTP response
HTML content
Nginx configuration
Security headers
```

If testing fails:

```text
Build
  ↓
Test
  ↓
FAIL
  ↓
Deployment stops
```

Production is not intentionally replaced by the failed build.

---

# Smoke Testing

Smoke tests are located at:

```text
tests/smoke-test.ps1
```

The test verifies that the application:

1. Starts successfully
2. Returns HTTP `200`
3. Contains valid HTML content
4. Provides required security headers

Example:

```powershell
.\tests\smoke-test.ps1 -Url http://127.0.0.1:18080
```

Successful result:

```text
HTTP status: 200
index.html validation passed
Security headers validation passed
Smoke test PASSED
```

---

# Production Deployment

After successful testing, the existing portfolio container is replaced.

```text
Existing container
       │
       ▼
Stop + remove
       │
       ▼
Start new image
       │
       ▼
Health verification
```

The production image is:

```text
vignesh-portfolio:v<GitHub Run Number>
```

Example:

```text
vignesh-portfolio:v43
```

---

# Production Verification

After deployment, GitHub Actions checks:

```text
http://127.0.0.1:8085/
```

The workflow waits for the container to become available.

If HTTP `200` is returned:

```text
Production deployment successful
```

If the application does not become available:

```text
Deployment verification failed
```

Container logs are then displayed for troubleshooting.

---

# Complete Deployment Flow

```text
                    git push main
                         │
                         ▼
                  GitHub Repository
                         │
                         ▼
                  GitHub Actions
                         │
                         ▼
                 Self-hosted Runner
                         │
                         ▼
                Validate project files
                         │
                         ▼
                   Docker Build
                         │
                         ▼
               vignesh-portfolio:v43
                         │
                         ▼
                Nginx configuration
                    validation
                         │
                         ▼
                 Test Container
                    :18080
                         │
                         ▼
                  Smoke Testing
                         │
                    ┌────┴────┐
                    │         │
                   FAIL      PASS
                    │         │
                    ▼         ▼
                  STOP      Deploy
                              │
                              ▼
                    vignesh-portfolio:v43
                              │
                              ▼
                       :8085 → :80
                              │
                              ▼
                    Production Verification
                              │
                              ▼
                         Application
```

---

# External Access

The portfolio is not directly exposed through Docker's public interface.

The intended production path is:

```text
Internet
   │
   ▼
Cloudflare
   │
   ▼
Cloudflare Tunnel
   │
   ▼
Nginx Proxy Manager
   │
   ▼
127.0.0.1:8085
   │
   ▼
Portfolio Container
   │
   ▼
Nginx :80
```

This separates:

* Public edge access
* TLS/domain handling
* Reverse proxy
* Application container

---

# Useful Docker Commands

## Check running portfolio

```powershell
docker ps --filter "name=vignesh-portfolio"
```

## Check all portfolio containers

```powershell
docker ps -a --filter "name=vignesh-portfolio"
```

## Check images

```powershell
docker image ls vignesh-portfolio
```

## View logs

```powershell
docker logs vignesh-portfolio
```

## Follow logs

```powershell
docker logs -f vignesh-portfolio
```

## Inspect container

```powershell
docker inspect vignesh-portfolio
```

## Check port mapping

```powershell
docker port vignesh-portfolio
```

Expected:

```text
80/tcp -> 127.0.0.1:8085
```

## Test locally

```powershell
curl http://127.0.0.1:8085
```

---

# Troubleshooting

## Port 8085 already allocated

Check which container is using the port:

```powershell
docker ps --filter "publish=8085"
```

Check Windows:

```powershell
Get-NetTCPConnection -LocalPort 8085 -ErrorAction SilentlyContinue
```

---

## Portfolio container is not running

Check:

```powershell
docker ps -a --filter "name=vignesh-portfolio"
```

Then:

```powershell
docker logs vignesh-portfolio
```

---

## GitHub Actions waiting for runner

Check:

```text
GitHub
→ Repository
→ Settings
→ Actions
→ Runners
```

The runner should show:

```text
Online
```

If it shows:

```text
Offline
```

the workflow cannot execute on the self-hosted runner.

---

# Security Considerations

The self-hosted runner has access to the local Docker environment.

Therefore:

* Only trusted workflow code should execute on the runner.
* Deployment is restricted to the `main` branch.
* Pull requests from untrusted sources should not automatically execute deployment jobs.
* Secrets should never be committed to Git.
* `.env` files should be excluded through `.gitignore`.
* Production services should not unnecessarily expose host ports.
* Docker Engine access from the runner should be treated as privileged access.

---

# Future Improvements

The current pipeline provides:

```text
Source Control
       +
CI
       +
Docker Build
       +
Automated Testing
       +
Versioned Images
       +
CD
       +
Health Verification
```

Planned improvements:

* [ ] Automatic rollback
* [ ] Blue/green deployment
* [ ] Docker image vulnerability scanning with Trivy
* [ ] Secret scanning with Gitleaks
* [ ] HTML validation
* [ ] JavaScript linting
* [ ] Broken-link testing
* [ ] Lighthouse performance testing
* [ ] Accessibility testing
* [ ] Prometheus monitoring
* [ ] Grafana dashboard
* [ ] Deployment notifications
* [ ] Deployment history
* [ ] Container health monitoring

---

# DevOps Concepts Demonstrated

This project demonstrates practical experience with:

```text
Git
GitHub
GitHub Actions
Self-hosted CI/CD
Docker
Docker Desktop
Docker image versioning
Container health checks
Automated smoke testing
Nginx
Reverse proxy
Security headers
Caching
Compression
Cloudflare Tunnel
Infrastructure automation
Deployment verification
```

---

# Deployment Philosophy

The pipeline follows:

```text
Build → Test → Deploy → Verify
```

rather than:

```text
Build → Deploy
```

The objective is to ensure that a new application version is validated before it replaces the currently deployed version.

Each deployment is associated with a GitHub Actions run number and Docker image version, providing traceability between:

```text
Git commit
    ↓
GitHub Actions run
    ↓
Docker image version
    ↓
Production container
```

Example:

```text
GitHub Actions Run #43
        ↓
vignesh-portfolio:v43
        ↓
Production
        ↓
127.0.0.1:8085
```

This makes deployment troubleshooting and rollback easier.

---

# Project Goal

This portfolio is not only a frontend project.

It is also a practical demonstration of a complete local DevOps delivery pipeline:

```text
Developer
   ↓
Git
   ↓
GitHub
   ↓
GitHub Actions
   ↓
Self-hosted Runner
   ↓
Docker
   ↓
Automated Testing
   ↓
Versioned Image
   ↓
Production Deployment
   ↓
Nginx
   ↓
Nginx Proxy Manager
   ↓
Cloudflare Tunnel
   ↓
Public Portfolio
```

The infrastructure runs locally on Docker Desktop while GitHub provides source control and CI/CD orchestration.
