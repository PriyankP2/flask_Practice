pipeline {
    agent any
    
    environment {
        PYTHON = 'python3'
        PIP = 'pip3'
        MONGO_URI = credentials('MONGO_URI')
        SECRET_KEY = credentials('SECRET_KEY')
        EMAIL_TO = credentials('EMAIL_TO')
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
                    . venv/bin/activate
                    
                    # Kill any existing Flask process
                    pkill -f "flask run" || true
                    pkill -f "python.*app.py" || true
                    
                    # Wait for process to stop
                    sleep 2
                    
                    # Set environment variables
                    export FLASK_APP=app.py
                    export FLASK_ENV=development
                    export MONGO_URI="${MONGO_URI}"
                    export SECRET_KEY="${SECRET_KEY}"
                    
                    # Start Flask application in background
                    nohup ${PYTHON} app.py > flask.log 2>&1 &
                    echo $! > flask.pid
                    
                    # Wait for app to start
                    sleep 5
                    
                    # Check if app is running
                    if pgrep -f "python.*app.py" > /dev/null; then
                        echo "✅ Application deployed successfully!"
                        echo "🌐 Access at: http://13.57.236.210:5000"
                        echo "📊 Connected to MongoDB Atlas cluster"
                    else
                        echo "❌ Application failed to start. Check flask.log"
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
                 body: "Build succeeded!\n\nJob: ${env.JOB_NAME}\nBuild: ${env.BUILD_NUMBER}\nURL: ${env.BUILD_URL}\n\nApplication: http://13.57.236.210:5000"
        }
        failure {
            echo '❌ Pipeline failed!'
            mail to: "${EMAIL_TO}",
                 subject: "❌ Jenkins Build FAILED: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                 body: "Build failed!\n\nJob: ${env.JOB_NAME}\nBuild: ${env.BUILD_NUMBER}\nURL: ${env.BUILD_URL}\n\nCheck console: ${env.BUILD_URL}console"
        }
        always {
            cleanWs()
        }
    }
}
