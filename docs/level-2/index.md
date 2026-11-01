# Level 2 · Intermediate <span class="level-badge">Intermediate</span>

Goal: move from "a server that runs one app" to "a server (or fleet) that
serves production traffic safely" — reverse proxies, TLS, load balancing,
staged environments, zero-downtime deploys, and basic monitoring.

## Modules

1. [Reverse Proxies (nginx basics)](01-reverse-proxies-nginx.md)
2. [TLS/SSL Certificates (Let's Encrypt / certbot)](02-tls-ssl-certs.md)
3. [Load Balancing Basics](03-load-balancing-basics.md)
4. [Environment-Based Deployment (staging vs. prod)](04-environment-based-deployment.md)
5. [Zero-Downtime Deploy Patterns](05-zero-downtime-deploy.md)
6. [Secrets & Configuration Management](06-secrets-config-management.md)
7. [CI/CD Pipeline Basics](07-cicd-pipeline-basics.md)
8. [Basic Monitoring (uptime checks, resource metrics)](08-basic-monitoring.md)
9. [Centralized Logging Basics](09-centralized-logging-basics.md)
10. [Capstone — Blue-Green Deploy Pipeline](10-capstone-blue-green-deploy.md)

Work through the modules in order — each one builds on the primitives
(nginx, systemd, health checks) introduced in the ones before it, and the
capstone ties all ten together into one working blue-green pipeline.
