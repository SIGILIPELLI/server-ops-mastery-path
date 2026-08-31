# Level 2 · Intermediate <span class="level-badge">Intermediate</span>

Goal: move from "a server that runs one app" to "a server (or fleet) that
serves production traffic safely" — reverse proxies, TLS, load balancing,
staged environments, zero-downtime deploys, and basic monitoring.

## Modules

1. Reverse Proxies (nginx basics)
2. TLS/SSL Certificates (Let's Encrypt / certbot)
3. Load Balancing Basics
4. Environment-Based Deployment (staging vs. prod)
5. Zero-Downtime Deploy Patterns
6. Secrets & Configuration Management
7. CI/CD Pipeline Basics
8. Basic Monitoring (uptime checks, resource metrics)
9. Centralized Logging Basics
10. Capstone — Blue-Green Deploy Pipeline

Work through the modules in order — each one builds on the primitives
(nginx, systemd, health checks) introduced in the ones before it, and the
capstone ties all ten together into one working blue-green pipeline.
