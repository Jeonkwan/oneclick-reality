# AWS Lightsail Linux Instance Plan

## Overview
- **Objective:** Reuse the existing Docker Compose bundle from `less-vision-reality` on a lightweight Lightsail VM with aggressive start/stop automation to minimize hourly charges.
- **Minimal runtime needs:** Provision a base image with Docker Engine (no Compose plugin necessary if `docker compose` binary is baked into AMI snapshot). The proxy only needs the rendered config directory, persistent secret store, and the `ghcr.io/xtls/xray-core` container runtime.
- **Automation entry point:** Terraform or AWS CDK instantiates the VM with a prebuilt snapshot where Docker and Compose are already installed, enabling sub-2-minute boot-to-ready by skipping package installations.

## Reference Architecture
```mermaid
graph TD
    gha[GitHub Actions Runner] -->|Render config + push| s3cfg[(Config bundle S3/Artifact)]
    gha -->|Invoke| tf[Terralform/CLI]
    tf --> lsVM[Lightsail Linux Instance ($5 tier)]
    s3cfg -->|Fetch on boot| lsVM
    lsVM -->|Docker Compose up| xray[Xray Vision/Reality Container]
    scheduler[EventBridge Scheduler] -->|Stop/start| lsVM
    xray --> users[Vision/Reality Clients]
```

## Components & Responsibilities
- **Prebaked snapshot** includes Docker Engine, Compose plugin, and `less-vision-reality` playbooks cached locally to avoid runtime installation.
- **Bootstrap script** downloads the latest config bundle and credentials from S3 or Artifact storage, mounts them under `/var/lib/xray`, and runs `docker compose up -d`.
- **EventBridge Scheduler** (or GitHub Actions cron) stops the instance nightly and restarts it before expected usage windows.
- **Terraform state management** prevents accidental duplication; workspaces create intentional additional environments when needed.

## Deployment Workflow
1. **Image build:** Use Packer or EC2 Image Builder to create a custom Lightsail blueprint with Docker pre-installed.
2. **Config packaging:** Render configuration artifacts in GitHub Actions, upload them to S3, and record metadata (version, checksum) in the repo.
3. **Provision:** Terraform applies to launch the instance using the custom blueprint and attaches a startup script that fetches the latest config bundle.
4. **Operate:** Use simple `aws lightsail stop-instance` / `start-instance` commands (or scheduled automation) for nightly shutdowns to save cost.
5. **Update:** Redeploy by uploading a new config bundle and triggering a lightweight systemd service to `docker compose pull && docker compose up -d`.

## Cost Considerations
- The smallest Linux/Unix bundle (0.5 GB RAM, 1 vCPU, 20 GB SSD) is **$5 USD/month** with 1 TB included transfer.【f75632†L16-L24】
- Stopping a Lightsail instance stops compute billing immediately; storage and static IPs continue to incur minimal charges.
- Additional data transfer beyond the bundled 1 TB falls back to standard AWS data transfer rates, so monitor heavy streaming usage.

## Pros
- Fully compatible with existing Docker Compose workflow—no refactoring required.
- Predictable pricing with generous 1 TB bandwidth included.【f75632†L16-L24】
- Easy to snapshot/clone for staging environments without re-running long bootstrap sequences.

## Cons
- Even when stopped, block storage and snapshots still incur cost; full teardown must remove them to reach $0 spend.
- Manual vigilance required to avoid orphaned instances when running multiple environments.
- Startup relies on Lightsail boot scripts; debugging failure requires instance console access.
