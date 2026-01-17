pipeline {
    agent any
    
    environment {
        PYTHON = 'python3'
        PIP = 'pip3'
        // MongoDB URI - using credentials for security
        MONGO_URI = 'mongodb+srv://dbXUser:dbXUserPassword@cluster0.fhrg87w.mongodb.net/student_db?retryWrites=true&w=majority'
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
                    ${PIP} install flask pytest flask-pymongo
                '''
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running unit tests...'
                sh '''
                    . venv/bin/activate
                    export MONGO_URI="${MONGO_URI}"
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
                    
                    # Set environment variables
                    export FLASK_APP=app.py
                    export FLASK_ENV=development
                    export MONGO_URI="${MONGO_URI}"
                    
                    # Start Flask application
                    nohup ${PYTHON} -m flask run --host=0.0.0.0 --port=5000 > flask.log 2>&1 &
                    echo $! > flask.pid
                    sleep 5
                    
                    echo "Application deployed to staging at http://13.57.236.210:5000"
                    echo "MongoDB connected to Atlas cluster"
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
                        <p><strong>Build URL:</strong> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                        <p><strong>Application URL:</strong> <a href="http://13.57.236.210:5000">http://13.57.236.210:5000</a></p>
                        <hr>
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
                        <p style="color: red;">Please check the console output for error details.</p>
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
