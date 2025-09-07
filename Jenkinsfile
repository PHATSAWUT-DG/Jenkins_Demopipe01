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
            stage('Install SonarQube Scanner') {
                steps {
                    sh '''
                    apt-get update && apt-get install -y npm
                    npm install -g sonar-scanner
                    '''
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
        stage('SonarQube Analysis') {
          stage('Install SonarQube Scanner') {
            steps {
                sh '''
                apt-get update && apt-get install -y npm
                npm install -g sonar-scanner
                '''
            }
        }          
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
