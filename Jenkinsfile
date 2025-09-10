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
        stage('Setup venv and Dependencies') {
            steps {
                sh '''
                # Install Java for SonarQube Scanner
                apt-get update && apt-get install -y openjdk-17-jre

                # Set up Python virtual environment
                python3 -m venv venv
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                '''
            }
        }
        stage('Run Tests & Coverage') {
            steps {
                sh 'venv/bin/pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=xml'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                    # Download, unzip, and configure SonarQube Scanner
                    wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.7.0.2747-linux.zip
                    unzip -o sonar-scanner-cli-4.7.0.2747-linux.zip
                    export PATH="$PATH:`pwd`/sonar-scanner-4.7.0.2747-linux/bin"

                    # Run SonarQube Scanner with the new Java version
                    sonar-scanner
                    '''
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
                sh 'docker run -d -p 8000:8000 fastapi-app:latest'
            }
        }
        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker tag fastapi-app:latest $DOCKER_USER/fastapi-app:latest
                    docker push $DOCKER_USER/fastapi-app:latest
                    '''
                }
            }
        }
    }
    post {
        always {
            echo "Pipeline finished"
        }
    }
}