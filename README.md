# ERPNext CI/CD Deployment Pipeline (Jenkins + Docker)

This repository contains a Jenkins Pipeline for **zero data loss** ERPNext deployments using Docker.

## 📋 Overview

The pipeline automates:

1. **Code Checkout** – Pulls the latest ERPNext/Frappe Docker setup.
2. **Pre-Deployment Checks** – Verifies disk space, backup directory, and DB container status.
3. **Database Backup** – Takes a full MySQL dump before deploying (ensures rollback capability).
4. **Deployment** – Rebuilds and starts ERPNext containers.
5. **Post-Deployment Verification** – Checks service health and DB connectivity.
6. **Rollback on Failure** – Restores DB from backup if deployment fails.
7. **Backup Cleanup** – Keeps the last 7 backups.

---

## ⚙️ Prerequisites

- **Jenkins** (Pipeline plugin installed)
- **Docker & Docker Compose**
- ERPNext Docker repository (e.g., [`frappe_docker`](https://github.com/frappe/frappe_docker))
- MySQL client tools available in Jenkins environment
- Appropriate permissions for Jenkins to run Docker commands

---

## 📁 Project Structure







---

## 🔧 Configuration

Update the **environment** section of the Jenkinsfile:

| Variable       | Description |
|----------------|-------------|
| `APP_NAME`     | Application name (default: `ERPNext`) |
| `BACKUP_DIR`   | Local path for DB backups |
| `DEPLOY_ENV`   | Environment name (`staging` or `production`) |
| `GIT_REPO`     | Git repository URL for ERPNext Docker setup |
| `DB_CONTAINER` | Name of the running MySQL container |
| `DB_NAME`      | Database name used by ERPNext |
| `DB_USER`      | MySQL username |
| `DB_PASS`      | MySQL password |
| `COMPOSE_FILE` | Docker Compose file name |

💡 **Tip:** For security, use Jenkins Credentials instead of hardcoding DB user/password.

---

## 🚀 Setup Instructions

1. **Clone Repository**
   ```bash
   git clone https://github.com/YOUR-USERNAME/erpnext-pipeline.git
   cd erpnext-pipeline
