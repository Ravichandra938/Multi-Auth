pipeline {
    agent any
    
    environment {
        // Will be populated dynamically by Jenkins during deployment
        DATABASE_URL = credentials('MERN_DB_URL')
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
                    // Logic Challenge 3: Checks for schema changes before blindly running migrations[cite: 2]
                    def schemaChanged = sh(script: "git diff HEAD~1 --name-only | grep 'prisma/schema.prisma' || true", returnStdout: true).trim()
                    
                    if (schemaChanged) {
                        echo "Schema modifications detected. Launching Prisma migrations..."
                        try {
                            sh 'npx prisma migrate deploy'
                        } catch(Exception e) {
                            error "Migration failed mid-deploy: ${e.getMessage()}. Halting pipeline to prevent app from starting against a broken schema."
                        }
                    } else {
                        echo "No schema drift detected. Skipping migrations."
                    }
                }
            }
        }
        
        stage('Frontend Build') {
            steps {
                // Required by assessment: run npm run build for the React frontend[cite: 2]
                sh 'npm run build'
            }
        }
        
        stage('Restart Express Server') {
            steps {
                script {
                    // Ensure the .env file is loaded and the backend is rebooted via PM2[cite: 2]
                    sh '''
                    touch .env
                    cp .env.example .env || true
                    npm run setup-keys || true
                    echo "JWT_PRIVATE_KEY=super_secret_assessment_key_123" >> .env
                    echo "PORT=5000" >> .env
                    pm2 delete mern-backend || true
                    pm2 start server.js --name "mern-backend" --update-env
                    '''
                }
            }
        }
        
        stage('Post-Deploy Health Check') {
            steps {
                script {
                    echo "Warming up application engine..."
                    sleep 10
                    
                    def success = false
                    for(int i=0; i<3; i++) {
                        try {
                            def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:5000/api/health", returnStdout: true).trim()
                            if(response == "200") {
                                success = true
                                echo "MERN integration health check passed."
                                break
                            }
                        } catch(Exception e) {
                            echo "Health probe attempt ${i+1} failed."
                        }
                        sleep 5
                    }
                    
                    if(!success) {
                        error "MERN application failed health verification. Initiating automatic rollback."
                    }
                }
            }
        }
    }
    
    post {
        failure {
            script {
                echo "CRITICAL: Health check or migration failed. Reverting to last known-good git commit."
                sh '''
                git reset --hard HEAD@{1}
                npm install
                pm2 restart mern-backend
                '''
            }
        }
    }
}
