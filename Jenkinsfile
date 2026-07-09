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
                   # 1. Prepare environment and force a blank line to prevent collisions
                    touch .env
                    cp .env.example .env || true
                    echo "" >> .env
    
                    # 2. Clear old keys to prevent script panic
                    rm -rf keys/ || true
    
                    # 3. Generate keys
                    npm run setup-keys
    
                    # 4. Extract the raw text from the files
                    PRIV=$(cat keys/private_env.txt)
                    PUB=$(cat keys/public_env.txt)
    
                    # 5. Inject correctly formatted keys with variable names and quotes
                    echo "JWT_PRIVATE_KEY=\\"$PRIV\\"" >> .env
                    echo "JWT_PUBLIC_KEY=\\"$PUB\\"" >> .env
                    echo "PORT=5000" >> .env
                    echo "DATABASE_URL=\\"$DATABASE_URL\\"" >> .env
    
                    # 6. Restart server with the clean environment
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
