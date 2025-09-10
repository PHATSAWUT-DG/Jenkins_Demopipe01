pipeline {
    agent {
        docker {
            image 'python:3.11'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    environment {
        // Ensure this matches the credential ID you create in Jenkins
        SONARQUBE = credentials('sonarqube_token1')
        // Explicitly set JAVA_HOME for SonarQube Scanner CLI if needed
        // JAVA_HOME = '/tmp/java/jdk-17.8' // Uncomment if using the wget method and needing to hint SonarQube
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/PHATSAWUT-DG/Jenkins_Demopipe01.git'
            }
        }

        // Combine Setup stages into one, removing the duplicate
        stage('Setup Environment (Java & Python)') {
            steps {
                sh '''
                echo "===== Installing Java 17 (JDK) ====="
                # Option 1: Using apt (simpler, but requires updating package list)
                apt-get update && apt-get install -y openjdk-17-jdk

                # Option 2: Manual download (Alternative, if apt is problematic inside the python container)
                # Note: You might need wget/curl first: apt-get update && apt-get install -y wget
                # wget -O openjdk-17.tar.gz https://download.java.net/java/GA/jdk17.8/2d25502d5506458985d96318953f08ee/7/GPL/openjdk-17.8_linux-x64_bin.tar.gz
                # mkdir -p /tmp/java
                # tar -xzf openjdk-17.tar.gz -C /tmp/java
                # export JAVA_HOME=/tmp/java/jdk-17.8
                # export PATH=$JAVA_HOME/bin:$PATH
                # echo "JAVA_HOME set to $JAVA_HOME"

                # Verify Java installation
                java -version
                which java

                echo "===== Setting up Python Virtual Environment ====="
                python3 -m venv venv
                # Activate venv and install dependencies in one sh block
                . venv/bin/activate
                pip install --upgrade pip
                pip install -r requirements.txt

                # Verify installations if needed
                # . venv/bin/activate && python -c "import sys; print(sys.version)"
                # . venv/bin/activate && pip list
                '''
            }
        }

        stage('Run Tests & Coverage') {
            steps {
                // Ensure the virtual environment is activated for this step
                sh '''
                . venv/bin/activate
                pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=xml
                '''
            }
        }

        // SonarQube Analysis Stage (from original lab sheet)
        // Make sure SonarQube is running and credentials are configured in Jenkins
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('Sonarqube') { // Ensure 'Sonarqube' matches the name in Jenkins config
                    sh '''
                    # Ensure Java is available for Sonar Scanner
                    java -version
                    # Run Sonar Scanner
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
                 // Optional: Stop and remove previous container if it exists
                sh '''
                docker stop fastapi-app-container || true
                docker rm fastapi-app-container || true
                docker run -d --name fastapi-app-container -p 8000:8000 fastapi-app:latest
                '''
            }
        }

        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred', // Ensure this matches the credential ID in Jenkins
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    # Tag the image correctly for Docker Hub
                    docker tag fastapi-app:latest $DOCKER_USER/fastapi-app:latest
                    docker push $DOCKER_USER/fastapi-app:latest
                    # Optional: Push with a build number or git commit tag
                    # GIT_COMMIT_SHORT=$(git rev-parse --short HEAD)
                    # docker tag fastapi-app:latest $DOCKER_USER/fastapi-app:$GIT_COMMIT_SHORT
                    # docker push $DOCKER_USER/fastapi-app:$GIT_COMMIT_SHORT
                    '''
                }
            }
        }
    }
    post {
        always {
            echo "Pipeline execution finished."
            // Optional: Clean up the local Docker image after pushing?
            // sh 'docker rmi fastapi-app:latest $DOCKER_USER/fastapi-app:latest || true'
        }
    }
}