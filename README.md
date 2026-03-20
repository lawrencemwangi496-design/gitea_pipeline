# Self-Hosted Gitea with CI/CD Runners

A production-ready self-hosted Git service with integrated CI/CD infrastructure, built as a foundational DevOps toolchain.

## Overview

This repository contains the complete infrastructure as code for deploying a self-hosted Gitea instance with:
- PostgreSQL database backend
- Three parallel Gitea Actions runners for CI/CD workloads
- Automated security scanning pipeline for infrastructure code
- Production deployment to VPS infrastructure

**Status:** Running in production on personal VPS, actively used for CI/CD pipelines.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Host (VPS)                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  PostgreSQL  │  │    Gitea     │  │   Runners    │     │
│  │   (15)       │◄─┤   (latest)   │  │   (x3)       │     │
│  │              │  │              │  │              │     │
│  │ - Database   │  │ - Web UI     │  │ - Job exec   │     │
│  │ - State      │  │ - API        │  │ - Docker     │     │
│  │ - Health     │  │ - SSH        │  │   socket     │     │
│  │   checks     │  │              │  │              │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│         │                 │                   │            │
│         └─────────────────┼───────────────────┘            │
│                           │                                │
│                     gitea_net (bridge)                     │
└─────────────────────────────────────────────────────────────┘
```

### Components

| Component | Image | Purpose |
|-----------|-------|---------|
| PostgreSQL | `postgres:15` | Persistent storage for Gitea metadata, users, repositories |
| Gitea | `gitea/gitea:latest` | Git repository hosting, web interface, API |
| Runners (3x) | `gitea/act_runner:latest` | Execute CI/CD jobs (parallel, load-balanced) |

### Networking

- All services communicate over `gitea_net` bridge network
- Gitea exposed on `127.0.0.1:3000` (reverse proxy in production)
- Runners mount Docker socket for container-based job execution
- Persistent data stored in `/opt/gitea/` (host bind mounts)

## Deployment

### Prerequisites

- Docker Engine 20.10+
- Docker Compose v2+
- Domain with DNS configured (for HTTPS)
- Reverse proxy (nginx/Caddy) for SSL termination

### Environment Configuration

Copy `.env.example` to `.env` and configure:

```bash
POSTGRES_USER=gitea
POSTGRES_DB=giteadb
POSTGRES_PASSWORD=<secure-password>
GITEA_RUNNER_TOKEN=<registration-token-from-gitea-ui>
```

**Getting the runner token:**
1. Access Gitea web UI after deployment
2. Navigate to Site Administration → Actions → Runners
3. Create new runner, copy registration token

### Start Services

```bash
# Deploy all services
docker compose up -d

# Verify health
docker compose ps

# Check logs
docker compose logs -f
```

### Post-Deployment

1. Access Gitea at `http://your-server:3000`
2. Complete initial setup (admin user, site config)
3. Configure reverse proxy for HTTPS
4. Register runners using the token from step 2

## CI/CD Pipeline (GitHub Actions)

This repository includes GitHub Actions workflows that **scan the infrastructure code itself** — implementing DevSecOps practices before deployment.

### Security Scan Pipeline

**Trigger:** PR to main, daily at 7am UTC, manual dispatch

**Jobs:**

| Job | Tool | Purpose |
|-----|------|---------|
| `secret_scan` | TruffleHog | Detect hardcoded secrets before they reach production |
| `syntax` | `docker compose config` | Validate compose file syntax |
| `image_scan` | Trivy | Scan container images (postgres, gitea, runners) for vulnerabilities |

**Workflow logic:**
- All jobs run on PRs and scheduled scans
- Manual dispatch allows scoped scans (secret_scan, syntax, image_scan, full_scan)
- Image scan depends on successful syntax + secret scan
- Fails on HIGH/CRITICAL vulnerabilities

### Deployment Pipeline

Triggered on push to main, deploys configuration to production VPS.

## Security Features

### Implemented
- **Secret scanning** in CI — prevents credential leaks
- **Container vulnerability scanning** — Trivy checks base images
- **Health checks** on PostgreSQL — ensures database readiness
- **Separate network** — service isolation
- **Environment variables** — secrets not hardcoded in compose
- **Scheduled security scans** — weekly automated checks

### Lessons Learned

**Runner capacity:** Three runners provide redundancy but resource constraints (VPS specs) limit concurrent job execution. Runner labels allow targeted job assignment.

**Local images:** Trivy failures occurred with locally-built OpenClaw images due to missing vulnerability database layers. Solution: push to registry first, then scan.

**Testing methodology:** Intentional failures (broken code, resource limits) helped understand system boundaries and failure modes before production deployment.

## Future Improvements

- [ ] Add monitoring stack (Prometheus + Grafana)
- [ ] Implement automated backups for PostgreSQL
- [ ] Add staging environment for testing deployment changes
- [ ] Create example Gitea Actions workflows for common tasks
- [ ] Add OAuth/SSO integration

## Learning Outcomes

This project demonstrates:
- **Infrastructure as Code:** Docker Compose defines complete, reproducible stack
- **CI/CD Pipeline Security:** Scanning infrastructure before deployment
- **Self-Hosted Toolchains:** Independence from SaaS platforms
- **Production Mindset:** Health checks, persistent storage, redundancy
- **DevSecOps Integration:** Security tools embedded in development workflow

## References

- [Gitea Documentation](https://docs.gitea.com/)
- [Gitea Actions](https://docs.gitea.com/usage/actions/overview)
- [Trivy Vulnerability Scanner](https://trivy.dev/)
- [TruffleHog Secret Scanner](https://trufflesecurity.com/trufflehog/)

---

*Built in two days as part of DevSecOps learning journey. Deployed and operational on personal VPS.*
