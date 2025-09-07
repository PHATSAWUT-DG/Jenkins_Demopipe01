pipeline {
    // Run the pipeline steps directly on the Jenkins agent node (must have Docker installed)
    agent any

    stages {
        stage('Checkout') {
            steps {
                // This will run directly on the Jenkins agent node
                git branch: 'main', url: 'https://github.com/PHATSAWUT-DG/Jenkins_Demopipe01.git'
            }
        }

        stage('Setup venv & Install Dependencies') {
            steps {
                // --- VERIFICATION STEP (Keep this for debugging) ---
                sh '''
                echo "Verifying files in workspace after checkout:"
                ls -la "$WORKSPACE"
                if [ ! -f "$WORKSPACE/requirements.txt" ]; then
                  echo "ERROR: requirements.txt not found in workspace!"
                  exit 1
                else
                  echo "Found requirements.txt"
                fi
                '''
                // --- END OF VERIFICATION ---

                // --- RUN PYTHON SETUP INSIDE CONTAINER AS JENKINS USER ---
                sh '''
                # Get the UID and GID of the current user (jenkins on the agent)
                # Using 'id' command within the agent shell
                JENKINS_UID=$(id -u)
                JENKINS_GID=$(id -g)

                echo "Running Docker container as UID: $JENKINS_UID, GID: $JENKINS_GID"

                docker run --rm \
                  --user "$JENKINS_UID:$JENKINS_GID" \  # Run as the jenkins user
                  -v "$WORKSPACE:/workspace" \
                  -w /workspace \
                  python:3.11 \
                  bash -c "
                    set -e
                    echo 'Setting up virtual environment and installing dependencies...'
                    # Verify files are accessible inside the container
                    echo 'Contents of /workspace:'
                    ls -la
                    if [ ! -f requirements.txt ]; then
                      echo 'ERROR: requirements.txt not found inside the container!'
                      exit 1
                    fi
                    # Ensure the venv directory is created by the correct user
                    rm -rf venv # Clean up any potentially problematic previous venv
                    python3 -m venv venv
                    source venv/bin/activate
                    # pip install --upgrade pip # Optional: Skip upgrading pip if not strictly needed
                    pip install -r requirements.txt
                    echo 'Setup complete.'
                  "
                '''
                // --- END OF DOCKER RUN ---
            }
        }

        stage('Run Tests & Coverage') {
            steps {
                // Run tests inside a python:3.11 container, reusing the installed dependencies
                // Also run as the jenkins user for consistency
                sh '''
                JENKINS_UID=$(id -u)
                JENKINS_GID=$(id -g)
                echo "Running Docker container for tests as UID: $JENKINS_UID, GID: $JENKINS_GID"

                docker run --rm \
                  --user "$JENKINS_UID:$JENKINS_GID" \
                  -v "$WORKSPACE:/workspace" \
                  -w /workspace \
                  python:3.11 \
                  bash -c "
                    set -e
                    echo 'Running tests...'
                    source venv/bin/activate
                    export PYTHONPATH=.
                    pytest --maxfail=1 --disable-warnings -q --cov=app --cov-report=xml
                    echo 'Tests completed.'
                  "
                '''
            }
        }

        // ============ SONARSCANNER INSTALL STAGE ============
        // You might want to install SonarScanner on the Jenkins agent node itself
        // or download it fresh each time. Installing on the agent is more efficient.
        stage('Install SonarScanner') {
            steps {
                // This runs on the Jenkins agent node
                sh '''
                echo "Cleaning up previous SonarScanner files..."
                rm -f sonar-scanner.zip
                rm -rf sonar-scanner sonar-scanner-cli-7.2.0.5079-linux-x64

                echo "Downloading SonarScanner..."
                curl -sSL -o sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-7.2.0.5079-linux-x64.zip
                unzip -q -o sonar-scanner.zip
                mv sonar-scanner-7.2.0.5079-linux-x64 sonar-scanner
                echo "SonarScanner installed on Jenkins agent."
                '''
            }
        }
        // ===================================

        stage('SonarQube Analysis') {
            steps {
                // This runs on the Jenkins agent node, using the locally installed scanner
                // The scanner will analyze the code in the $WORKSPACE directory
                withSonarQubeEnv('SonarQube Scanner') { // Make sure this name matches your Jenkins config
                    withCredentials([string(credentialsId: 'sonarqube_token', variable: 'SCANNER_TOKEN')]) {
                        sh '''
                        # Debug: Confirm SCANNER_TOKEN is set from withCredentials
                        if [ -n "$SCANNER_TOKEN" ]; then
                            echo "SUCCESS: SCANNER_TOKEN (from withCredentials) is set."
                        else
                            echo "ERROR: SCANNER_TOKEN is NOT set. Check the credentialsId 'sonarqube_token'."
                            exit 1
                        fi

                        # Debug: Check if withSonarQubeEnv provided SONAR_HOST_URL
                        if [ -n "$SONAR_HOST_URL" ]; then
                            echo "INFO: SONAR_HOST_URL provided by withSonarQubeEnv: $SONAR_HOST_URL"
                            SONAR_SCANNER_OPTS="-Dsonar.host.url=$SONAR_HOST_URL"
                        else
                            echo "WARNING: SONAR_HOST_URL not provided by withSonarQubeEnv. Using default or sonar-project.properties."
                            SONAR_SCANNER_OPTS=""
                        fi

                        # Ensure sonar-scanner is in PATH (adjust path if needed)
                        export PATH=$PATH:$WORKSPACE/sonar-scanner/bin

                        # Run the scanner, explicitly passing the token
                        export SONAR_TOKEN="$SCANNER_TOKEN"
                        sonar-scanner $SONAR_SCANNER_OPTS
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                // This runs directly on the Jenkins agent node, where Docker CLI should be available
                sh 'docker build -t fastapi-app:latest .'
            }
        }

        stage('Deploy Container') {
            steps {
                 // This also runs directly on the Jenkins agent node
                sh '''
                docker stop fastapi_app || true
                docker rm fastapi_app || true
                docker run -d -p 8000:8000 --name fastapi_app fastapi-app:latest
                '''
            }
        }
    }
    post {
        always {
            echo "Pipeline finished"
             // Optional: Cleanup downloaded scanner files
            // sh 'rm -rf sonar-scanner sonar-scanner.zip || true'
        }
    }
}