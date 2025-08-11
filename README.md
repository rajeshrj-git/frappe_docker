# ERPNext Jenkins Deployment Pipeline

## 1. Overview
This Jenkins pipeline automates the backup, deployment, and verification of an ERPNext environment running on Docker.
It ensures zero data loss by taking a MySQL/MariaDB database backup before deployment and includes an automatic rollback mechanism if deployment fails.

### Key Features
- Git checkout of ERPNext Docker repository
- Automatic Docker Compose file detection
- Pre-deployment checks (disk space, container status)
- Database backup with validation
- Docker container stop/build/start
- Post-deployment verification
- Optional rollback testing (non-production only)
- Automatic rollback on failure
- Backup retention management (keep last 7 backups)
- Deployment logging for audit purposes

---

## 2. Requirements

**Server & Software:**
- Jenkins with Pipeline plugin
- Docker & Docker Compose
- Git installed on build server
- ≥1 GB free disk space

**Credentials:**
- Docker access for Jenkins user
- Read/write permissions to backup directory
- MySQL/MariaDB user with full DB access

---

## 3. Environment Variables

| Variable       | Purpose |
|----------------|---------|
| `APP_NAME`     | Application name (for logging/reference) |
| `BACKUP_DIR`   | Directory path for storing DB backups |
| `DEPLOY_ENV`   | Deployment environment (staging or production) |
| `GIT_REPO`     | ERPNext Docker Git repository URL |
| `DB_CONTAINER` | Name of the MySQL/MariaDB container |
| `DB_NAME`      | Database name |
| `DB_USER`      | Database user |
| `DB_PASS`      | Database password |
| `COMPOSE_FILE` | Docker Compose file name |

---

## 4. Setup Steps

1. **Prepare Jenkins Server:**
   - Install Docker, Docker Compose, and Git.
   - Add Jenkins user to Docker group and restart Jenkins:
     ```bash
     sudo usermod -aG docker jenkins
     sudo systemctl restart jenkins
     ```

2. **Prepare Backup Directory:**
   ```bash
   sudo mkdir -p /path/to/backups
   sudo chown -R jenkins:jenkins /path/to/backups
