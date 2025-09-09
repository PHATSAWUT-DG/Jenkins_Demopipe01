pipeline {
    agent any // This uses the main Jenkins agent for all general tasks
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
                sh 'echo "Workspace cleaned"'
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/PHATSAWUT-DG/Jenkins_Demopipe01.git'
                sh 'echo "=== FILES AFTER CHECKOUT ==="; ls -la'
            }
        }

        stage('Run Tests & Coverage') {
            agent {
                docker {
                    image 'python:3.11'
                }
            }
            steps {
                sh '''
                echo "Running tests inside Docker container..."
                python -m venv venv
                source venv/bin/activate
                pip install --no-cache-dir -r requirements.txt
                echo "Dependencies installed. Running tests..."
                pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=xml
                echo "Tests completed successfully."
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonarqube_token', variable: 'SCANNER_TOKEN')]) {
                        sh '''
                        echo "SUCCESS: SCANNER_TOKEN is set."
                        export SONAR_TOKEN="${SCANNER_TOKEN}"
                        /var/jenkins_home/workspace/FastAPI-Clean-Demo@2/sonar-scanner/bin/sonar-scanner -Dsonar.host.url="${SONAR_HOST_URL}"
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            agent {
                docker {
                    image 'docker:19.03.13'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh 'docker build -t fastapi-app:latest .'
            }
        }

        stage('Deploy Container') {
            agent {
                docker {
                    image 'docker:19.03.13'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh '''
                docker stop fastapi_app || true
                docker rm fastapi_app || true
                docker run -d -p 8000:8000 --name fastapi_app fastapi-app:latest
                '''
            }
        }
    }
}