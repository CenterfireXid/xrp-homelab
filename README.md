# XRP Homelab

>**Engineering documentation and Infrastructure-as-Code for the XRP Homelab.**



## Overview

XRP Homelab is my personal self-hosted infrastructure project built around a Raspberry Pi 5 (16 GB). This repository serves as the single source of truth for every service, configuration, and architectural decision within the homelab.

Unlike many homelab repositories that only contain Docker Compose files, this repository is intended to function as complete engineering documentation.

Every application includes:

- Docker Compose configuration
- Installation documentation
- Configuration notes
- Backup procedures
- Update procedures
- Recovery procedures
- Changelog
- Design decisions

The goal is simple:

> **If the Raspberry Pi failed today, I should be able to rebuild the entire homelab using only this repository.**

Ironically, the Raspberry Pi failed during the early stages of this project, forcing a complete rebuild from scratch.

What started as a design goal quickly became a real-world disaster recovery exercise. This repository is the direct result of that experience.

Apparently, Murphy's Law reads Markdown too.

# Design Philosophy

The homelab follows several core principles.

## Infrastructure as Code

All permanent services are deployed using Docker Compose.

Container creation through the Portainer GUI is intentionally avoided to ensure every deployment is reproducible and version controlled.



## Documentation First

Every service is documented before it is considered complete.

Documentation is treated as part of the deployment—not an afterthought.



## Disaster Recovery

A rebuild should never rely on memory.

Every service should include enough documentation that a complete rebuild can be performed from scratch.



## Simplicity

Whenever possible:

- Docker Compose over docker run
- Bind mounts when appropriate
- Standardized directory structure
- Consistent naming conventions



## Observability

Portainer is used as the operational dashboard for:

- Container health
- Logs
- Resource usage
- Networks
- Volumes

Docker Compose remains the source of truth.



# Repository Structure

```
docs/
```

General documentation and design decisions.

```
stacks/
```

Docker Compose projects organized by category.

```
scripts/
```

Automation scripts.

```
assets/
```

Images, diagrams, screenshots, and architecture drawings.



# Installation Workflow

Every service follows the same workflow:

1. Research
2. Directory creation
3. Persistent storage planning
4. Compose creation
5. Validation
6. Deployment
7. Verification
8. Documentation
9. Git commit

This keeps every deployment consistent and repeatable.



# Current Infrastructure

| Component | Status |
|------------|--------|
| Raspberry Pi 5 (16GB) | ✅ |
| Raspberry Pi OS Lite | ✅ |
| Docker Engine | ✅ |
| Docker Compose | ✅ |
| Tailscale | ✅ |
| Portainer | ✅ |
| Tracktor | ✅ |



# Future Services

- Uptime Kuma
- Tracktor
- Home Assistant
- Immich
- Paperless-ngx
- Nginx Proxy Manager
- Fail2Ban
- Karaoke Eternal
- Watchtower (notifications only)
- Dozzle
- Palmr




# Goals

- Learn modern infrastructure management
- Improve Linux administration skills
- Develop DevOps best practices
- Maintain production-quality documentation
- Build a reproducible homelab



## License

This repository is maintained for educational purposes and personal documentation.
