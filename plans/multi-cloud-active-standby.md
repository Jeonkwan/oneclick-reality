# Multi-Cloud Active/Standby Plan (Cloud Run + Lightsail)

## Overview
- **Objective:** Combine Google Cloud Run (active) with AWS Lightsail Container Service (standby) to hedge against regional outages and experiment with traffic steering while keeping deployment automation unified.
- **Minimal runtime needs:** Both targets consume the same pre-rendered Xray configuration and container image. Secrets remain provider-specific but stem from a single secure source (GitHub Actions OIDC + secret store).
- **Automation entry point:** GitHub Actions multi-job workflow builds once, pushes to Artifact Registry and Lightsail, and updates each platform in sequence with environment-specific flags.

## Reference Architecture
```mermaid
graph TD
    gha[GitHub Actions Pipeline]
    gha -->|Build once| img[(Container Image + Config bundle)]
    img -->|Push| gar[Artifact Registry]
    img -->|Push| lsRegistry[Lightsail Container Registry]
    gar -->|Deploy| cloudrun[Cloud Run (Active)]
    lsRegistry -->|Deploy| lightsail[Lightsail Container (Standby)]
    secrets[Central Secrets (GitHub/OIDC -> Secret Manager & Lightsail Secrets)] --> cloudrun
    secrets --> lightsail
    cloudrun --> dns[Geo DNS / Cloudflare]
    lightsail --> dns
    scheduler[Scheduler Policies] -->|Scale-to-zero nightly| cloudrun
    scheduler -->|Stop service nights| lightsail
```

## Components & Responsibilities
- **GitHub Actions matrix workflow** produces identical images and triggers deployments to both providers using dedicated jobs with minimal inline scripting.
- **Cloud Run** serves primary traffic with scale-to-zero; Cloud Scheduler enforces overnight pauses to limit charges.【23a9c8†L1-L9】
- **Lightsail Container Service** idles at zero nodes until failover testing; manual or scripted scaling restores service within ~2 minutes.【f75632†L6-L24】
- **DNS steering (Cloudflare Load Balancer or Route 53 + Cloud DNS)** toggles traffic between providers or splits per geography.

## Deployment Workflow
1. **Render & build** artifacts once per commit; store config checksum as image label.
2. **Parallel deploy:** Use GitHub Actions matrix (providers: gcp, aws) calling `gcloud run deploy` and `aws lightsail create-container-service-deployment` respectively.
3. **Secret sync:** Leverage GitHub OIDC to read from a central secret store (1Password, HashiCorp Vault) and push provider-specific secrets via CLI APIs.
4. **Health checks:** Cloud Run and Lightsail both expose HTTP health probes; integrate with Cloudflare Load Balancer origin health to automate failover.
5. **Scheduled dormancy:** Configure Cloud Scheduler to set Cloud Run min instances to 0 overnight and AWS CLI cron to stop Lightsail nodes for zero cost when unused.

## Cost Considerations
- Cloud Run active usage accrues **$0.000024 per vCPU-second** and **$0.0000025 per GiB-second**, plus **$0.40 per million requests** in us-central1.【23a9c8†L1-L9】
- Lightsail standby incurs **$10/month** only when nodes are running; scaling to zero removes compute charges, leaving registry storage negligible.【f75632†L6-L24】
- Dual-provider egress may double bandwidth spend; consider routing standby traffic only during outages to avoid duplicate transfer charges.

## Pros
- Single pipeline builds once and feeds multiple targets, supporting blue/green or failover rehearsals.
- Ability to fully tear down standby compute nightly without losing configuration artifacts.
- Cloudflare/Route 53 health checks provide automated failover with minimal manual intervention.

## Cons
- Requires disciplined secret propagation; mismatched Reality keys between providers will break clients.
- Separate observability stacks (Cloud Logging vs. CloudWatch) increase operational overhead.
- Duplicate infrastructure may still incur minimum charges (e.g., Cloud Run revisions storage, Lightsail image storage) even when scaled to zero.
