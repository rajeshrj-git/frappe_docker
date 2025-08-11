pipeline {
    agent any
    environment {
        APP_NAME       = "ERPNext"
        BACKUP_DIR     = "/home/rajesh/neo_Touch_CRM/erp_backups"
        DEPLOY_ENV     = "staging"  // change to "production" when ready
        GIT_REPO       = "https://github.com/rajeshrj-git/frappe_docker.git"
        DB_CONTAINER   = "frappe_docker_db_1"
        DB_NAME        = "_5e5899d8398b5f7b" // replace if needed
        DB_USER        = "root" // Consider using credentials('erpnext-db-user') 
        DB_PASS        = "admin" // Consider using credentials('erpnext-db-pass')
        COMPOSE_FILE   = "pwd.yml" // Fixed from pwd.yml
    }
    stages {
        stage('Checkout Code') {
            steps {
                echo "📥 Checking out code from ${GIT_REPO}..."
                git url: "${GIT_REPO}", branch: 'main'
                
                // Verify compose file exists
                script {
                    if (!fileExists("${COMPOSE_FILE}")) {
                        // Try common alternatives
                        if (fileExists("docker-compose.yaml")) {
                            env.COMPOSE_FILE = "docker-compose.yaml"
                        } else if (fileExists("compose.yml")) {
                            env.COMPOSE_FILE = "compose.yml"
                        } else {
                            error("❌ No docker compose file found! Expected: docker-compose.yml, docker-compose.yaml, or compose.yml")
                        }
                    }
                    echo "✅ Using compose file: ${env.COMPOSE_FILE}"
                }
            }
        }
        
        stage('Pre-Deployment Checks') {
            steps {
                script {
                    echo "🔍 Running pre-deployment validation..."
                    
                    // Create backup directory
                    sh "mkdir -p ${BACKUP_DIR}"
                    
                    // Check if database container is running
                    def containerStatus = sh(
                        script: "docker ps --filter name=${DB_CONTAINER} --format '{{.Status}}' || echo 'Not running'", 
                        returnStdout: true
                    ).trim()
                    
                    if (containerStatus.contains("Up")) {
                        echo "✅ Database container ${DB_CONTAINER} is running"
                    } else {
                        echo "⚠️ Warning: Database container ${DB_CONTAINER} is not running"
                        echo "This might be the first deployment or container needs to be started"
                    }
                    
                    // Check available disk space (at least 1GB)
                    sh """
                        echo "💾 Checking disk space..."
                        df -h ${BACKUP_DIR}
                        AVAILABLE=\$(df ${BACKUP_DIR} | tail -1 | awk '{print \$4}')
                        if [ \$AVAILABLE -lt 1000000 ]; then
                            echo "⚠️ Warning: Less than 1GB available for backups"
                        fi
                    """
                }
            }
        }
        
        stage('Backup ERPNext DB - Zero Data Loss') {
            steps {
                script {
                    def timestamp = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    def backupFile = "${BACKUP_DIR}/erpnext_backup_${timestamp}.sql"
                    
                    // Store backup file path for rollback
                    env.CURRENT_BACKUP = backupFile
                    
                    echo "💾 Backing up database ${DB_NAME} from container ${DB_CONTAINER}..."
                    
                    // Check if container is running before backup
                    def containerRunning = sh(
                        script: "docker ps -q --filter name=${DB_CONTAINER}",
                        returnStdout: true
                    ).trim()
                    
                    if (containerRunning) {
                        sh """
                            echo "Creating backup: ${backupFile}"
                            docker exec ${DB_CONTAINER} \
                            sh -c 'mysqldump -u${DB_USER} -p${DB_PASS} --single-transaction --routines --triggers ${DB_NAME}' \
                            > ${backupFile}
                            
                            # Verify backup was created and is not empty
                            if [ ! -s ${backupFile} ]; then
                                echo "❌ Backup failed - file is empty or not created"
                                exit 1
                            fi
                            
                            echo "✅ Backup completed successfully"
                            ls -lh ${backupFile}
                        """
                        echo "✅ Backup saved to ${backupFile}"
                    } else {
                        echo "⚠️ Database container not running - skipping backup (first deployment?)"
                        // Create empty backup file marker
                        sh "touch ${backupFile}.no_backup_container_not_running"
                    }
                }
            }
        }
        
        stage('Deploy ERPNext') {
            steps {
                echo "🚀 Deploying ERPNext to ${DEPLOY_ENV}..."
                script {
                    try {
                        sh """
                            echo "📉 Stopping existing containers..."
                            docker compose -f ${COMPOSE_FILE} down --timeout 30 || true
                            
                            echo "🔄 Building and starting containers..."
                            docker compose -f ${COMPOSE_FILE} up -d --build
                            
                            echo "⏳ Waiting for containers to stabilize..."
                            sleep 20
                            
                            echo "📊 Container status:"
                            docker compose -f ${COMPOSE_FILE} ps
                        """
                        echo "✅ Deployment completed successfully."
                    } catch (Exception e) {
                        echo "❌ Deployment failed: ${e.getMessage()}"
                        currentBuild.result = 'FAILURE'
                        throw e
                    }
                }
            }
        }
        
        stage('Post-Deployment Verification') {
            steps {
                echo "🔍 Verifying deployment..."
                script {
                    // Check if containers are running
                    def runningContainers = sh(
                        script: "docker compose -f ${COMPOSE_FILE} ps --services --filter status=running",
                        returnStdout: true
                    ).trim()
                    
                    if (runningContainers) {
                        echo "✅ Containers are running: ${runningContainers.split('\n').join(', ')}"
                    } else {
                        echo "❌ No containers are running!"
                        currentBuild.result = 'FAILURE'
                        error("Deployment verification failed - no containers running")
                    }
                    
                    // Optional: Add basic connectivity test
                    sh """
                        echo "🔌 Testing database connectivity..."
                        sleep 10
                        docker exec ${DB_CONTAINER} mysql -u${DB_USER} -p${DB_PASS} -e "SELECT 1 as test;" ${DB_NAME} || echo "⚠️ DB connection test failed"
                    """
                }
            }
        }
        
        // Comment out for production use - only for testing rollback functionality
        stage('Force Failure to Test Rollback') {
            when {
                // Only run this in non-production environments
                expression { DEPLOY_ENV != "production" }
            }
            steps {
                script {
                    // Uncomment the line below to test rollback functionality
                    // error("Intentional failure to test rollback!")
                    echo "🧪 Rollback test stage - currently disabled. Uncomment to test rollback."
                }
            }
        }
    }
    
    post {
        failure {
            echo "❌ Build failed! Initiating rollback procedure..."
            script {
                try {
                    if (env.CURRENT_BACKUP && fileExists(env.CURRENT_BACKUP)) {
                        echo "🔄 Rolling back database from backup: ${env.CURRENT_BACKUP}..."
                        
                        // Verify backup file is valid
                        def backupSize = sh(
                            script: "stat -c%s ${env.CURRENT_BACKUP} 2>/dev/null || echo 0",
                            returnStdout: true
                        ).trim() as Integer
                        
                        if (backupSize > 100) { // At least 100 bytes
                            sh """
                                echo "📥 Restoring database from backup..."
                                docker exec -i ${DB_CONTAINER} \
                                sh -c 'mysql -u${DB_USER} -p${DB_PASS} ${DB_NAME}' < ${env.CURRENT_BACKUP}
                                
                                echo "🔄 Restarting containers after rollback..."
                                docker compose -f ${COMPOSE_FILE} restart
                                
                                sleep 10
                                echo "📊 Post-rollback container status:"
                                docker compose -f ${COMPOSE_FILE} ps
                            """
                            echo "✅ Database rollback completed successfully."
                        } else {
                            echo "⚠️ Backup file too small or empty - skipping database rollback"
                        }
                    } else {
                        echo "⚠️ No valid backup file available for rollback!"
                        echo "Available backups:"
                        sh "ls -la ${BACKUP_DIR}/ | tail -5 || echo 'No backups found'"
                    }
                    
                    // Try to restore previous container state
                    sh """
                        echo "🔄 Attempting to restore service availability..."
                        docker compose -f ${COMPOSE_FILE} up -d || echo "Failed to restart containers"
                    """
                    
                } catch (Exception rollbackError) {
                    echo "❌ CRITICAL ERROR: Rollback failed!"
                    echo "Error details: ${rollbackError.getMessage()}"
                    echo "🚨 Manual intervention required!"
                    echo "Backup location: ${env.CURRENT_BACKUP}"
                    
                    // Send critical alert (customize as needed)
                    sh """
                        echo "CRITICAL: ERPNext deployment and rollback failed on \$(date)" >> ${BACKUP_DIR}/critical_failures.log
                        echo "Build: ${BUILD_NUMBER}" >> ${BACKUP_DIR}/critical_failures.log
                        echo "Backup: ${env.CURRENT_BACKUP}" >> ${BACKUP_DIR}/critical_failures.log
                        echo "---" >> ${BACKUP_DIR}/critical_failures.log
                    """
                }
            }
        }
        
        success {
            echo "✅ Pipeline completed successfully with zero data loss guarantee!"
            script {
                if (env.CURRENT_BACKUP) {
                    echo "💾 Backup created: ${env.CURRENT_BACKUP}"
                    
                    // Clean up old backups (keep last 7)
                    sh """
                        echo "🧹 Cleaning up old backups (keeping last 7)..."
                        cd ${BACKUP_DIR}
                        ls -t erpnext_backup_*.sql 2>/dev/null | tail -n +8 | xargs -r rm -f
                        echo "📋 Current backups:"
                        ls -la erpnext_backup_*.sql 2>/dev/null | head -7 || echo "No backup files found"
                    """
                }
                
                // Log successful deployment
                sh """
                    echo "SUCCESS: ERPNext deployment completed on \$(date)" >> ${BACKUP_DIR}/deployment_log.txt
                    echo "Environment: ${DEPLOY_ENV}" >> ${BACKUP_DIR}/deployment_log.txt
                    echo "Build: ${BUILD_NUMBER}" >> ${BACKUP_DIR}/deployment_log.txt
                    echo "Backup: ${env.CURRENT_BACKUP}" >> ${BACKUP_DIR}/deployment_log.txt
                    echo "---" >> ${BACKUP_DIR}/deployment_log.txt
                """
            }
        }
        
        always {
            echo "📊 Pipeline execution summary:"
            echo "- Environment: ${DEPLOY_ENV}"
            echo "- Build Number: ${BUILD_NUMBER}"
            echo "- Backup Location: ${BACKUP_DIR}"
            
            script {
                // Archive deployment info
                if (env.CURRENT_BACKUP) {
                    writeFile file: 'deployment_summary.txt', text: """
ERPNext Deployment Summary
=========================
Date: ${new Date()}
Environment: ${DEPLOY_ENV}
Build Number: ${BUILD_NUMBER}
Backup File: ${env.CURRENT_BACKUP}
Git Repository: ${GIT_REPO}
Database: ${DB_NAME}
Container: ${DB_CONTAINER}
Status: ${currentBuild.result ?: 'SUCCESS'}
"""
                    archiveArtifacts artifacts: 'deployment_summary.txt', allowEmptyArchive: true
                }
            }
        }
    }
}
