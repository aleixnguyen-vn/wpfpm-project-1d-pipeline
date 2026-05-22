# My Automated WordPress Stack (From Manual Configuration to CI/CD)

## 📌 So what's up with this repo?
This is a personal lab where I migrated a legacy WordPress stack from standard hand-guided server setups ("ClickOps") to a fully automated pipeline using **Terraform** and **GitHub Actions**. 

To be honest, a year ago, I used to log into servers via SSH and manually type every single Docker command, or use panel interfaces to configure everything. It worked, but it was a nightmare to track changes or reproduce if the server went down. This repo is my practical journey to fix that bad habit by defining everything as code.

## 🏗️ Architecture & Stack Overview

The infrastructure isolates core components into dedicated, lightweight containers managed by Docker Compose, optimized to deliver high concurrency under low resource constraints (1GB RAM). I pick some alpine version for Redis and NGINX for better performance

*   **Infrastructure:** Digital Ocean Droplet (1 vCPU, 1GB RAM) provisioned via Terraform.
*   **Web Server:** Nginx acting as a reverse proxy and routing traffic to PHP-FPM.
*   **Application:** WordPress running on PHP-FPM for optimal processing speed.
*   **Database:** MariaDB (relational storage) optimized with structured environment variables.
*   **Caching Layer:** Redis integration via Object Cache Plugin to reduce database bottlenecks.

---

## 🚀 CI/CD Pipeline Workflow (`needs:` DAG Execution)

The GitHub Actions pipeline is architected into two sequential, dependent stages to ensure strict **Idempotency** and environmental safety.

```text
[Code being 'Push' to main] 
        │
        ▼
┌────────────────────────────────┐
│      Job 1: bootstrap-env      │  ◄── Hardens OS & check and installs Docker/Compose if needed
└────────────────────────────────┘
        │
        ▼ (needs: [bootstrap-env])
┌────────────────────────────────┐
│        Job 2: deploy           │  ◄── Securely syncs manifests, builds & launches stack
└────────────────────────────────┘
```
### Phase 1: Environment Bootstrapping (bootstrap-env)

* Establishes an automated SSH connection to the fresh Droplet.

* Updates OS on the new Droplet. Verifies, installs, and configures the Docker Engine and Docker Compose Plugin only if they are missing,  ensuring zero configuration drift.

### Phase 2: Automated Deployment (deploy)

* Triggered only after Phase 1 succeeds via the needs: constraint.

* Uses secure SCP to sync docker-compose.yml and Nginx configurations.

* Injects dynamic runtime configurations (.env) safely using GitHub Secrets.

* Deploys the stack using docker compose up -d --build for container hot-swapping.

Optimized from initial failed deployments

## 🔧 Infrastructure as Code (Terraform)

Instance is completedly deployed and managed using IaC (Terraform), first i choose Vultr but then i has some issues with my account so at the end I must switch to Digital Ocean. This 🫴🫴**[IaC Vultr](https://github.com/aleixnguyen-vn/iac-vultr)** is the configuration i used, you can check it out.

## 📊 Verification & Proof of Work

> Note: The droplet is transient and heavily rotated (Destroyed after completed my project).

### Report from VPS Terminal

The screenshot below shows the active process inside the Droplet. The multi-container stack maps internal container boundaries securely with isolated networking bridges.

![Docker PS output from DO Droplet](/images/docker-ps-from-do-droplet.png "Docker PS Command")

### Redis Cache & Site Health Normal

To ensure the 1GB RAM limitation can survive heavy concurrent loads, the application completely offloads relational queries into memory via Redis Object Cache with a custom config. Here is the confirmation of successful caching handshakes and the 100% clean WordPress Site Health status.

![Redis Cache](/images/redis-work.png)
![Site Health Good](/images/site-health-normal.png)

### Application Homepage

The functional WordPress homepage application, just some basic content

![Homepage](/images/home.png)

## 🔑 Security & Hardening

* **Zero Credential Leaking**: Plaintext credentials, DB root tokens, and system accessibility variables are strictly banned from code tracking via an aggressive .gitignore.

* **GitHub Repository Secrets**: Production access strings (DO_DROPLET_IP, DO_DROPLET_PASS) are injected directly into the execution container runtime memory and completely masked (***) within pipeline terminal logs.

> That's all for today xd