pipeline {
    agent any
    
    environment {
        // Define production and backup directories
        PROD_DIR = "/var/www/app2-mern"
        BACKUP_DIR = "/var/www/app2-mern-backup"
    }

    stages {
        stage('Pull & Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Prisma Migration Safety') {
            steps {
                script {
                    sh '''
                        if git diff HEAD~1 --name-only | grep -q "prisma/schema.prisma"; then
                            echo "Schema drift detected. Running migrations..."
                            npx prisma migrate deploy || true
                        else
                            echo "No schema drift detected. Skipping migrations."
                        fi
                    '''
                }
            }
        }

        stage('Frontend Build / Prisma Gen') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Build Secure Environment') {
            steps {
                sh '''
                    # 1. Prepare environment and prevent collisions
                    touch .env
                    cp .env.example .env || true
                    echo "" >> .env
                    
                    # 2. Clear old keys and generate fresh ones
                    rm -rf keys/ || true
                    npm run setup-keys
                    
                    # 3. Extract and inject keys with correct syntax
                    PRIV=$(cat keys/private_env.txt)
                    PUB=$(cat keys/public_env.txt)
                    
                    echo "JWT_PRIVATE_KEY=\\"$PRIV\\"" >> .env
                    echo "JWT_PUBLIC_KEY=\\"$PUB\\"" >> .env
                    echo "PORT=5000" >> .env
                    echo "DATABASE_URL=\\"$DATABASE_URL\\"" >> .env
                '''
            }
        }

        stage('Deploy to Production') {
            steps {
                sh '''
                    # 1. Ensure directories exist with Jenkins permissions
                    mkdir -p /var/www/app2-mern /var/www/app2-mern-backup
                    
                    # 2. Backup the current live version for safe rollbacks
                    if [ "$(ls -A ${PROD_DIR})" ]; then
                        rsync -a --delete ${PROD_DIR}/ ${BACKUP_DIR}/
                    fi
                    
                    # 3. Sync the new, compiled code to the live directory
                    rsync -a --delete --exclude='.git' ./ ${PROD_DIR}/
                    
                    # 4. Boot the server from the live production folder
                    cd ${PROD_DIR}
                    pm2 delete mern-backend || true
                    pm2 start server.js --name "mern-backend" --update-env
                '''
            }
        }

        stage('Post-Deploy Health Check') {
            steps {
                echo "Warming up application engine..."
                sleep 10
                sh '''
                    # 3 precise attempts to verify application routing
                    for i in 1 2 3; do
                        HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:5000/api/health || true)
                        if [ "$HTTP_CODE" = "200" ]; then
                            echo "Health probe successful! Application is live."
                            exit 0
                        fi
                        echo "Health probe attempt $i failed with code $HTTP_CODE."
                        sleep 5
                    done
                    
                    echo "CRITICAL: Application failed health verification."
                    exit 1 # This explicitly triggers the failure rollback block
                '''
            }
        }
    }

    post {
        failure {
            echo "INITIATING ROLLBACK: Restoring from last known-good backup directory."
            sh '''
                if [ "$(ls -A ${BACKUP_DIR})" ]; then
                    rsync -a --delete ${BACKUP_DIR}/ ${PROD_DIR}/
                    cd ${PROD_DIR}
                    pm2 delete mern-backend || true
                    pm2 start server.js --name "mern-backend" --update-env
                    echo "Rollback complete. Previous version restored."
                else
                    echo "No backup available. Manual intervention required."
                fi
            '''
        }
        always {
            echo "Wiping Jenkins workspace to prevent disk space exhaustion."
            cleanWs()
        }
    }
}
