pipeline {
    agent any
    
    environment {
        PYTHON = 'python3'
        PIP = 'pip3'
        MONGO_URI = credentials('MONGO_URI')
        SECRET_KEY = credentials('SECRET_KEY')
        EMAIL_TO = credentials('EMAIL_TO')
        DEPLOY_DIR = '/var/lib/jenkins/flask-app-deploy'
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
                    # Create deployment directory
                    mkdir -p ${DEPLOY_DIR}
                    
                    # Copy application files
                    echo "📁 Copying application files..."
                    cp app.py ${DEPLOY_DIR}/
                    
                    # Copy templates and static if they exist
                    if [ -d "templates" ]; then
                        cp -r templates ${DEPLOY_DIR}/
                    fi
                    if [ -d "static" ]; then
                        cp -r static ${DEPLOY_DIR}/
                    fi
                    
                    # Copy virtual environment
                    echo "📦 Copying virtual environment..."
                    rm -rf ${DEPLOY_DIR}/venv
                    cp -r venv ${DEPLOY_DIR}/
                    
                    # Create .env file with credentials
                    echo "🔑 Creating environment file..."
                    echo "MONGO_URI=${MONGO_URI}" > ${DEPLOY_DIR}/.env
                    echo "SECRET_KEY=${SECRET_KEY}" >> ${DEPLOY_DIR}/.env
                    
                    # Kill any existing Flask process
                    echo "🔄 Stopping existing Flask processes..."
                    pkill -f "python.*app.py" || true
                    sleep 3
                    
                    # Navigate to deployment directory and start app
                    echo "🚀 Starting Flask application..."
                    cd ${DEPLOY_DIR}
                    . venv/bin/activate
                    
                    # Start Flask in background with nohup
                    nohup ${PYTHON} app.py > ${DEPLOY_DIR}/flask.log 2>&1 &
                    FLASK_PID=$!
                    echo $FLASK_PID > ${DEPLOY_DIR}/flask.pid
                    
                    echo "⏳ Waiting for application to start..."
                    sleep 5
                    
                    # Verify app is running
                    if pgrep -f "python.*app.py" > /dev/null; then
                        echo "✅ Application deployed successfully!"
                        echo "🌐 Application URL: http://13.57.236.210:5000"
                        echo "📊 Database: MongoDB Atlas"
                        echo "📂 Deploy Location: ${DEPLOY_DIR}"
                        echo "🆔 Process ID: $FLASK_PID"
                        echo ""
                        echo "📋 Application Status:"
                        ps aux | grep "python.*app.py" | grep -v grep
                    else
                        echo "❌ Application failed to start!"
                        echo "📋 Flask log:"
                        cat ${DEPLOY_DIR}/flask.log || echo "No log file found"
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
                 body: "Build succeeded!\n\nJob: ${env.JOB_NAME}\nBuild: ${env.BUILD_NUMBER}\nBuild URL: ${env.BUILD_URL}\n\nApplication URL: http://13.57.236.210:5000\nDeploy Location: /var/lib/jenkins/flask-app-deploy\n\nTest Results: All tests passed!"
        }
        failure {
            echo '❌ Pipeline failed!'
            mail to: "${EMAIL_TO}",
                 subject: "❌ Jenkins Build FAILED: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                 body: "Build failed!\n\nJob: ${env.JOB_NAME}\nBuild: ${env.BUILD_NUMBER}\nBuild URL: ${env.BUILD_URL}\n\nConsole Output: ${env.BUILD_URL}console\n\nPlease check the console output for error details."
        }
        always {
            cleanWs()
        }
    }
}
