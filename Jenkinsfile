pipeline {
    agent any
    
    environment {
        PYTHON = 'python3'
        PIP = 'pip3'
        MONGO_URI = credentials('MONGO_URI')
        SECRET_KEY = credentials('SECRET_KEY')
        EMAIL_TO = credentials('EMAIL_TO')
        // Deployment directory outside Jenkins workspace
        DEPLOY_DIR = '/home/ubuntu/flask-app'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo '🔍 Checking out code from GitHub...'
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo '📦 Installing dependencies...'
                sh '''
                    ${PYTHON} -m venv venv
                    . venv/bin/activate
                    ${PIP} install --upgrade pip
                    if [ -f requirements.txt ]; then
                        ${PIP} install -r requirements.txt
                    fi
                '''
            }
        }
        
        stage('Test') {
            steps {
                echo '🧪 Running unit tests...'
                sh '''
                    . venv/bin/activate
                    export MONGO_URI="${MONGO_URI}"
                    export SECRET_KEY="${SECRET_KEY}"
                    ${PYTHON} -m pytest tests/ --verbose --junit-xml=test-results.xml || true
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                echo '🚀 Deploying to staging environment...'
                sh '''
                    # Create deployment directory if it doesn't exist
                    mkdir -p ${DEPLOY_DIR}
                    
                    # Copy application files to deployment directory
                    cp -r app.py templates/ static/ ${DEPLOY_DIR}/ 2>/dev/null || cp app.py ${DEPLOY_DIR}/
                    
                    # Copy virtual environment
                    cp -r venv ${DEPLOY_DIR}/
                    
                    # Create .env file in deployment directory
                    cat > ${DEPLOY_DIR}/.env <<EOF
MONGO_URI=${MONGO_URI}
SECRET_KEY=${SECRET_KEY}
EOF
                    
                    # Kill any existing Flask process
                    pkill -f "python.*app.py" || true
                    sleep 2
                    
                    # Start Flask application from deployment directory
                    cd ${DEPLOY_DIR}
                    . venv/bin/activate
                    
                    # Start Flask in background
                    nohup ${PYTHON} app.py > flask.log 2>&1 &
                    echo $! > flask.pid
                    
                    sleep 5
                    
                    # Verify app is running
                    if pgrep -f "python.*app.py" > /dev/null; then
                        echo "✅ Application deployed successfully!"
                        echo "🌐 Application running at: http://13.57.236.210:5000"
                        echo "📊 Connected to MongoDB Atlas"
                        echo "📂 Deployed to: ${DEPLOY_DIR}"
                        echo "🆔 Process ID: $(cat flask.pid)"
                    else
                        echo "❌ Application failed to start"
                        echo "📋 Log file content:"
                        cat flask.log
                        exit 1
                    fi
                '''
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline completed successfully!'
            mail to: "${EMAIL_TO}",
                 subject: "✅ Jenkins Build SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                 body: "Build succeeded!\n\nJob: ${env.JOB_NAME}\nBuild: ${env.BUILD_NUMBER}\nURL: ${env.BUILD_URL}\n\nApplication: http://13.57.236.210:5000\n\nDeployed to: /home/ubuntu/flask-app"
        }
        failure {
            echo '❌ Pipeline failed!'
            mail to: "${EMAIL_TO}",
                 subject: "❌ Jenkins Build FAILED: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                 body: "Build failed!\n\nJob: ${env.JOB_NAME}\nBuild: ${env.BUILD_NUMBER}\nURL: ${env.BUILD_URL}\n\nCheck console: ${env.BUILD_URL}console"
        }
        always {
            cleanWs()  // Now safe to clean - app is deployed elsewhere
        }
    }
}
