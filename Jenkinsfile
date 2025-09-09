pipeline {
    agent any
    environment {
        SONARQUBE = credentials('sonarqube_token')
    }
    stages {
        stage('Checkout') {
            steps {
                // เปลี่ยนเป็น branch ที่คุณต้องการ
                git branch: 'feature', url: 'https://github.com/PHATSAWUT-DG/Jenkins_Demopipe01.git'
            }
        }
        stage('Run Tests & Coverage') {
            steps {
                sh '''
                python3 -m venv venv
                source venv/bin/activate
                pip install -r requirements.txt
                pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=xml
                '''
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('Sonarqube') {
                    sh 'sonar-scanner'
                }
            }
        }
        stage('Build Docker Image') {
            steps {
                // แก้ไข image name เป็น lowercase
                sh 'docker build -t fastapi-clean-demo:latest .'
            }
        }
        stage('Deploy Container') {
            steps {
                sh '''
                docker stop fastapi_app || true
                docker rm fastapi_app || true
                // แก้ไข port เป็น 8001
                docker run -d -p 8001:8000 --name fastapi_app fastapi-clean-demo:latest
                '''
            }
        }
    }
    post {
        always {
            echo "Pipeline finished"
        }
    }
}