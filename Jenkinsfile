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

        stage('Setup Environment (Java & Python)') {
            steps {
                sh '''
                echo "===== Checking for curl (alternative to wget) ====="
                if ! command -v curl &> /dev/null; then
                    echo "ERROR: Neither wget nor curl is available in the container. Cannot download Java."
                    exit 1
                else
                    echo "curl is available."
                fi

                echo "===== Downloading and Installing Java 17 (JDK) Manually using curl ====="
                # Use a verified download link from Eclipse Temurin (Adoptium)
                # Example for Temurin 17.8+7 JDK for Linux x64:
                JAVA_TAR_GZ="OpenJDK17U-jdk_x64_linux_hotspot_17.8_7.tar.gz"
                JAVA_DOWNLOAD_URL="https://github.com/adoptium/temurin17-binaries/releases/download/jdk-17.8%2B7/OpenJDK17U-jdk_x64_linux_hotspot_17.8_7.tar.gz"

                # Check if file already exists to avoid re-download
                if [ ! -f "$JAVA_TAR_GZ" ]; then
                    echo "Downloading Java 17 from $JAVA_DOWNLOAD_URL..."
                    # Use curl to download
                    curl -L -o "$JAVA_TAR_GZ" "$JAVA_DOWNLOAD_URL"
                    if [ $? -ne 0 ]; then
                        echo "ERROR: Failed to download Java 17 using curl. Aborting."
                        exit 1
                    fi
                else
                    echo "Java archive already exists, skipping download."
                fi

                # Create directory and extract
                mkdir -p /tmp/java
                tar -xzf "$JAVA_TAR_GZ" -C /tmp/java
                # Find the extracted JDK directory name dynamically
                JDK_DIR_NAME=$(tar -tzf "$JAVA_TAR_GZ" | head -1 | cut -f1 -d"/")
                if [ -z "$JDK_DIR_NAME" ]; then
                    echo "ERROR: Could not determine JDK directory name after extraction."
                    exit 1
                fi
                export JAVA_HOME=/tmp/java/$JDK_DIR_NAME
                export PATH=$JAVA_HOME/bin:$PATH

                echo "JAVA_HOME set to $JAVA_HOME"
                echo "PATH updated to include $JAVA_HOME/bin"

                # Verify Java installation
                echo "Verifying Java installation:"
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