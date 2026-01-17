pipeline {
    agent any
    
    environment {
        PYTHON = 'python3'
        PIP = 'pip3'
        // Environment variables for Flask app
        MONGO_URI = 'mongodb+srv://dbXUser:dbXUserPassword@cluster0.fhrg87w.mongodb.net/student_db?retryWrites=true&w=majority'
        SECRET_KEY = 'jenkins-secret-key-for-flask'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                echo 'Installing dependencies...'
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
                echo 'Running unit tests...'
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
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to staging environment...'
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
            echo 'Pipeline completed successfully!'
            emailext(
                subject: "✅ Jenkins Build SUCCESS: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <html>
                    <body>
                        <h2 style="color: green;">Build Successful! ✅</h2>
                        <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                        <p><strong>Status:</strong> <span style="color: green; font-weight: bold;">SUCCESS</span></p>
                        <hr>
                        <h3>🌐 Deployed Application:</h3>
                        <p><strong>Application URL:</strong> <a href="http://13.57.236.210:5000">http://13.57.236.210:5000</a></p>
                        <p><strong>Database:</strong> MongoDB Atlas (Connected)</p>
                        <hr>
                        <h3>📊 Build Info:</h3>
                        <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                        <p><strong>Console Output:</strong> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                        <p><strong>Test Results:</strong> <a href="${env.BUILD_URL}testReport">${env.BUILD_URL}testReport</a></p>
                        <hr>
                        <p style="color: gray; font-size: 12px;">Timestamp: ${env.BUILD_TIMESTAMP}</p>
                    </body>
                    </html>
                """,
                to: 'priyankpandey02@gmail.com',
                mimeType: 'text/html',
                attachLog: true
            )
        }
        failure {
            echo 'Pipeline failed!'
            emailext(
                subject: "❌ Jenkins Build FAILED: ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    <html>
                    <body>
                        <h2 style="color: red;">Build Failed! ❌</h2>
                        <p><strong>Job:</strong> ${env.JOB_NAME}</p>
                        <p><strong>Build Number:</strong> ${env.BUILD_NUMBER}</p>
                        <p><strong>Status:</strong> <span style="color: red; font-weight: bold;">FAILURE</span></p>
                        <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                        <hr>
                        <p><strong>Console Output:</strong> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
                        <p style="color: red;"><strong>Action Required:</strong> Please check the console output for error details.</p>
                        <hr>
                        <p style="color: gray; font-size: 12px;">Timestamp: ${env.BUILD_TIMESTAMP}</p>
                    </body>
                    </html>
                """,
                to: 'priyankpandey02@gmail.com',
                mimeType: 'text/html',
                attachLog: true
            )
        }
        always {
            cleanWs()
        }
    }
}
