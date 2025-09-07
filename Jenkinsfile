pipeline {
    agent {
        docker {
            image 'python:3.11'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        SONARQUBE = credentials('sonarqube_token')
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/PHATSAWUT-DG/Jenkins_Demopipe01.git'
            }
        }
        stage('Setup venv') {
            steps {
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                '''
            }
        }
        stage('Run Tests & Coverage') {
            steps {
                sh '''
                . venv/bin/activate
                export PYTHONPATH=.
                pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=xml
                '''
            }
        }
        // ============ NEW STAGE ============
        stage('Install SonarScanner') {
            steps {
                sh '''
                echo "Cleaning up previous files..."
                rm -f sonar-scanner.zip
                rm -rf sonar-scanner sonar-scanner-cli-7.2.0.5079-linux-x64        

                echo "Downloading SonarScanner..."
                curl -sSL -o sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-7.2.0.5079-linux-x64.zip?_gl=1*j7wxsl*_gcl_au*NDY1NzU2ODYzLjE3NTcyNzQ2MjA.*_ga*MjIyNzYzMTkuMTc1NzI3NDYxOQ..*_ga_9JZ0GZ5TC6*czE3NTcyNzQ2MTgkbzEkZzEkdDE3NTcyNzQ3MjkkajYwJGwwJGgw
                unzip -q -o sonar-scanner.zip
                ls -la
                mv sonar-scanner-7.2.0.5079-linux-x64 sonar-scanner
                export PATH=$PATH:/var/jenkins_home/workspace/FastAPI-Pipeline/sonar-scanner/bin
                echo "SonarScanner installed."
                '''
            }
        }
        // ===================================
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube Scanner') {
                    sh 'sonar-scanner'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t fastapi-app:latest .'
            }
        }
        stage('Deploy Container') {
            steps {
                sh 'docker run -d -p 8000:8000 --name fastapi_app fastapi-app:latest'
            }
        }
    }
    post {
        always {
            echo "Pipeline finished"
        }
    }
}