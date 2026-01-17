pipeline {
    agent any
    
    environment {
        PYTHON = 'python3'
        PIP = 'pip3'
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
                    ${PIP} install flask pytest
                '''
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running unit tests...'
                sh '''
                    . venv/bin/activate
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
                    pkill -f "flask run" || true
                    export FLASK_APP=app.py
                    export FLASK_ENV=development
                    nohup ${PYTHON} -m flask run --host=0.0.0.0 --port=5000 > flask.log 2>&1 &
                    echo $! > flask.pid
                    sleep 5
                    echo "Application deployed to staging at http://localhost:5000"
                '''
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            cleanWs()
        }
    }
}
