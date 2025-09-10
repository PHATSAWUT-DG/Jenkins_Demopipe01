pipeline {
    agent {
        docker {
            image 'python:3.11'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        SONARQUBE = credentials('sonarqube_token1')
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
                # Install Java 17 (JDK, not just JRE)
                apt-get update && apt-get install -y openjdk-17-jdk

                # Verify Java installation
                java -version
                which java

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
        stage('Setup venv and Dependencies') {
            steps {
                sh '''
                # Download and install Java 17 without using apt
                wget -O openjdk-17.tar.gz https://download.java.net/java/GA/jdk17.8/2d25502d5506458985d96318953f08ee/7/GPL/openjdk-17.8_linux-x64_bin.tar.gz
                mkdir -p /tmp/java
                tar -xzf openjdk-17.tar.gz -C /tmp/java
                export JAVA_HOME=/tmp/java/jdk-17.8
                export PATH=$JAVA_HOME/bin:$PATH
                
                # Verify Java
                java -version
                
                # Set up Python virtual environment
                python3 -m venv venv
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt
                '''
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